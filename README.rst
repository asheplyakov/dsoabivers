=============================================
Shared libraries versioning on Linux, Solaris
=============================================


Contents
========

#. `Why shared libraries should be versioned?`_
#. `ABI`_
#. `ELF`_
#. `ABI evolution`_
#. `SONAME ABI versioning`_
#. `Advanced: versioned symbols`_


Why shared libraries should be versioned?
=========================================

Library interfaces (API, ABI) are *protocols*, they must be versioned
just like any other protocol (think of network protocols, DB schemas, etc).

Unlike network protocols dynamic libraries have two interfaces:

#. API: interface available at the compile time
#. ABI: interface available at the run time

The pitfal: a compatible change in API (such the app code can be recompiled
with a new version of the library without any changes) might yield
an *INCOMPATIBLE* change of the ABI (so the binary linked with a previous
version of the library won't work with a new version of a dynamic library).


Demo: what happens when the ABI gets silently broken
----------------------------------------------------

* Compile initial version of libfoo and a binary fooclient::

    cd libfoo
    git checkout wrong_v0
    make -f GNUMakefile -j2

* Run `fooclient`::

    ./bin/fooclient a b c
    B::foo() s: ./bin/fooclient
    B::foo() s: a
    B::foo() s: b
    B::foo() s: c

* Introduce compatible ABI change::

    cp -a bin/fooclient bin/fooclient_old
    git checkout wrong_v1_compat
    make -f GNUMakefile -j2 lib

* Run the old `fooclient` binary::

    ./bin/fooclient_old a b c
    B::foo() s: ./bin/fooclient
    B::foo() s: a
    B::foo() s: b
    B::foo() s: c

* Introduce slightly incompatible ABI change, recompile the library::

    git checkout wrong_v2
    make -f GNUMakefile -j2

* Run the old `fooclient` binary::

    ./bin/fooclient_old a b c
    ./bin/fooclient_old: Symbol `_ZTV1X' has different size in shared object, consider re-linking
    B::foo() s: ./bin/fooclient_old
    B::foo() s: a
    B::foo() s: b
    B::foo() s: c

* Introduce incompatible ABI change, recompile the library::

    git checkout wrong_v3
    make -f GNUMakefile -j2

* Try running the old `fooclient` binary::

    ./bin/fooclient_old a b c
    ./bin/fooclient_old: Symbol `_ZTV1X' has different size in shared object, consider re-linking
    B::foo() s: ./bin/fooclient_old
    B::foo() s: a
    B::foo() s: b
    B::foo() s: c

* Introduce another incompatible ABI change, recompile the library::

    git checkout wrong_v4_oops
    make -f GNUMakefile -j2

* Try running the old `fooclient` binary::

    ./bin/fooclient_old a b c
    ./bin/fooclient_old: Symbol `_ZTV1X' has different size in shared object, consider re-linking
    Segmentation fault (core dumped)

All of above changes keep API compatible (the app source still compiles
without any changes), however most of these changes break ABI compatibility.


ABI
===

Application Binary Interface: set of interfaces available at the run time
including, but not limited to:

* Formats of executables and shared libraries
* Calling conventions (which arguments are passed via registers/stack,
  scratch registers versus saved registers),
* List of public symbols: functions, methods, data
* Data layout: member pointers, alignment, size of structures
* Name mangling scheme
* `vtable` location and layout
* `typeinfo` pointer(s) location and layout

and so on, see itanium-cpp-ABI_, sysV-ABI_.

.. _itanium-cpp-ABI: https://itanium-cxx-abi.github.io/cxx-abi/abi.html
.. _sysV-ABI: http://www.sco.com/developers/devspecs/gabi41.pdf

Shared library versioning == ABI versioning.
The ABI version has **NOTHING TO DO** with the software release number
(as in apache version 2.4.18 supports HTTP version 1.1).


ELF
===

Executable and Linkable Format

* Consists of the header and arbitrary number of `sections`
* Two mandatory tables:

  - program header table: describes program `segments`
  - section header table: describes the file `sections`

Segment: continous region of the process address space
Section: continous region of the ELF file

Typical sections:

* `.text` the program code
* `.rodata` string constants
* `.data` global variables
* `.bss` zero-initialized variables (arrays)
* `.interp` path to the run time linker (ELF interpreter)

.. _Executable and Linkable Format: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format


Tools for examining ELF: objdump, nm
------------------------------------

Dump all headers::

  $ objdump -x /bin/bash

Which shared libraries are required for a binary/library::

  $ objdump -p /bin/bash | grep NEEDED
    NEEDED               libtinfo.so.5
    NEEDED               libdl.so.2
    NEEDED               libc.so.6

Which dynamic symbols are exported/referenced by a shared library::

  $ nm -B -D -C /usr/lib/x86_64-linux-gnu/libstdc++.so.6

* `T` exported symbol from the `.text` section -- function, method
* `U` undefined symbols (presumably should be defined in `NEEDED` DSOs)
* `W` weak exported symbols
* `V` weak objects

(see `man nm` for more info)


ABI evolution
=============


BREAKING ABI
------------

* Remove or unexport exported class(es).

* Change type hierarchy in any way (add, remove, or reorder base classes).

* For a template classes: change the template arguments (add, remove, reorder).

* For virtual methods:

  - Add a virtual method to a class which has no other virtual methods or virtual bases.
  - Add new virtual method to non-leaf class (in particular to class which is designed
    to be derived from by library clients).
  - Change the order of virtual methods in the class declaration.
  - Override existing virtual method which is not in the primary base class
  - Remove a virtual method, even if it's a reimplementation of a virtual method
    from the base class
  - Override an existing virtual function if the overriding function has a covariant
    return type for which the more-derived type has a pointer address different
    from the less-derived one (usually happens when, between the less-derived and
    the more-derived ones, there's multiple inheritance or virtual inheritance).

* Changing a method/function signature:

  - changing any types of the arguments in the parameter list, including changing
    const/volatile qualifiers of existing parameters
  - changing const/volatile qualifiers of the method/function
  - extending a method with another parameter, even if it has a default value
  - changing access rights (say, from `private` to `public`)
  - changing the return type in any way


Backward compatible ABI changes
-------------------------------

* Add new class(es).
* Add or remove friend declarations to classes.
* Add new non-virtual methods (including constructors).
* Add a new enum to a class.
* Remove private non-virtual functions if they are not called by any inline
  functions (and have never been).
* Reimplement virtual functions defined in the primary base class (first non-virtual
  base class, or first non-virtual parent of the base class, etc) IF it's safe
  for prior versions to call implementation in the base class rather than in 
  derived ones.
* When overriding methods with a `covariant return type`_ more-dervied type
  must have the same pointer address as the less-dervied one.

For a more detailed list see `KDE ABI policy`_

.. _KDE ABI policy: https://community.kde.org/Policies/Binary_Compatibility_Issues_With_C%2B%2B#Note_about_ABI
.. _covariant return type: http://en.wikipedia.org/wiki/Covariant_return_type


Every library no matter how carefully designed breaks ABI at certain point.
How to properly inform users (programs as opposed to humans) about an incompatible
ABI change?


SONAME ABI versioning
=====================

Goals: 

* Avoid relinking client apps/libraries on compatible changes
* Clearly mark incompatible changes
* Application which need incompatible versions of library can coexist

::

  objdump -p /lib/x86_64-linux-gnu/libglib-2.0.so.0.4800.2 | grep SONAME
  SONAME               libglib-2.0.so.0

`SONAME` is similar to a protocol name ("HTTP", "FIX", "X11"), in general
it does *NOT* match the library filename (`libglib-2.0.so.0.4800.2`).
When the binary is linked with a shared library it's the `SONAME` of
the library which gets recorded as a dependency::

  objdump -p /usr/bin/vim.gtk | grep NEEDED | grep glib
    NEEDED               libglib-2.0.so.0

::

  ls -1 -l /usr/lib/x86_64-linux-gnu/libglib-2.0.so
  lrwxrwxrwx 1 root root 38 Jan  6  2017 /usr/lib/x86_64-linux-gnu/libglib-2.0.so -> /lib/x86_64-linux-gnu/libglib-2.0.so.0
  ls -1 -l /lib/x86_64-linux-gnu/libglib-2.0.so*
  lrwxrwxrwx 1 root root      23 Jan  6  2017 /lib/x86_64-linux-gnu/libglib-2.0.so.0 -> libglib-2.0.so.0.4800.2
  -rw-r--r-- 1 root root 1115136 Jan  6  2017 /lib/x86_64-linux-gnu/libglib-2.0.so.0.4800.2

* `libglib-2.0.so` used by the compile time linker only (-lglib-2.0),
  usually this symlink points to the latest available SONAME version
  of the library (libglib-2.0.so.0).

* `libglib-2.0.so.0` -- SONAME symlink, used by the dynamic linker,
  points to the latest *COMPATIBLE* version of the library

* `libglib-2.0.so.0.4800.2` -- the actual DSO (shared library), it's
  revision is `4800`, and patchlevel version is `2`

Why such indirection? Historically *NIX'es had troubles writing files
with public read-only mappings, hence the upgrade procedure was to

- install newer version into a different file (named after the revision
  and the patchlevel version)
- change the `SONAME` symlink to point to the newly installed file

This way the processes which use the previous version of the library
can continue uninterrupted, and the newly started processes will use
the upgraded library.


Rules of the game
-----------------

* When making a change which does not affect the ABI:

  - patchlevel++;

* When making a backward compatible ABI change:

  - revision++;
  - patchlevel = 0;

* When making an incompatible ABI change:
  
  - SONAME++;
  - revision = patchlevel = 0;


Practical implementation: CMake
-------------------------------

::
  set_target_properties(foo PROPERTIES SOVERSION X VERSION X.Y.Z)


Practical implementation: libtool
---------------------------------

In attempt to be portable libtool makes things even more confusing:

* LT_CURRENT: the most recent ABI version supported by the library
* LT_REVISION: sort of patchlevel version
* LT_AGE: number of compatible ABIs, that is, LT_CURRENT-LT_AGE is
  the oldest backward compatible ABI version supported by the library

::
  libfoo_la_LDFLAGS = -version-info $(LT_CURRENT):$(LT_REVISION):$(LT_AGE)
   

Advanced: versioned symbols
===========================

`SONAME ABI versioning`_ is inconvenient: a single incompatible change
requires SONAME bump, which forces re-linking the client apps (to use
the new version of the library), even if the app in question hasn't been
using the class (function) which has changed in an incompatible manner.

Just like a server can support multiple versions of the protocol a shared
library can support several ABI versions. Linux' and Solaris' linkers
support versioning of individual symbols::

  $ objdump -p /bin/bash | sed -rne '/^Version References:/,$ { p }'
  Version References:
    required from libdl.so.2:
      0x09691a75 0x00 10 GLIBC_2.2.5
    required from libtinfo.so.5:
      0x02a6c513 0x00 04 NCURSES_TINFO_5.0.19991023
    required from libc.so.6:
      0x06969191 0x00 11 GLIBC_2.11
      0x06969194 0x00 09 GLIBC_2.14
      0x0d696918 0x00 08 GLIBC_2.8
      0x06969195 0x00 07 GLIBC_2.15
      0x0d696914 0x00 06 GLIBC_2.4
      0x09691974 0x00 05 GLIBC_2.3.4
      0x0d696913 0x00 03 GLIBC_2.3
      0x09691a75 0x00 02 GLIBC_2.2.5

* When adding a new function, mark them with a new version
* When changing an existing function `foo` in a incompatible manner::

  - rename existing function to `foo_old`
  - write the new code into `foo_new`
  - export `foo_new` as `foo` version N+1, where N is a previous version of `foo`
  - export `foo_old` as `foo` version N
  - set the default version of `foo` to N+1
* Increment the patchlevel version of the library

Advantages: 

* dependency on specific *compatible* version of the ABI can be recorded
* backward compatibility can be maintained over a long time

Disadvantages:

* It's tricky, especially for C++ libraries with non-trivial class hierarchies
  (in fact the only C++ library which uses ELF symbol versioning is GCC's libstdc++)

