--------------------

**Note from Philippe Fremy**

*I started porting winpdb-reborn to Python 3 / Phoenix but the amount of effort exceeds my availability. So: WINPDB ON PYTHON 3 IS BUGGY AND NOT WORKING.*

*If you like winpdb and want it on python 3 please contribute a bit of your time to fix one or two bugs. With the help of everybody, we can make it work ! It is on the master branch of this repo*

Winpdb for Python 2 is maintained under the branch *winpdb-reborn-python2*

--------------------

|Build Status Linux| |Build status Windows|

*Winpdb Reborn - A GPL Python Debugger, reborn from the unmaintained
Winpdb*

by Philippe Fremy as maintainer, and Nir Aides as initial author

Website: https://github.com/bluebird75/winpdb 

**This version of Winpdb is for Python 3 only. It is under development. Use it at your own risk !**

A stable version of Winpdb for Python 2 is available on PyPi: https://pypi.org/project/winpdb-reborn/1.5.0/ and in the branch *winpdb-reborn-python2*

Description
===========

*Winpdb Reborn* is a portable (Windows / Linux / macOS) standalone graphical debugger for Python. It focuses on making debugging
easy and does not include any IDE features. It works on both Python 2 and 3 and has very little dependencies (only wxPython for the GUI).

**Features**:

- graphical interface with stack/variable inspection, breakpoints and more
- breakpoints: on lines or functions, with possible conditions, load/save breakpoint info
- multi-threaded debugging
- smart handling of fork, follow parent or child
- remote debugging: GUI on one computer, program to debug on another computer with encrypted communication
- up to 20 times quicker than pdb, the default Python debugger (on which many other debuggers are built)
- debug PyPy scripts

**Note:** The original Winpdb was no longer maintained since the v1.4.8 release in
2010. I (Philippe Fremy) am providing a maintained version with new
functionality under the *Winpdb Reborn* name.

Requirements
============

Platform supported:

-  Linux
-  Windows XP and above
-  macOS is probably working but not tested

To run Winpdb Reborn:

- CPython 2.6 or 2.7 and wxPython 3
- or CPython 3.4 or above and wxPython 4

Note that running Winpdb with PyPy is possible, by launching rpdb2.py with ``pypy`` and using the Winpdb GUI to connect to it.

Winpdb Reborn is NOT compatible with Jython or IronPython.

Release history
===============

Version 2.0.0 - in development
------------------------------

(version in progress)

- port Winpdb to wxPython 4 / Python 3


Version 1.5.0
-------------

-  First official release by Philippe Fremy
-  Add support for Python 3 GUI (using wxPython Phoenix)
-  Allow installation of all dependencies with pip
-  Add support for Python 2.7, 3.5 and 3.6
-  Drop support for Python 2.5 and wxPython 2.6, min versions are now Python 2.6 and wxPython 3.0
-  rpdb2.py is now compatible with Python 3
-  Add support for PyPy to Rpdb2
-  Can now specify a different Python interpreter for the program under
   debug, allowing PyPy support
-  Avoid crash on Windows when closing debugger
-  Support drag’n drop of files to load source code
-  Add unit-tests and a functional test suite
-  Add a Continuous Integration server with travis CI
-  Can adjust behavior of debugger to activate/deactivate the breakpoint before exit
-  fix for launching Gnome Terminal properly


Version 1.4.8
-------------
Last stable version released by Nir Aides.

Installation
============

(not working yet until the new version is released).

The standard way to install Winpdb Reborn is with pip, as administrator/root::

    # python -m pip install winpdb-reborn

You must also install the wxPython for your version of Python: 

-  for Python 3, wxPython is automatically installed as a dependency with pip
-  for Python 2, check https://sourceforge.net/projects/wxpython/files/wxPython/

*Winpdb Reborn* is not packaged yet by any Linux distro. If your package manager proposes
to install Winpdb, that’s the old unmaintained Winpdb which works neither with Python 2.7 nor with Python 3.

Additional installation methods
-------------------------------

To install from a checkout or from an archive::

    # python setup.py install -f

No install mode
---------------

If you don’t want to install Winpdb Reborn, you can still try it by calling it explicitly with
your program to debug::

    $ python /the/path/to/winpdb.py my_program.py 

Where do the files go ?
-----------------------

The setup script copies rpdb2.py and winpdb.py modules to the Python
site-packages folder. The scripts rpdb2 and winpdb are copied to the
Python binaries (scripts) folder. On Linux, this folder is usually ``/usr/bin`` 
and is in the path by default. On Windows, this folder is ``%PYTHONHOME%\Scripts``,
where you should see a winpdb.exe and rpdb2.exe .


Usage
=====

If you have installed Winpdb Reborn, the simplest way to launch it is::

    $ python -m winpdb my_program.py

or even::

    $ winpdb my_program.py

Find out about the other command-line options with ``–-help`` .

Documentation
=============

Use the ``-h`` or ``--help``  command-line flag for command-line help.

Inside Winpdb/Rpdb2 console, use the ``help`` command for detailed description of
debugger commands.

Online documentation is available at: http://www.winpdb.org/docs

An introduction to Winpdb usage, by Pr Norm Matloff: http://heather.cs.ucdavis.edu/%7Ematloff/winpdb.html

A detailed Winpdb tutorial is also available at: https://code.google.com/archive/p/winpdb/wikis/DebuggingTutorial.wiki

Community
=========

You can ask questions about Winpdb Reborn on the dedicated Google group:
https://groups.google.com/forum/#!forum/winpdb

Feel free to raise issues or propose improvements on the GitHub repository: https://github.com/bluebird75/winpdb/issues

|stats|

.. |Build Status Linux| image:: https://travis-ci.org/bluebird75/winpdb.svg?branch=winpdb
   :target: https://travis-ci.org/bluebird75/winpdb
.. |Build Status Windows| image:: https://ci.appveyor.com/api/projects/status/l3a98gaeamkgwrl7?svg=true&passingText=Windows%20Build%20passing&failingText=Windows%20Build%20failed
   :target: https://ci.appveyor.com/project/bluebird75/winpdb
.. |stats| image:: https://stats.sylphide-consulting.com/piwik/piwik.php?idsite=38&rec=1
