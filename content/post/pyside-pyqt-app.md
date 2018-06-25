---
title: "Cross Platform Desktop Apps with PySide/PyQt"
date: 2018-06-03T17:13:04-07:00
---


## Introduction

While web applications are increasingly the go-to approach for modern applications, native desktop applications undoubtedly still have their place.

Qt is a very robust, well documented, cross-platform GUI framework. Qt is a C++ framework, but conveniently there are bindings for many other languages including Python. Additionally, we can use Qt Designer for WYSIWYG design of our GUI layout.

We can also use other tools such as cx_Freeze to create an executable application and InnoSetup to create a Windows installer.

Full source for example PySide and PyQt5 applications are found here:

https://bitbucket.org/Odoth/pyside-template/

https://bitbucket.org/Odoth/pyqt5-template/


## Qt vs Other GUI Toolkits

Some alternatives to Qt for building cross-platform desktop GUI aps with Python are:

* Tkinter (Python standard library)
* wxPython
* Kivy

Generally, these are all good toolkits and can be used to build great apps.


## PySide vs PyQt4 vs PySide2 vs PyQt5 vs "Qt for Python"

PySide and PyQt4 are two different Python binding libraries for Qt4. Similarly, PySide2 and PyQt5 are both bindings for Qt5. "Qt for Python" is the name of the project that produces the PySide2 library.

So which should you choose?

For Qt4 (PySide vs PyQt4), the API for both bindings are nearly interchangeable, so the bigger consideration would be the license difference. PyQt4 can be used under either GPL or commercial license, while PySide can be used under the more permissive LGPL license. PyPi has win32 PySide wheels supporting Python v2.6 - v3.4, making it easy to install on Windows, but building is required on other platforms. PyQt4 latest release is available as source only.

As of this writing, PySide2 is still under development so the only viable option for Qt5 is PyQt5. PyQt5 also uses the dual GPL/commercial license like PyQt4.

