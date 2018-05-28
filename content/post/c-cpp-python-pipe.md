---
title: "Pipes for basic IPC between C/C++ and Python"
date: 2018-05-27T23:02:46Z
---


## Introduction

Recently I encountered the following circumstances:

* C++ program (Linux) needs to call a Python script.
* The Python script needs access to some APIs from the C++ program.
* Using the Python/C API is not an option.

Python/C API for embedding Python in the application would probably be the cleanest approach, but as mentioned above, it was not an option in this case.

One approach considered was to use pipes. Here I describe the approach and provide a proof of concept. The full proof of concept source code can be found at https://bitbucket.org/Odoth/cpp_python_pipe/. The proof of concept was made using C++11 and Python 2.7.

## Pipes
A pipe is a unidirectional data channel that can be used for interprocess communication. Basic pipes are created using the `pipe()` system call and can be used to facilitate communication between a parent process and a child process.

In this case, our parent process is the C++ program and the child process is the Python interpreter running the script.

`pipe()` creates the pipe and gives us two file descriptors numbers, one for either end of the pipe (read end and write end). 

So we can simply create a pipe, `fork()` our program, call our python script from the forked process, and the script and C++ program can use these file descriptors to communicate.

The important detail here is that the child process inherits a copy of the parent's file descriptors at the time of forking. This allows both processes to access the same underlying pipe using the same file descriptor numbers. Other processes that are not descendants of the process which called `pipe()` can't access the pipe. So, for example, if `pipe()` gives us 5 and 6 as the pipe file descriptor numbers, both the parent and child can access the underlying pipe with those file descriptor numbers. If some separate, unrelated process, tried to do some operation using file descriptor numbers 5 and 6, it would not resolve to the pipe we created. These numbers may be invalid file descriptor numbers in that process or refer to some other file.

## Proof of concept
Our `main()` function looks pretty similar to any simple pipe/fork example you'll find elsewhere. 

Since we need bi-directional communication, we open 2 pipes before forking. We pass the needed file descriptor numbers to the Python script using environment variables, but this could also be done with command line arguments or other ways if needed.

We are using `system()` to call the Python script, but ``execve()`` or ``popen()`` may be more appropriate in a real implementation.

We are careful to call `close()` on all file descriptor numbers from both child and parent process. The underlying pipe will not be destroyed until no processes have the file descriptors open. In this example it's not really necessary as everything will be cleaned up when the processes end anyway. In a real implementation, the C++ program is presumably long running, so leaking pipes should be a concern. Note that, after the fork, in each process we immediately close the ends of each pipe that won't be used in that process. This is considered good practice.

```c++
int main() 
{
    int pipe_cpp_to_py[2];
    int pipe_py_to_cpp[2];

    if (::pipe(pipe_cpp_to_py) || ::pipe(pipe_py_to_cpp))
    {
        std::cout << "Couldn't open pipes" << std::endl;
        ::exit(1);
    }

    pid_t pid = fork();

    if ( pid == 0 )
    {
        ::close(pipe_py_to_cpp[0]);
        ::close(pipe_cpp_to_py[1]);
        std::ostringstream oss;

        oss << "export PY_READ_FD=" << pipe_cpp_to_py[0] << " && "
            << "export PY_WRITE_FD=" << pipe_py_to_cpp[1] << " && "
            << "export PYTHONUNBUFFERED=true && " // Force stdin, stdout and stderr to be totally unbuffered.
            << "python src/main.py";


        ::system(oss.str().c_str());
        ::close(pipe_py_to_cpp[1]);
        ::close(pipe_cpp_to_py[0]);

    }
    else if ( pid < 0 )
    {
        std::cout << "Fork failed." << std::endl;
        ::exit(1);
    }
    else
    {
        ::close(pipe_py_to_cpp[1]);
        ::close(pipe_cpp_to_py[0]);
        read_loop(pipe_py_to_cpp[0], pipe_cpp_to_py[1]);
        ::close(pipe_py_to_cpp[0]);
        ::close(pipe_cpp_to_py[1]);
    }

    return 0;
}
```

We have some APIs we want to expose to the script. In this example the APIs are very simple, taking a single string input argument, and outputing a string (through a std::ostream).

```c++
/* 
APIs to be accessed by the script.
*/

static void foo_api(std::ostream &os, const std::string &arg)
{
    os << "Foo was called with arg " << arg;
}

static void bar_api(std::ostream &os, const std::string &arg)
{
    os <<"Bar was called with arg " << arg;
}

/* end API section */
```
So we need some simple protocol whereby the Python script can request some API call on one pipe and the C++ program can give the response on the other pipe.

We use the simple message format of \[apiNameSize\]\[apiName\]\[apiArgSize\]\[apiArg\] for the API request, and \[resultSize\]\[resultString\] for the response.

