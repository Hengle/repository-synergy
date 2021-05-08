pyjaco - Python to JavaScript compiler
======================================

web page: http://pyjaco.org

mailing list: http://groups.google.com/group/pyjaco

Installation
------------

There are several ways to install pyjaco. Since pyjaco is still under
development we recommend to use virtualenv. Although pythons standard
installation shall work fine.

virtualenv::

    git clone https://github.com/chrivers/pyjaco
    virtualenv nameOfVenv
    cd pyjaco
    ../nameOfVenv/bin/python setup.py install

standard::

    git clone https://github.com/chrivers/pyjaco
    cd pyjaco
    python setup.py install
   
.. case 3 is maybe outdated?

make::

    git clone https://github.com/chrivers/pyjaco
    cd pyjaco
    make


Usage
-----
In general there are two ways to use pyjaco. The first is to compile a 
Python file to a JavaScript file and the second is to invoke the 
compiler into a python script.

py file to js file (with and without virtualenv)::
    
    pyjs.py [options] <infile> -o <outfile>
    
    pathToVenv/bin/pyjs.py [options] <infile> -o <outfile>

Where <infile> is the name of a file that will be compiled.
If the infile is a directory, all files in that directory that have
an extension of .py or .pyjaco will be compiled to .js files into the output 
directory. 

to generate the buildins, run::

    pyjs.py -b generate

For more parameter see the options section.

invoke the compiler into a python program::

    import sys
    from pyjaco import Compiler
    code = open("tojs.py", "r")
    compiler = Compiler()
    compiler.append_string(code.read())
    print str(compiler)



Examples
--------

Like explained previously there are 2 ways to use pyjaco. In the pyjaco folder 
you can find a few examples that you can use for testing.

py file to js file::

    pyjs.py tests/algorithms/fib.py -o fib.js


invoked::

    python examples/gol.py > examples/gol.html
    firefox examples/gol.html


If it doesn't work, it's a bug. 


Options
-------

available options::

  -h, --help            show this help message and exit
  -o OUTPUT, --output=OUTPUT
                        write output to OUTPUT, can be a file or directory
  -q, --quiet           Do not print informative notes to stderr
  -b BUILTINS, --builtins=BUILTINS
                        INCLUDE builtins statically in each file 
                        IMPORT builtins using a load statement in each file 
                        GENERATE a separate file for builtins (output must be a directory) 
                        NONE don't include builtins
  -I, --import          IMPORT builtins using a load statement in each file
                        This is an alias for -b import
  -w, --watch           Watch the input files for changes and recompile. If
                        the input file is a single file, watch it for changes
                        and recompile. If a directory, recompile if any .py or
                        .pyjaco files in the directory have changes.


Tests
-----

Pyjaco brings a test facility with it. However, to run the test sets a JS console
is necessary. See here: http://code.google.com/p/v8/

Run all tests, that are supposed to work. ::

    ./run_tests.py

Run all tests with "foo" in the name. Useful for running a group of tests. ::

    ./run_tests.py foo

Run all tests including those that are known to fail (currently). It
should be understandable from the output. ::

    ./run_tests.py -a

Run tests but ignore if an error is raised by the test. This is not
affecting the error generated by the test files in the tests directory. ::

    ./run_tests.py -x
    or
    ./run_tests.py --no-error

For more flags then described here, use -h option ::

    ./run_tests.py -h
    

Single test cases
-----------------

With the "casetest" script you can keep comparing the output of pyjaco to
the output of python on the same python script. It's very useful if you are
debugging the compiler or standard library.

./casetest foo.py

Will run foo.py through python and pyjaco, and display the differences. It
will then display a line of "#", and wait for you to press enter to do another
iteration. When the files match, casetest will exit.


License
-------

Free Software. See the LICENSE file for exact details.