.. image:: https://github.com/Maratyszcza/PeachPy/blob/master/logo/peachpy.png
  :alt: PeachPy logo
  :align: center

===========================================================================
Portable Efficient Assembly Code-generator in Higher-level Python (PeachPy)
===========================================================================

.. image:: https://img.shields.io/badge/License-BSD%202--Clause%20%22Simplified%22%20License-blue.svg
  :alt: PeachPy License: Simplified BSD
  :target: https://github.com/Maratyszcza/PeachPy/blob/master/LICENSE.rst

.. image:: https://travis-ci.org/Maratyszcza/PeachPy.svg?branch=master
  :alt: Travis-CI Build Status
  :target: https://travis-ci.org/Maratyszcza/PeachPy/

.. image:: https://ci.appveyor.com/api/projects/status/p64ew9in189bu2pl?svg=true
  :alt: AppVeyor Build Status
  :target: https://ci.appveyor.com/project/MaratDukhan/peachpy

PeachPy is a Python framework for writing high-performance assembly kernels.

PeachPy aims to simplify writing optimized assembly kernels while preserving all optimization opportunities of traditional assembly. Some PeachPy features:

- Universal assembly syntax for Windows, Unix, and Golang assembly.

  * PeachPy can directly generate ELF, MS COFF and Mach-O object files and assembly listings for Golang toolchain

- Automatic adaption of function to different calling conventions and ABIs.
  
  * Functions for different platforms can be generated from the same assembly source
  * Supports Microsoft x64 ABI, System V x86-64 ABI (Linux, OS X, and FreeBSD), Linux x32 ABI, Native Client x86-64 SFI ABI, Golang AMD64 ABI, Golang AMD64p32 ABI
      
- Automatic register allocation.
  
  * PeachPy is flexible and lets mix auto-allocated and hardcoded registers in the same code.

- Automation of routine tasks in assembly programming:

  * Function prolog and epilog and generated by PeachPy
  * De-duplication of data constants (e.g. `Constant.float32x4(1.0)`)
  * Analysis of ISA extensions used in a function

- Supports x86-64 instructions up to AVX-512 and SHA
  
  * Including 3dnow!+, XOP, FMA3, FMA4, TBM and BMI2.
  * Excluding x87 FPU and most system instructions.
  * Rigorously tested with `auto-generated tests <https://github.com/Maratyszcza/PeachPy/tree/master/test/x86_64/encoding>`_ to produce the same opcodes as binutils.

- Auto-generation of metadata files

  * Makefile with module dependencies (`-MMD` and `-MF` options)
  * C header for the generated functions
  * Function metadata in JSON format

- Python-based metaprogramming and code-generation.
- Multiplexing of multiple instruction streams (helpful for software pipelining).
- Compatible with Python 2 and Python 3, CPython and PyPy.

Online Demo
-----------

You can try online demo on `PeachPy.IO <http://www.peachpy.io>`_

Installation
------------

PeachPy is actively developed, and thus there are presently no stable releases of 0.2 branch. We recommend that you use the `master` version:

.. code-block:: bash

  pip install --upgrade git+https://github.com/Maratyszcza/PeachPy

Installation for development
****************************

If you plan to modify PeachPy, we recommend the following installation procedure:

.. code-block:: bash

  git clone https://github.com/Maratyszcza/PeachPy.git
  cd PeachPy
  python setup.py develop


Using PeachPy as a command-line tool
------------------------------------

.. code-block:: python
  
  # These two lines are not needed for PeachPy, but will help you get autocompletion in good code editors
  from peachpy import *
  from peachpy.x86_64 import *

  # Lets write a function float DotProduct(const float* x, const float* y)
  
  # If you want maximum cross-platform compatibility, arguments must have names
  x = Argument(ptr(const_float_), name="x")
  # If name is not specified, it is auto-detected
  y = Argument(ptr(const_float_))

  # Everything inside the `with` statement is function body
  with Function("DotProduct", (x, y), float_,
    # Enable instructions up to SSE4.2
    # PeachPy will report error if you accidentially use a newer instruction
    target=uarch.default + isa.sse4_2):
  
    # Request two 64-bit general-purpose registers. No need to specify exact names.
    reg_x, reg_y = GeneralPurposeRegister64(), GeneralPurposeRegister64()

    # This is a cross-platform way to load arguments. PeachPy will map it to something proper later.
    LOAD.ARGUMENT(reg_x, x)
    LOAD.ARGUMENT(reg_y, y)

    # Also request a virtual 128-bit SIMD register...
    xmm_x = XMMRegister()
    # ...and fill it with data
    MOVAPS(xmm_x, [reg_x])
    # It is fine to mix virtual and physical (xmm0-xmm15) registers in the same code
    MOVAPS(xmm2, [reg_y])

    # Execute dot product instruction, put result into xmm_x
    DPPS(xmm_x, xmm2, 0xF1)

    # This is a cross-platform way to return results. PeachPy will take care of ABI specifics.
    RETURN(xmm_x)

