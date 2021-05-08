# **pytaglib**
[![CircleCI](https://img.shields.io/circleci/project/github/supermihi/pytaglib/master.svg)](https://circleci.com/gh/supermihi/pytaglib)
[![PyPI](https://img.shields.io/pypi/v/pytaglib.svg)](https://pypi.org/project/pytaglib/)

pytaglib is a [Python](http://www.python.org) audio tagging library. It is cross-platform, works with all Python versions, and is very simple to use yet fully featured:
 - [supports more than a dozen file formats](http://taglib.github.io) including mp3, flac, ogg, wma, and mp4,
 - support arbitrary, non-standard tag names,
 - support multiple values per tag.

pytaglib is a very thin wrapper (≈150 lines of [code](src/taglib.pyx)) around the fast and rock-solid [TagLib](http://taglib.github.io) C++ library.
## News
See the [Changelog](CHANGELOG.md).
## Get it
In most cases, you should install pytaglib with [pip](https://pip.pypa.io/en/stable/):

        pip install pytaglib

See [installation notes](#installation-notes) below for requirements and manual compilation.

## Usage

```python
>>> import taglib
>>> song = taglib.File("/path/to/my/file.mp3")
>>> song.tags
{'ARTIST': ['piman', 'jzig'], 'ALBUM': ['Quod Libet Test Data'], 'TITLE': ['Silence'], 'GENRE': ['Silence'], 'TRACKNUMBER': ['02/10'], 'DATE': ['2004']}

>>> song.length
239
>>> song.tags["ALBUM"] = ["White Album"] # always use lists, even for single values
>>> del song.tags["DATE"]
>>> song.tags["GENRE"] = ["Vocal", "Classical"]
>>> song.tags["PERFORMER:HARPSICHORD"] = ["Ton Koopman"] 
>>> song.save()
```
For detailed API documentation, use the docstrings of the `taglib.File` class or view the [source code](src/taglib.pyx) directly.


**Note:** pytaglib uses unicode strings (type `str` in Python 3 and `unicode` in Python 2) for both tag names and values. The library converts byte-strings to unicode strings on assignment, but it is recommended to provide unicode strings only to avoid encoding problems.


## `pyprinttags`
This package also installs the `pyprinttags` script. It takes one or more files as
command-line parameters and will display all known metadata of that files on the terminal.
If unsupported tags (a.k.a. non-textual information) are found, they can optionally be removed
from the file.

## Installation Notes

* Ensure that `pip` is installed and points to the correct Python version
  - on Windows, be sure to check *install pip* in the Python installer
  - on Debian/Ubuntu/Mint, install `python3-pip` (and/or `python-pip`)
  - you might need to type, e.g., `pip-3` to install pytaglib for Python 3 if your system's default is Python 2.7.
* For Windows users, there are some precompiled binary packages (wheels). See the [PyPI page](https://pypi.python.org/pypi/pytaglib) for a list of supported Python versions.
* If no binary packages exists, you need to have both Python and taglib installed with development headers (packages `python3-dev` (or `python-dev`) and `libtag1-dev` for debian / ubuntu and derivates, `python-devel` and `taglib-devel` for fedora and friends, `brew install taglib` on OS X).


### Linux: Distribution-Specific Packages
* Debian- and Ubuntu-based linux flavors have binary packages for the Python 3 version, called `python3-taglib`. Unfortunatelly, they are heavily outdated, so you should instally the recent version via `pip` whenever possible.
* For Arch users, there is a [package](https://aur.archlinux.org/packages/python-pytaglib/) in the user repository (AUR).

### Manual Compilation: General
You can download or checkout the sources and compile manually:

        python setup.py build
        python setup.py test  # optional, run unit tests
        sudo python setup.py install


**Note**: The `taglib` Python extension is built from [`taglib.cpp`](src/taglib.cpp) which in turn is
auto-generated by [Cython](http://www.cython.org) from [`taglib.pyx`](src/taglib.pyx).
To regenerate the `taglib.cpp` after making changes to `taglib.pyx`, set the environment variable `PYTAGLIB_CYTHONIZE` to `1` before calling `setup.py` or `pip`.

### Manual Compilation: Windows

The following procedure was tested for Python 3.5 and Python 3.6 on Windows 10. Other platforms might require different steps; see e.g. [this](https://blog.ionelmc.ro/2014/12/21/compiling-python-extensions-on-windows/) page.

1. Install [Microsoft Visual Studio 2015 Community Edition](https://www.visualstudio.com/downloads/download-visual-studio-vs). In the installation process, be sure to enable C/C++ support. Alternatively, install Visual Studio 2017, but install the "v140" C++ toolset and use the "Visual Studio 2015" version of the developer command prompt below.
2. Download and build taglib:
    1. Download the current [taglib release](https://github.com/taglib/taglib/releases) and extract it somewhere   on your computer.
    2. Start the VS2015 x64 Native Tools Command Prompt. On Windows 8/10, it might not appear in your start menu, but you can find it here: `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Visual Studio 2015\Visual Studio Tools\Windows Desktop Command Prompts`
    3. Navigate to the extracted taglib folder and type: `cmake -G "Visual Studio 14 2015 Win64" -DCMAKE_INSTALL_PREFIX=".\taglib-install"` to generate the Visual Studio project files.
    4. Type `msbuild INSTALL.vcxproj /p:Configuration=Release` which will "install" taglib into the `taglib-install` subdirectory.
3. Still in the VS2015 command prompt, navigate to the pytaglib directory.
4. Tell pytaglib where to find taglib: `set TAGLIB_HOME=C:\Path\To\taglib-install`
5. Build pytaglib: `python setup.py build` and install: `python setup.py install`



## Contact
For bug reports or feature requests, please use the
[issue tracker](https://github.com/supermihi/pytaglib/issues) on GitHub. For anything else, contact
me by [email](mailto:michaelhelmling@posteo.de).