Note that even though the Python script is running in a separate process, our ``read_loop`` won't exit until the script finishes, so it's akin to making a synchronous call to the script with respect to the overall flow of our C++ program.

```c++
/* return true if val is set, false for EOF */
static bool read_uint32(int read_fd, uint32_t &val)
{
    unsigned char msgSizeBuf[4];
    unsigned iBuf = 0;

    while (iBuf < sizeof(msgSizeBuf))
    {
        ssize_t rc = ::read(read_fd, msgSizeBuf + iBuf, sizeof(msgSizeBuf) - iBuf);

        if (rc == 0)
        {
            return false;
        }
        else if (rc < 0 )
        {
            std::cout << __func__ << "@" << __LINE__ << ":::Read ERROR" << std::endl;
            exit(1);
        }
        else
        {
            iBuf += rc;
        }
    }

    val = *(static_cast<uint32_t *>(static_cast<void *>(&msgSizeBuf[0])));
    
    return true;
}


static void send_msg(int write_fd, std::string msg)
{
    uint32_t msgSize = msg.size();
    unsigned char msgSizeBuf[4];

    ::memcpy(msgSizeBuf, &msgSize, sizeof(msgSize));

    unsigned iBuf = 0;
    while (iBuf < 4)
    {
        ssize_t rc = ::write(write_fd, msgSizeBuf + iBuf, sizeof(msgSizeBuf) - iBuf);
        if ( rc < 0 )
        {
            std::cout << "Error writing message size" << std::endl;
            ::exit(1);
        }
        else if ( rc == 0 )
        {
            std::cout << "rc == 0, what does that mean?" << std::endl;
            ::exit(1);
        }
        else
        {
            iBuf += rc;
        }
    }

    iBuf = 0;
    const char *msgBuf = msg.c_str();
    while (iBuf < msgSize)
    {
        ssize_t rc = ::write(write_fd, msgBuf + iBuf, msgSize - iBuf);
        if ( rc < 0 )
        {
            std::cout << "Error writing message" << std::endl;
            ::exit(1);
        }
        else if ( rc == 0 )
        {
            std::cout << "rc == 0, what does that mean?" << std::endl;
            ::exit(1);
        }
        else
        {
            iBuf += rc;
        }
    }
}

static std::string read_string(int read_fd, uint32_t sz)
{
    std::vector<char> msgBuf( sz + 1 );
    msgBuf[ sz ] = '\0';
    unsigned iBuf = 0;

    while (iBuf < sz)
    {
        ssize_t rc = ::read(read_fd, &(msgBuf[0]) + iBuf, sz - iBuf);

        if ( rc == 0 )
        {
            std::cout << __func__ << "@" << __LINE__ << ":::EOF read" << std::endl;
            exit(1);
        }
        else if ( rc < 0 )
        {
            std::cout << __func__ << "@" << __LINE__ << ":::Read ERROR during message" << std::endl;
            exit(1);
        }
        else
        {
            iBuf += rc;
        }
    }

    return std::string( &(msgBuf[0]) );
}


static void read_loop(int read_fd, int write_fd)
{
    while ( 1 )
    {
         /* Python sends format
            [apiNameSize][apiName][apiArgSize][apiArg]
            on the pipe */
        std::cout << "Waiting for message from python..." << std::endl;
        uint32_t apiNameSize;
        if (!read_uint32(read_fd, apiNameSize))
        {
            // EOF waiting for a message, script ended
            std::cout << "EOF waiting for message, script ended" << std::endl;
            return;
        }
        std::string apiName = read_string(read_fd, apiNameSize);
        uint32_t apiArgSize;
        if (!read_uint32(read_fd, apiArgSize))
        {
            std::cout << "EOF white reading apiArgSize" << std::endl;
            ::exit(1);
        }
        std::string apiArg = read_string(read_fd, apiArgSize);

        std::cout << "apiName: " << apiName << std::endl
                  << "apiArg:  " << apiArg << std::endl;


        // Response comes as [resultSize][resultString]
        if (apiName == "foo")
        {
            std::ostringstream os;
            foo_api(os, apiArg);
            send_msg(write_fd, os.str());
        }
        else if (apiName == "bar")
        {
            std::ostringstream os;
            bar_api(os, apiArg);
            send_msg(write_fd, os.str());
        }
        else
        {
            std::cout << "UNSUPPORTED API " << apiName << std::endl;
            send_msg(write_fd, "__BAD API__"); 
        }
    }
}
```

Python lets us use `os.fdopen()` to create a Python file-like object from our file descriptor numbers.

We are using the `struct` library to help us translate between byte sequence and Python numbers. We are hard coding little endian here with the `"<"` in `"<I"`.