Now you can compile this code into a binary object file that you can link into a program...

.. code-block:: bash

  # Use MS-COFF format with Microsoft ABI for Windows
  python -m peachpy.x86_64 -mabi=ms -mimage-format=ms-coff -o example.obj example.py
  # Use Mach-O format with SysV ABI for OS X
  python -m peachpy.x86_64 -mabi=sysv -mimage-format=mach-o -o example.o example.py
  # Use ELF format with SysV ABI for Linux x86-64
  python -m peachpy.x86_64 -mabi=sysv -mimage-format=elf -o example.o example.py
  # Use ELF format with x32 ABI for Linux x32 (x86-64 with 32-bit pointer)
  python -m peachpy.x86_64 -mabi=x32 -mimage-format=elf -o example.o example.py
  # Use ELF format with Native Client x86-64 ABI for Chromium x86-64
  python -m peachpy.x86_64 -mabi=nacl -mimage-format=elf -o example.o example.py

What else? You can convert the program to Plan 9 assembly for use with Go programming language:

.. code-block:: bash

  # Use Go ABI (asm version) with -S flag to generate assembly for Go x86-64 targets
  python -m peachpy.x86_64 -mabi=goasm -S -o example_amd64.s example.py
  # Use Go-p32 ABI (asm version) with -S flag to generate assembly for Go x86-64 targets with 32-bit pointers
  python -m peachpy.x86_64 -mabi=goasm-p32 -S -o example_amd64p32.s example.py

If Plan 9 assembly is too restrictive for your use-case, generate ``.syso`` objects `which can be linked into Go programs <https://github.com/golang/go/wiki/GcToolchainTricks#use-syso-file-to-embed-arbitrary-self-contained-c-code>`_:

.. code-block:: bash

  # Use Go ABI (syso version) to generate .syso objects for Go x86-64 targets
  # Image format can be any (ELF/Mach-O/MS-COFF)
  python -m peachpy.x86_64 -mabi=gosyso -mimage-format=elf -o example_amd64.syso example.py
  # Use Go-p32 ABI (syso version) to generate .syso objects for Go x86-64 targets with 32-bit pointers
  # Image format can be any (ELF/Mach-O/MS-COFF)
  python -m peachpy.x86_64 -mabi=gosyso-p32 -mimage-format=elf -o example_amd64p32.syso example.py

See `examples <https://github.com/Maratyszcza/PeachPy/tree/master/examples>`_ for real-world scenarios of using PeachPy with ``make``, ``nmake`` and ``go generate`` tools.

Using PeachPy as a Python module
--------------------------------

When command-line tool does not provide sufficient flexibility, Python scripts can import PeachPy objects from ``peachpy`` and ``peachpy.x86_64`` modules and do arbitrary manipulations on output images, program structure, instructions, and bytecodes.

PeachPy as Inline Assembler for Python
**************************************

PeachPy links assembly and Python: it represents assembly instructions and syntax as Python classes, functions, and objects.
But it also works the other way around: PeachPy can represent your assembly functions as callable Python functions!

.. code-block:: python

  from peachpy import *
  from peachpy.x86_64 import *

  x = Argument(int32_t)
  y = Argument(int32_t)

  with Function("Add", (x, y), int32_t) as asm_function:
      reg_x = GeneralPurposeRegister32()
      reg_y = GeneralPurposeRegister32()

      LOAD.ARGUMENT(reg_x, x)
      LOAD.ARGUMENT(reg_y, y)

      ADD(reg_x, reg_y)

      RETURN(reg_x)

  python_function = asm_function.finalize(abi.detect()).encode().load()

  print(python_function(2, 2)) # -> prints "4"

PeachPy as Instruction Encoder
******************************

PeachPy can be used to explore instruction length, opcodes, and alternative encodings:

.. code-block:: python

  from peachpy.x86_64 import *

  ADD(eax, 5).encode() # -> bytearray(b'\x83\xc0\x05')

  MOVAPS(xmm0, xmm1).encode_options() # -> [bytearray(b'\x0f(\xc1'), bytearray(b'\x0f)\xc8')]
  
  VPSLLVD(ymm0, ymm1, [rsi + 8]).encode_length_options() # -> {6: bytearray(b'\xc4\xe2uGF\x08'),
                                                         #     7: bytearray(b'\xc4\xe2uGD&\x08'),
                                                         #     9: bytearray(b'\xc4\xe2uG\x86\x08\x00\x00\x00')}

Tutorials
---------

- `Writing Go assembly functions with PeachPy <https://blog.gopheracademy.com/advent-2016/peachpy/>`_ by `Damian Gryski <https://github.com/dgryski>`_

- `Adventures in JIT compilation (Part 4) <http://eli.thegreenplace.net/2017/adventures-in-jit-compilation-part-4-in-python/>`_ by `Eli Bendersky <https://github.com/eliben>`_

Users
-----

- `NNPACK <https://github.com/Maratyszcza/NNPACK>`_ -- an acceleration layer for convolutional networks on multi-core CPUs.