Python version also matters. PyPi has PyQt5 wheels only for Python v3.5+. Supposedly you can build PyQt5 for lower version if needed (I haven't tried it).

__TL;DR: For Python v2.6 - v3.4 use PySide. For Python v3.5+ use PyQt5.__

Below examples are for Python 3.6 with PyQt5.

## Installation
```
pip install pyqt5
Collecting pyqt5
  Downloading https://files.pythonhosted.org/packages/e4/15/4e2e49f64884edbab6f833c6fd3add24d7938f2429aec1f2883e645d4d8f/PyQt5-5.10.1-5.10.1-cp35.cp36.cp37.cp38-abi3-manylinux1_x86_64.whl (107.8MB)
    100% |████████████████████████████████| 107.8MB 363kB/s 
Collecting sip<4.20,>=4.19.4 (from pyqt5)
  Downloading https://files.pythonhosted.org/packages/8a/ea/d317ce5696dda4df7c156cd60447cda22833b38106c98250eae1451f03ec/sip-4.19.8-cp36-cp36m-manylinux1_x86_64.whl (66kB)
    100% |████████████████████████████████| 71kB 13.4MB/s 
Installing collected packages: sip, pyqt5
Successfully installed pyqt5-5.10.1 sip-4.19.8

```

## Project Structure
The basic folder structure looks like this:

```
project
|
|--/designer
|
|--/icons
|
|--/src
   |
   |--/ui
   |
   |--/widgets
   |
   |--app_info.py
   |
   |--application.py
```

__`designer`__ - This is where all the `.ui` files created with Designer will be saved.

__`src/ui`__ - The `.ui` files from `designer` will be processed and `.py` files will be generated in this folder.

__`src/widgets`__ - Here we'll write the actual logic for each of the widgets we created with Designer.

__`src/app_info.py`__ - Simple file containing version and other details.

__`src/application.py`__ - The application entry point.


## WYSIWYG with Designer
You'll need to make sure Qt5 Designer is installed on your system. Installation steps will vary depending on your platform.

I'll leave general Designer tutorial to others, but one thing I will cover is how to include other custom widgets when using the Designer.

In our PyQt5 example application we have created `SomeWidget.ui` and `AnotherWidget.ui` with Designer and saved them in the `designer` directory. Now we want to add these widgets to our `MainWindow.ui`. We simply add a place holder `Widget` to our MainWindow then Right Click that Widget and select `Promote to...`.

In the __Promoted class name__ field we put the class name that we will use in our Python code, e.g. `SomeWidget`. In the __Header file__ field we put the path to our Python file containing this class, except using __.h__ extension instead of __.py__, e.g. `widgets/some_widget.h`.


## Compiling UI Files
The tool used to compile .ui files to Python files is `uic`. It's usable as a command line tool or can be imported and used as a Python library. We use this snippet in our `build.py` script to compile all our .ui files and store the output files in the `src/ui` directory.

```python
def build_ui():
    import os
    from PyQt5.uic import compileUiDir

    design_dir = os.path.join(os.path.dirname(__file__), 'designer')

    def uicmap(py_dir, py_file):
        rtn_dir = os.path.join("src", "ui")
        rtn_file = py_file.replace(".py", "_ui.py")

        return rtn_dir, rtn_file

    compileUiDir(design_dir, map=uicmap)
```

## Implementing Widgets
Compiling our `.ui` files just generated a bunch of ui setup code for us. We have to define the actual Widget classes in our Python code and call into the generated code using the `setupUi` method.

Any additional logic then can be added to our class, such as adding signal handlers.

```python
# src/widgets/main_window.py

from PyQt5 import QtWidgets, QtGui
from ui.MainWindow_ui import Ui_MainWindow
from app_info import APP_INFO


class MainWindow(QtWidgets.QMainWindow, Ui_MainWindow):
    def __init__(self, *args, **kwargs):
        super(MainWindow, self).__init__(*args, **kwargs)

        self.setupUi(self)

        self.setWindowTitle(APP_INFO.APP_NAME)
        self.setWindowIcon(QtGui.QIcon(":/icons/icon.ico"))

        self.actionAbout.triggered.connect(self.actionAbout_triggered)
        self.actionAbout_Qt.triggered.connect(self.actionAbout_Qt_triggered)

    def actionAbout_triggered(self):
        QtWidgets.QMessageBox.about(
            self,
            APP_INFO.APP_NAME,
            """
            <h1>{APP_NAME}</h1>
            <br>
            Author:  {APP_AUTHOR}<br>
            Version: {Major}.{Minor}.{Revision}<br>
            """.format(
                APP_NAME=APP_INFO.APP_NAME,
                APP_AUTHOR=APP_INFO.APP_AUTHOR,
                Major=APP_INFO.APP_VERSION.Major,
                Minor=APP_INFO.APP_VERSION.Minor,
                Revision=APP_INFO.APP_VERSION.Revision
            )
        )

    def actionAbout_Qt_triggered(self):
        QtWidgets.QMessageBox.aboutQt(self)

```

## Icons and Resources
Icons and other resources are specified in `.qrc` files.

```
<!DOCTYPE RCC><RCC version="1.0">
<qresource>
    <file>icons/icon.ico</file>
</qresource>
</RCC>
```

The `.qrc` file must then be compiled to a .py file which will be imported in the application.

The tool used to compile .qrc files to Python files is `pyrcc`. It's usable as a command line tool or can be imported and used as a Python library. We use this snippet in our `build.py` script to compile our resources into `resources.py`.

```python
def build_resources():
    import os
    from PyQt5.pyrcc_main import main as pyrcc_main_func

    QRC_FILE = 'resources.qrc'
    PY_OUTPUT = os.path.join('src', 'resources.py')

    sys.argv = ['', '-o', PY_OUTPUT, QRC_FILE]
    pyrcc_main_func()
```


## Application Entry Point
Our application entry point is `src/application.py`. 

```python
# src/application.py

if __name__ == '__main__':
    import sys
    from PyQt5 import QtWidgets
    import resources
    from widgets.main_window import MainWindow

    app = QtWidgets.QApplication(sys.argv)
    main_win = MainWindow()
    main_win.show()

    sys.exit(app.exec_())
```

And if all goes well, we should see our awesome app on screen!

![SomeApp](/img/some-app.png)


## Creating an Executable with cx_Freeze
cx_Freeze is a tool that allows you to create executables from Python scripts. It's usable as a command line tool or can be imported and used as a Python library. We use this snippet in our `build.py` to invoke it.

```python
def build_exe():
    import os
    import sys
    import cx_Freeze

    from app_info import APP_INFO

    # have to make sure args looks right
    sys.argv = sys.argv[:1] + ['build']

    app_path = os.path.join(os.path.dirname(__file__), "src", "application.py")

    if sys.platform == 'win32':
        executables = [cx_Freeze.Executable(
            app_path,
            targetName=APP_INFO.APP_NAME + ".exe",
            icon=os.path.join('icons', 'icon.ico'),
            base="Win32GUI")]
    else:
        executables = [cx_Freeze.Executable(
            app_path,
            targetName=APP_INFO.APP_NAME,
            icon=os.path.join('icons', 'icon.png'))]

    include_files = [
        os.path.join("icons", 'icon.ico')
    ]

    options = {
        'build_exe': {
            "include_files": include_files
        }
    }

    cx_Freeze.setup(
        name=APP_INFO.APP_NAME,
        version=_get_ver_string(),
        executables=executables,
        options=options
    )
```

## Creating a Windows Installer with Inno Setup
Inno Setup is a tool used for creating Windows installers. We need to use the `iscc` command line tool to invoke it. We provide several arguments to define variable values and finally specify our `.iss` setup script file.

```python
def build_win_install():
    from app_info import APP_INFO
    import os
    os.system('iscc' +
              ' /DMyAppVersion="{}"'.format(_get_ver_string()) +
              ' /DMyAppName="{}"'.format(APP_INFO.APP_NAME) +
              ' /DMyAppPublisher="{}"'.format(APP_INFO.APP_PUBLISHER) +
              ' /DMyAppURL="{}"'.format(APP_INFO.APP_URL) +
              ' /DMyAppExeName="{}"'.format(APP_INFO.APP_NAME + ".exe") +
              ' inno_setup.iss'
    )
```


## References
https://wiki.qt.io/Qt_for_Python

https://bitbucket.org/Odoth/pyside-template/

https://bitbucket.org/Odoth/pyqt5-template/

http://doc.qt.io/Qt-5/designer-using-custom-widgets.html