```python
import os
import time
import struct


_r_fd = int(os.getenv("PY_READ_FD"))
_w_fd = int(os.getenv("PY_WRITE_FD"))


_r_pipe = os.fdopen(_r_fd, 'rb', 0)
_w_pipe = os.fdopen(_w_fd, 'wb', 0)


def _read_n(f, n):
    buf = ''
    while n > 0:
        data = f.read(n)
        if data == '':
            raise RuntimeError('unexpected EOF')
        buf += data
        n -= len(data)
    return buf


def _api_get(apiName, apiArg):
    # Python sends format
    # [apiNameSize][apiName][apiArgSize][apiArg]
    # on the pipe
    msg_size = struct.pack('<I', len(apiName))
    _w_pipe.write(msg_size)
    _w_pipe.write(apiName)

    apiArg = str(apiArg)  # Just in case
    msg_size = struct.pack('<I', len(apiArg))
    _w_pipe.write(msg_size)
    _w_pipe.write(apiArg)

    # Response comes as [resultSize][resultString]
    buf = _read_n(_r_pipe, 4)
    msg_size = struct.unpack('<I', buf)[0]

    data = _read_n(_r_pipe, msg_size)

    if data == "__BAD API__":
        raise Exception(data)

    return data


# APIs to C++
def foo(arg):
    return _api_get("foo", arg)


def bar(arg):
    return _api_get("bar", arg)


# this one doesn't actually exist
def boo(arg):
    return _api_get("boo", arg)


def main():
    print 'Script Starting'
    for i in xrange(10):
        res = foo(100 - i)
        print i, " foo: ", res
        res = bar("something " + str(i))
        print i, " bar: ", res
        time.sleep(1)

        # if i == 5:
        #     res = boo(i)


if __name__ == "__main__":
    main()

```

And our output shows that it is working!


```normal
Waiting for message from python...
Script Starting
apiName: foo
apiArg:  100
Waiting for message from python...
0  foo:  Foo was called with arg 100
apiName: bar
apiArg:  something 0
Waiting for message from python...
0  bar:  Bar was called with arg something 0
apiName: foo
apiArg:  99
Waiting for message from python...
1  foo:  Foo was called with arg 99
apiName: bar
apiArg:  something 1
Waiting for message from python...
1  bar:  Bar was called with arg something 1
apiName: foo
apiArg:  98
Waiting for message from python...
2  foo:  Foo was called with arg 98
apiName: bar
apiArg:  something 2
Waiting for message from python...
2  bar:  Bar was called with arg something 2
apiName: foo
apiArg:  97
Waiting for message from python...
3  foo:  Foo was called with arg 97
apiName: bar
apiArg:  something 3
Waiting for message from python...
3  bar:  Bar was called with arg something 3
apiName: foo
apiArg:  96
Waiting for message from python...
4  foo:  Foo was called with arg 96
apiName: bar
apiArg:  something 4
Waiting for message from python...
4  bar:  Bar was called with arg something 4
apiName: foo
apiArg:  95
Waiting for message from python...
5  foo:  Foo was called with arg 95
apiName: bar
apiArg:  something 5
Waiting for message from python...
5  bar:  Bar was called with arg something 5
apiName: foo
apiArg:  94
Waiting for message from python...
6  foo:  Foo was called with arg 94
apiName: bar
apiArg:  something 6
Waiting for message from python...
6  bar:  Bar was called with arg something 6
apiName: foo
apiArg:  93
Waiting for message from python...
7  foo:  Foo was called with arg 93
apiName: bar
apiArg:  something 7
Waiting for message from python...
7  bar:  Bar was called with arg something 7
apiName: foo
apiArg:  92
Waiting for message from python...
8  foo:  Foo was called with arg 92
apiName: bar
apiArg:  something 8
Waiting for message from python...
8  bar:  Bar was called with arg something 8
apiName: foo
apiArg:  91
Waiting for message from python...
9  foo:  Foo was called with arg 91
apiName: bar
apiArg:  something 9
Waiting for message from python...
9  bar:  Bar was called with arg something 9
EOF waiting for message, script ended
```

## Final Thoughts
Using this approach is viable for very simple interface between the C++ and Python programs. However, as the complexity of the API increases, so will the complexity of the IPC message protocol. It has to be defined and implemented on both sides, including robust error handling. This could be mitigated somewhat by using existing serialization formats such as JSON or Protocol Buffers, but those, of course, bring their own complexity. Using the Python/C API would be preferred whenever possible.

## References
http://man7.org/linux/man-pages/man2/pipe.2.html

https://linux.die.net/man/7/pipe

https://docs.python.org/2/library/struct.html

https://bitbucket.org/Odoth/cpp_python_pipe/