- `ChaCha20 <https://git.schwanenlied.me/yawning/chacha20>`_ -- Go implementation of ChaCha20 cryptographic cipher.

- `AEZ <https://git.schwanenlied.me/yawning/aez>`_ -- Go implemenetation of AEZ authenticated-encryption scheme.

- `bp128 <https://github.com/robskie/bp128>`_ -- Go implementation of SIMD-BP128 integer encoding and decoding.

- `go-marvin32 <https://github.com/dgryski/go-marvin32>`_ -- Go implementation of Microsoft's Marvin32 hash function.

- `go-highway <https://github.com/dgryski/go-highway>`_ -- Go implementation of Google's Highway hash function.

- `go-metro <https://github.com/dgryski/go-metro>`_ -- Go implementation of MetroHash function.

- `go-stadtx <https://github.com/dgryski/go-stadtx>`_ -- Go implementation of Stadtx hash function.

- `go-sip13 <https://github.com/dgryski/go-sip13>`_ -- Go implementation of SipHash 1-3 function.

- `go-chaskey <https://github.com/dgryski/go-chaskey>`_ -- Go implementation of Chaskey MAC.

- `go-speck <https://github.com/dgryski/go-speck>`_ -- Go implementation of SPECK cipher.

- `go-bloomindex <https://github.com/dgryski/go-bloomindex>`_ - Go implementation of Bloom-filter based search index.

- `go-groupvariant <https://github.com/dgryski/go-groupvarint>`_ - SSE-optimized group varint integer encoding in Go.

- `Yeppp! <http://www.yeppp.info>`_ performance library. All optimized kernels in Yeppp! are implemented in PeachPy (uses old version of PeachPy with deprecated syntax).

Peer-Reviewed Publications
--------------------------

- Marat Dukhan "PeachPy: A Python Framework for Developing High-Performance Assembly Kernels", Python for High-Performance Computing (PyHPC) 2013 (`slides <http://www.yeppp.info/resources/peachpy-slides.pdf>`_, `paper <http://www.yeppp.info/resources/peachpy-paper.pdf>`_, code uses deprecated syntax)

- Marat Dukhan "PeachPy meets Opcodes: Direct Machine Code Generation from Python", Python for High-Performance Computing (PyHPC) 2015 (`slides <http://www.peachpy.io/slides/pyhpc2015>`_, `paper on ACM Digital Library <https://dl.acm.org/citation.cfm?id=2835860>`_).

Other Presentations
-------------------

- Marat Dukhan "Developing Low-Level Assembly Kernels in PeachPy", presentation on `The First BLIS Retreat Workshop <https://www.cs.utexas.edu/users/flame/BLISRetreat/>`_, 2013 (`slides <https://www.cs.utexas.edu/users/flame/BLISRetreat/BLISRetreatTalks/PeachPy.pdf>`_, code uses deprecated syntax)

- Marat Dukhan "Porting BLIS micro-kernels to PeachPy", presentation on `The Third BLIS Retreat Workshop <https://www.cs.utexas.edu/users/flame/BLISRetreat2015/>`_, 2015 (`slides <http://www.peachpy.io/slides/blis-retreat-2015/>`_)

- Marat Dukhan "Accelerating Data Processing in Go with SIMD Instructions", presentation on `Atlanta Go Meetup <http://www.meetup.com/Go-Users-Group-Atlanta>`_, September 16, 2015 (`slides <https://docs.google.com/presentation/d/1MYg8PyhEf0oIvZ9YU2panNkVXsKt5UQBl_vGEaCeB1k/edit?usp=sharing>`_)

Dependencies
------------

- Nearly all instruction classes in PeachPy are generated from `Opcodes Database <https://github.com/Maratyszcza/Opcodes>`_

- Instruction encodings in PeachPy are validated against `binutils <https://www.gnu.org/software/binutils/>`_ using auto-generated tests

- PeachPy uses `six <https://pythonhosted.org/six/>`_ and `enum34 <https://pypi.python.org/pypi/enum34>`_ packages as a compatibility layer between Python 2 and Python 3

Acknowledgements
----------------

.. image:: https://github.com/Maratyszcza/PeachPy/blob/master/logo/hpcgarage.png
  :alt: HPC Garage logo
  :target: http://hpcgarage.org/

.. image:: https://github.com/Maratyszcza/PeachPy/blob/master/logo/college-of-computing.gif
  :alt: Georgia Tech College of Computing logo
  :target: http://www.cse.gatech.edu/

This work is a research project at the HPC Garage lab in the Georgia Institute of Technology, College of Computing, School of Computational Science and Engineering.

The work was supported in part by grants to Prof. Richard Vuduc's research lab, `The HPC Garage <www.hpcgarage.org>`_, from the National Science Foundation (NSF) under NSF CAREER award number 0953100; and a grant from the Defense Advanced Research Projects Agency (DARPA) Computer Science Study Group program

Any opinions, conclusions or recommendations expressed in this software and documentation are those of the authors and not necessarily reflect those of NSF or DARPA.