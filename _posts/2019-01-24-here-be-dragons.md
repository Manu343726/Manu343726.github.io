---
title: Here be dragons
date: 2019-01-24
tags: [c++,conanio]
comments: true
---

As usually I like to start my blog posts reflecting a bit about my work.
For the past two years I have been trying to get a solution for C++ static
reflection that's cross platform and "clean" enough to be usable in real
world projects. Or at least to be usable in production by me and my team.
My first attempt was
[siplasplas](https://github.com/Manu343726/meetingcpp2016), but it didn't
scalate well at the end because of the lack of metadata codegen
customization (The One Ring To Rule Them All implementation didn't work as
expected...), lack of cross-compilation support (No1 requirement for me
since I work with ARM embedded systems), and the fact that I was using the
[libclang python
bindings](https://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang)
to write the code generation tool. I'm not a python expert, so debugging
performance issues in the codegen step was not easy for me (Most of these
issues coming from me writing non-optimal python code).

After Meeting C++ 2016 I decided to port the codegen tool to C++ writing
my own libclang C++ wrapper, but ultimately
[@foonathan](https://foonathan.net/) won the race with what has become
[cppast](https://github.com/foonathan/cppast). So there I was, with almost
two years of work going nowhere for different reasons. It was time to go
back to the start and analyzing my requirements from scratch.  
So as of September 2017 I started playing with a new concept: Static
reflection with an API as simple as possible, customizable metadata
independent from the reflection API, cross compilation support since day
1, and a codegen tool written with C++ and cppast:
[tinyrefl](https://github.com/Manu343726/tinyrefl)

I put tinyrefl in production and managed to get [a slot in a great C++
conference to tell my
story](https://github.com/Manu343726/meetingcpp2018). I'm no longer part
of the team developing the product **that uses tinyrefl as a core of its
frontend**, and I'm not receiveing almost any support call, so I would say
it's not going that bad after all.

All this story brings me to the center of this post: Dependencies. As you
may imagine, a reflection tool for C++ requires a parser, parser that in
this case depends on a **huge compiler framework: LLVM**. After working in
[a startup dedicated to C++
dependencies](https://www.reddit.com/r/cpp/comments/3h8o2r/biicode_c_dependency_manager_has_gone_out_of/)
I take the issue of C++ dependencies as something rather personal. So what
do I do now? Pull all my dependencies through hundreds of non-debuggeable
cmake scripts? Require everything to be installed in the system? Neither
of those, let's delegate all the hard problems to
[conan](https://conan.io/).

## Building components depending on LLVM

Let's make things clear: **LLVM is a framework**. It is not a set of C++
libraries, but a self-contained framework that's not designed to be built
separately. This becomes evident if you read the [official guide to build
Clang](https://clang.llvm.org/get_started.html):

> Check out LLVM [...] and checkout out Clang at `<llvm-repo>/tools/`

LLVM and its components (LLVM, Clang, compiler-rt, libc++, etc) use a lot
of common CMake scripts contained in the LLVM repo. Forget about writing
isolated libraries depending on each other. If you want to build LLVM from
sources you have to build **all of it**, and if you want to build Clang
you have to build it as part of LLVM or **build it out of LLVM
sourcetree** ***somehow***.

After almost a year and a half of trial and error, inheriting from some
[earlier
attempts](https://bintray.com/conan/conan-transit/llvm%3Asmspillaz),
I found a way to build each component separately in its own package:
**Patch the main CMakeLists.txt file of the component to import the CMake
scripts from LLVM**:

``` python
def source(self):
    replace_in_file(self._root_cmake_file, r'project\((.+?)\)',
r'''message(STATUS "Main {0} CMakeLists.txt (${{CMAKE_CURRENT_LIST_DIR}}) patched by conan")

project(\1)

message(STATUS "Loading conan scripts for {0} dependencies...")
include("${{CMAKE_BINARY_DIR}}/conanbuildinfo.cmake")
message(STATUS "Doing conan basic setup")
conan_basic_setup()
list(APPEND CMAKE_PROGRAM_PATH ${{CONAN_BIN_DIRS}})
message(STATUS "Conan setup done. CMAKE_PROGRAM_PATH: ${{CMAKE_PROGRAM_PATH}}")
'''.format(self.name), self.output)
```

The most important line is `list(APPEND CMAKE_PROGRAM_PATH
${{CONAN_BIN_DIRS}})` which adds the `bin/` directories of the conan
dependencies to the set of paths that CMake uses to search programs
through `find_program()` calls. This allows components such as Clang or
libc++ **to find llvm-config utility from LLVM**, which is compiled and
packaged by the LLVM build.
