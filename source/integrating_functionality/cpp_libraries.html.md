---
layout: documentation
title: C++ Library Packages
sort_info: 10
---

# C++ Library Packages
{:.no_toc}

- TOC
{:toc}

While Rock is designed to allow you for the separation between functionality
and framework, if you feel so inclined, Rock provides a C++ library template.
This template solves some of the common problems with setting up a C++ library
(build system, ...) and obviously integrate as-is with the rest of a Rock
system.

**However** there is nothing that forces you to use the Rock library template.
Autoproj generically integrates with autotools and cmake packages. One can also
set up a custom package for more exotic systems (such as some old packages that
use "plain make")

The only constraint when the aim is to create a library that will be integrated
in a Rock component is to provide a pkg-config file for it. This is how orogen
resolves its dependencies.

## Integrating 3rd-party library

Rock packages - even those using the Rock CMake macros - are **not** dependent
on autoproj. Autoproj is an external helper tool, but in no way does it interact
with the package's build process. It is therefore perfectly feasible to build and
use 3rd party libraries in a Rock system.

When doing so, try to follow these guidelines:

- do not change the package unless strictly necessary. The one exception for
  which there is currently no good solution is to provide a pkg-config file for
  orogen integration. If that is needed, you will probably want to integrate
  this change [as a patch](../workspace/add_packages.html#patch) into your build
  configuration. Also, try to get this patch in the package's mainline, it will
  make things easier down the line.
- provide [a package manifest](../workspace/add_packages.html#manifest_xml) in the package set.

## Creating a Package

You need to first pick a category and a name [the workspace
conventions](../workspace/conventions.html) for information about how to name
your library package.

Rock then provides a library template. One can create a new library with

~~~
rock-create-lib library_dir
~~~

e.g.

~~~
cd drivers
rock-create-lib imu_advanced_navigation_anpp
~~~

This library creates a dummy class and a dummy executable that uses this class.
It is great at providing you with the example CMake code for both library and
tests.

## Conventions for library design

There's a small number of conventions that Rock libraries follow:

- **Extensions** header files are `.hpp`, source `.cpp`
- **Naming** classes should be `CamelCase`. The library must be defined under a
  namespace that matches the package basename (e.g.
  `imu_advanced_navigation_anpp` for `drivers/imu_advanced_navigation_anpp`). Each class
  has its own file, with named like the class (i.e. the `Driver` class is in
  `src/Driver.hpp` and `src/Driver.cpp`)
- **File Structure** source and header files _tests excluded_ are saved in
  `src/`. Tests are in `test/`.

If the ultimate goal of a data type is to be used as an interface type on a
Rock component, you must have first read and understood the [limits this entails system](../type_system)

## Tests

This is 2017 (or later). Testing is now an integral part of modern development
process, and Rock provides support to integrate unit testing in the development
workflow.

All libraries that have a `test/` folder will be assumed to have a test suite.
However, testing is disabled by default - since building all the tests from all
the workspace's packages would be fairly expensive. One needs to enable a package
tests to build them:

~~~
acd package/name
autoproj test enable .
aup
amake
~~~

The `aup` step is needed if the package has test-specific dependencies, as
defined by the `test_depend` tag of its [manifest
file](../workspace/add_packages.html#manifest_xml).

Once the tests are built, run them manually if you want to see their results.
Autoproj can also run them with `autoproj test [package]`, but will redirect
the test's output to a log file (that can be visualized later with
[alog](../basics/day_to_day.html#alog).

## The Rock CMake macros

To ease the use of CMake within a Rock system - i.e. in packages that follow
Rock conventions, Rock provides CMake macros that are somewhat easier to use.
The following describes them. The macros can be found in
`base/cmake/modules/Rock.cmake` in a rock installation. There are also specific
support for other tools within the Rock system (such as
[vizkit3d](todo_link_to_vizkit3d)), but these will be introduced at another point.

## Using Rock's text logger

Rock provides a logger that facilitates and unifies
debugging of system libraries. The following section
describes the functionality of the base logger and how to
use it.

When using the Rock CMake Macros, integration of the base logger is fairly
easy.  All you need to do is adding _base-logging to your list of package config
dependencies and include the header _base/logging/Logging.hpp_ where logging is needed. 

~~~
rock_library(test_library 
         DEPS_PKGCONFIG base-logging
         ...)
~~~

and

~~~
#include <base-logging/Logging.hpp>
~~~

The logger allows you to log either using a printf-style function, or
stream-style.

~~~ cpp
#include <base-logging/Logging.hpp>

...
// printf-style logging
std::string information("test-logging happening here");
LOG_DEBUG("I provide you the following debug information: %s", 
    information.c_str());
LOG_INFO(...);
LOG_WARN(...);
LOG_ERROR(..);
LOG_FATAL(..);

// stream-style logging
LOG_DEBUG_S << information;
LOG_INFO_S << information;
...
LOG_FATAL << information;
~~~

The default output format of a log message looks like the following (without linebreak):

~~~
[20120120-11:58:00:577] [FATAL] - test_logger::A fatal error occurred (/tmp/test_logger/src/Main.cpp:10 - int main(int, char**))
~~~

The logger can be configured at compile time to remove unwanted logging
statement in a release version, and at runtime to control the verbosity level
and logging format.

**At compile time**, the following defines can be set to configure the logger:

* **BASE_LOG_NAMESPACE** a string that will be displayed in all log messages.
  Set it to a unique string that allows you to filter the logging output. The
  Rock CMake macros set it to the package name.
* **BASE_LOG_DISABLE** set to ON to deactivates all logging statements, e.g.
  for a realease binary.
* **BASE_LOG_xxx** compile only logs equal or higher than a certain level xxx
  into your binary, i.e. BASE_LOG_WARN disable all logs except for WARN, ERROR
  and FATAL. Since the remaining levels are compiled out, the runtime settings
  below will be limited to the remaining levels.
* **BASE_LONG_NAMES** set this define in order to enforce a BASE_ prefix for
  the logging Macros, to avoid clashes with other libraries, i.e.
  BASE_LOG_DEBUG(...)

**At runtime**, the following environment variables can be used to control the behaviour of the logger: 

* **BASE_LOG_LEVEL** Set to one of DEBUG, INFO, WARN, ERROR or FATAL to define
  the maximum logging level, e.g. export BASE_LOG_LEVEL="DEBUG"
* **BASE_LOG_COLOR** Activate colored logging (current configuration best for
  dark background coloured terminals), e.g. export BASE_LOG_COLOR
* **BASE_LOG_FORMAT** Select from a set of predefined logging formats: DEFAULT,
  MULTILINE, SHORT, example outputs are given below:

The DEFAULT format contains no linebreaks (here only due to fit the narrow presentation):

~~~ sh
[20120120-11:58:00:577] [FATAL] - test_logger::A fatal error 
occurred (/tmp/test_logger/src/Main.cpp:10 - int main(int, char**))
~~~
    
The SHORT format reduces information to the log priority and the message:

~~~ sh
[FATAL] - test_logger::A fatal error occurred
~~~
    
The MULTILINE format split each individual log message across a minimum of three lines:

~~~ sh
[20120120-11:58:59:976] in int main(int, char**)
        /tmp/test_logger/src/Main.cpp:10
        [FATAL] - test_logger::A fatal error occurred 
~~~


The end of this section will detail the CMake macros. Unless you need them, you may
want to go to the next topic: [the integration of Ruby packages](ruby_libraries.html)
{: .next-page}

## Rock.cmake Reference Documentation

### Executable Targets (`rock_executable`)

~~~
rock_executable(name
    SOURCES source.cpp source1.cpp ...
    [LANG_C]
    [DEPS target1 target2 target3]
    [DEPS_PKGCONFIG pkg1 pkg2 pkg3]
    [DEPS_CMAKE pkg1 pkg2 pkg3]
    [MOC qtsource1.hpp qtsource2.hpp])
~~~

Creates a C++ executable and (optionally) installs it.

The following arguments are mandatory:

**SOURCES**: list of the C++ sources that should be built into that library

The following optional arguments are available:

**LANG_C**: build as a C rather than a C++ library

**DEPS**: lists the other targets from this CMake project against which the
library should be linked

**DEPS_PKGCONFIG**: list of pkg-config packages that the library depends upon. The
necessary link and compilation flags are added

**DEPS_CMAKE**: list of packages which can be found with CMake's find_package,
that the library depends upon. It is assumed that the Find*.cmake scripts
follow the CMake accepted standard for variable naming

**MOC**: if the library is Qt-based, this is a list of either source or header
files of classes that need to be passed through Qt's moc compiler.  If headers
are listed, these headers should be processed by moc, with the resulting
implementation files are built into the library. If they are source files, they
get added to the library and the corresponding header file is passed to moc.

### Library Targets (`rock_library`)

~~~
rock_library(name
    [SOURCES source.cpp source1.cpp ...]
    [HEADERS header1.hpp header2.hpp header3.hpp ...]
    [LANG_C]
    [DEPS target1 target2 target3]
    [DEPS_PKGCONFIG pkg1 pkg2 pkg3]
    [DEPS_CMAKE pkg1 pkg2 pkg3]
    [MOC qtsource1.hpp qtsource2.hpp]
    [NOINSTALL])
~~~

Creates and (optionally) installs a shared library.

As with all rock libraries, the target must have a pkg-config file along, that
gets generated and (optionally) installed by the macro. The pkg-config file
needs to be in the same directory and called package_name.pc.in. See the template
created by `rock-create-lib` for an example.

The following arguments are mandatory:

**SOURCES**: list of the C++ sources that should be built into that library. If
absent, the library is assumed to be header-only (i.e. only the headers and
pkg-config file will be installed). Note that even in this case the DEPS_* arguments
can be provided as they are passed to the pkg-config file generation.

**HEADERS**: list of the C++ headers that should be installed. Headers are installed
in `include/<target_name>/`.

The following optional arguments are available:

**LANG_C**: build as a C rather than a C++ library

**DEPS**: lists the other targets from this CMake project against which the
library should be linked

**DEPS_PKGCONFIG**: list of pkg-config packages that the library depends upon. The
necessary link and compilation flags are added

**DEPS_CMAKE**: list of packages which can be found with CMake's find_package,
that the library depends upon. It is assumed that the Find*.cmake scripts
follow the CMake accepted standard for variable naming

**MOC**: if the library is Qt-based, this is a list of either source or header
files of classes that need to be passed through Qt's moc compiler.  If headers
are listed, these headers should be processed by moc, with the resulting
implementation files are built into the library. If they are source files, they
get added to the library and the corresponding header file is passed to moc.

**NOINSTALL**: by default, the library gets installed on 'make install'. If this
argument is given, this is turned off

### Boost Test Suite Targets (`rock_testsuite`)

~~~
rock_testsuite(name
    SOURCES source.cpp source1.cpp ...
    [LANG_C]
    [DEPS target1 target2 target3]
    [DEPS_PKGCONFIG pkg1 pkg2 pkg3]
    [DEPS_CMAKE pkg1 pkg2 pkg3]
    [MOC qtsource1.hpp qtsource2.hpp])
~~~

Creates a C++ test suite that is using the boost unit test framework

The following arguments are mandatory:

**SOURCES**: list of the C++ sources that should be built into that library

The following optional arguments are available:

**LANG_C**: build as a C rather than a C++ library

**DEPS**: lists the other targets from this CMake project against which the
library should be linked

**DEPS_PKGCONFIG**: list of pkg-config packages that the library depends upon. The
necessary link and compilation flags are added

**DEPS_CMAKE**: list of packages which can be found with CMake's find_package,
that the library depends upon. It is assumed that the Find*.cmake scripts
follow the CMake accepted standard for variable naming

**MOC**: if the library is Qt-based, this is a list of either source or header
files of classes that need to be passed through Qt's moc compiler.  If headers
are listed, these headers should be processed by moc, with the resulting
implementation files are built into the library. If they are source files, they
get added to the library and the corresponding header file is passed to moc.

### GTest Test Suite Targets (`rock_testsuite`)

~~~
rock_gtest(name
    SOURCES source.cpp source1.cpp ...
    [LANG_C]
    [DEPS target1 target2 target3]
    [DEPS_PKGCONFIG pkg1 pkg2 pkg3]
    [DEPS_CMAKE pkg1 pkg2 pkg3]
    [MOC qtsource1.hpp qtsource2.hpp])
~~~

Creates a C++ test suite that is using the Google unit test framework

The following arguments are mandatory:

**SOURCES**: list of the C++ sources that should be built into that library

The following optional arguments are available:

**LANG_C**: build as a C rather than a C++ library

**DEPS**: lists the other targets from this CMake project against which the
library should be linked

**DEPS_PKGCONFIG**: list of pkg-config packages that the library depends upon. The
necessary link and compilation flags are added

**DEPS_CMAKE**: list of packages which can be found with CMake's find_package,
that the library depends upon. It is assumed that the Find*.cmake scripts
follow the CMake accepted standard for variable naming

**MOC**: if the library is Qt-based, this is a list of either source or header
files of classes that need to be passed through Qt's moc compiler.  If headers
are listed, these headers should be processed by moc, with the resulting
implementation files are built into the library. If they are source files, they
get added to the library and the corresponding header file is passed to moc.

**Next** let's look at the creation of [ruby packages](ruby_libraries.html)
