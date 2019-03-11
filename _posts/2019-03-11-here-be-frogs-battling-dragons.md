---
title: Here be frogs battling dragons
date: 2019-03-11
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
of those, let's forward all the hard problems to
[conan](https://conan.io/).

!["Are you a frog?" by @lizardseraphim](/img/frog_vs_dragon.jpg)  
*"Are you a frog?" by [@lizardseraphim](https://www.deviantart.com/lizardseraphim)*

## Brief intro to conan recipes

Conan let's you do anything. It's not tied to any build system in particular but
gives you a *recipe*, a Python script where you tell how your project should be
configured, built, packaged, installed, etc. In this way is very similar to
package recipes of system package managers like Arch's `PKGBUILD` scripts or
RPM spec files:

``` python
from conans import ConanFile

class MyConanRecipe(ConanFile):
    name = 'MyPackage'
    description = 'My cool package'
    url = 'https://github/Manu343726/MyCoolProject'
    license = 'MIT'
    version = '0.1.2'

    # Define your package ABI compatibility
    settings = 'os', 'arch', 'compiler', 'build_type'

    def source(self): 
        # Download your sources here, or pick them from the source tree

    def build(self):
        # Configure and build your project using autotools, CMake, whatever

    def package(self):
        # Tell conan what artifacts to keep, what headers to package, etc
```

In addition conan has a lot of customization features for your recipes,
like package options, header only lib mode, overriding settings depending
on the target toolchain, etc. See the [conan
docs](https://docs.conan.io/en/latest/introduction.html) for details.

## Building LLVM

Building LLVM is as simple as cloning its sources and running CMake:

``` shell
$ git clone https://github.com/llvm-mirror/llvm/ --branch release_60
$ cd llvm
$ mkdir build && cd build
$ cmake ..
$ make -j$(nproc)
```

One to ten hours later (depending on your hardware) you will have
a compiled LLVM distribution ready to install in your system.  

When building LLVM with conan there's a couple of extra things to
careabout: 
 
 - [LLVMTestingSupport](https://github.com/llvm-mirror/llvm/blob/release_60/lib/Testing/Support/CMakeLists.txt): This module links against [Google Test](). I wanted to make sure all dependencies
are tracked by conan, so instead of using the gtest version hosted in LLVM's
sources, the LLVM conan package requires gtest as a dependency. This avoids
having to install gtest in the system in order to build LLVM.
 - [LLVMSupport](https://github.com/llvm-mirror/llvm/tree/release_60/lib/Support): For some obscure reason LLVM implemented terminal colored output
querying terminal capabilities through the `terminfo` library, instead of,
say, checking the `TERM` environment variable. This causes a lot of
headaches since `terminfo` is packaged in very different ways across
different linux variants: Sometimes it's `libtinfo`, sometimes it's part
of ncurses, etc.

Also, there are some self-explanatory CMake variables to control what
modules of LLVM are compiled:

 - `LLVM_INCLUDE_TESTS`
 - `LLVM_BUILD_TESTS`
 - `LLVM_INCLUDE_EXAMPLES`
 - `LLVM_BUILD_EXAMPLES`
 - `LLVM_INCLUDE_DOCS`
 - `LLVM_BUILD_TOOLS`

By default I set all of this variables to `FALSE` to reduce compile times.
All this steps could be summarized in the following conan recipe:

``` python
from conans import ConanFile, CMake
import conans.tools

class LLVM(ConanFile):
    name = 'LLVM'
    version = '6.0.1'
    ...

    def source(self):
        conan.tools.get('https://dl.bintray.com/manu343726/llvm-sources/llvm-6.0.1.tar.gz') 

    def build(self):
        cmake = CMake(self)
        cmake.configure(defs={
            'LLVM_INCLUDE_TESTS': False,
            'LLVM_BUILD_TESTS': False,
            'LLVM_INCLUDE_EXAMPLES': False,
            'LLVM_BUILD_EXAMPLES': False,
            'LLVM_INCLUDE_DOCS': False,
            'LLVM_BUILD_TOOLS': False,
        })
        cmake.build()
        cmake.install()

    def package(self):
        # Save headers into the package
        self.copy('*.hpp', src='install dir/include', dst='include/')
        # Save libraries into the package
        self.copy('*.so', src='install dir/lib', dst='lib/')
        self.copy('*.a', src='install dir/lib', dst='lib/')
        self.copy('*.dll', src='install dir/lib', dst='lib/')
        self.copy('*.lib', src='install dir/lib', dst='lib/')
        # Save CMake scripts
        self.copy('*.cmake', dst='lib/')
        # Save binary tools
        self.copy('*', src='install dir/bin', dst='bin/')

    def package_info(self):
        # Tell upstream users which libraries this package
        # contains
        self.cpp_info.libs = conan.tools.collect_libs(self)
```

*Note the above recipe is overly simplified, continue reading for the real
recipe*.

## Building other LLVM components

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
through
[`find_program()`](https://cmake.org/cmake/help/latest/command/find_program.html)
calls. This allows components such as Clang or libc++ **to find
llvm-config utility from LLVM**, which is compiled and packaged by the
LLVM build. LLVM components use `llvm-config` to figure out where LLVM was
installed (The LLVM conan package directory in this case) and get the
common CMake scripts from there.

> Note that conan recipes are Python scripts. You're free to add as much
> extra class methods, properties, etc to your recipe class as you may
> need. For example, the `self._root_cmake_file` there is a property of my
> LLVM recipes that tells me the location of the root CMakeLists.txt file
> of a project, defined more or less like this:
>
> ``` python
> 
> @property
> def _root_cmake_file(self):
>   return os.path.join(self.source_dir, 'CMakeLists.txt')
>
> ```


The cool thing about this approach is that I not only inject the path to
`llvm-config` but also **the full conan deps configuration**, which means
I have full control of the component dependencies. I can point any
external dependency to a conan package, making the build system dependency
free.

> Well, this is not 100% true in the current version of the packages.
> I had to leave a dependency on zlib and libtinfo for lack of time, but
> I plan to rebuild the packages with zero system deps asap.

After that point the build just the usual build of an LLVM component, as
if you were following the manual. Just `make -j$(nproc)` and go watch the
director's cut of The Lord Of The Rings.

## Conan recipes for each LLVM library

Having [one huge package for
LLVM](https://bintray.com/beta/#/manu343726/conan-packages/llvm:Manu343726?tab=overview)
and [another huge package for
Clang](https://bintray.com/beta/#/manu343726/conan-packages/clang:Manu343726?tab=overview)
is a big leap forward compared to having nothing in conan, but it comes
with some caveats. In order to use those packages you either have to use
the official LLVM cmake scripts ([Which are rather
obfuscated](https://github.com/snowzurfer/brto-llvm/blob/master/CMakeLists.txt#L88)
and lack dependency resolution), or do it the conan way, requesting the
component as a dependency and linking it in your project:

``` ini
# conanfile.txt

[requires]
clang/6.0.1@Manu343726/testing

[generators]
cmake
```

``` cmake
# CMakeLists.txt

project(MyCppParser)

include("${CMAKE_BINARY_DIR}/conanbuildinfo.cmake")
conan_basic_setup(TARGETS)

add_executable(mycppparser main.cpp)
target_link_libraries(mycppparser PRIVATE CONAN_PKG::clang)
```

However this last snippets may not be a solution to the problem, since
what that cmake script is doing is to link `mycppparser` **against every
LLVM, compiler-rt, and Clang library**. Remember that `collect_libs()`
call in `package_info()`?

``` python
def package_info(self):
    # Tell upstream users which libraries this package
    # contains
    self.cpp_info.libs = conan.tools.collect_libs(self)
```

That comment there is key: [`cpp_info.libs`](https://docs.conan.io/en/latest/reference/conanfile/attributes.html#cpp-info) is a list of the libraries
a package contains, that is, the way conan tells users of your package
what libraries user projects must link when consuming your package. So if
all LLVM components (LLVM, compiler-rt, Clang) are packaged the same way
(SPOILER: *They do*) you're actually telling your users to link against
every single LLVM, compiler-rt and Clang library.

If you're like me and you like doing full self-contained executable
distribution where all deps are linked statically, you could use Clang
libraries this way and cross your fingers whishing the linker will remove
every single piece of LLVM and Clang code you're not using.

I don't think that approach qualifies for ***Production Ready Software
Solution (TM)*** mark, so let's try another approach: What if we package
every single LLVM and Clang library as its own conan package?

## Dependency hell and why monolithic projects must die

So, the plan is the following:

1. For each component (LLVM, Compiler-rt, Clang) identify every library
   and its associated headers. Call each unit a ***LLVM module***.

2. Write a conan recipe for each module **to package the corresponing
   library file**. Libraries are copied from their parent component
   package, **no individual compilation of each lib is done**.

3. Figure out dependencies between the different modules, write them as
   `requires` between the different packages.

4. Try to reuse as much recipe code as possible.

Points 1. and 3. are the usual monkey manual work that requires patience
and tens of litres of coffee. I managed to get the work done in one of my
*straight-12-hours-not-moving-my-ass-from-the-computer* weekend
programming sessions by a combination of manually inspecting each library
`CMakeLists.txt` file and
[`lddtree`](https://codeyarns.com/2015/12/16/how-to-view-hierarchy-of-shared-library-dependencies-using-lddtree/).
For reference, this is the dependency tree of libclang 6.0.1:

```
libclang.so => path_to_clang_package/lib/libclang.so (interpreter => none)
    libclangAST.so.6 => path_to_clang_package/lib/../lib/libclangAST.so.6
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1
    libclangBasic.so.6 => path_to_clang_package/lib/../lib/libclangBasic.so.6
        libLLVMCore.so.6 => path_to_llvm_package/lib/libLLVMCore.so.6
            libLLVMBinaryFormat.so.6 => path_to_llvm_package/lib/../lib/libLLVMBinaryFormat.so.6
            ld-linux-x86-64.so.2 => /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
        libLLVMMC.so.6 => path_to_llvm_package/lib/libLLVMMC.so.6
    libclangFrontend.so.6 => path_to_clang_package/lib/../lib/libclangFrontend.so.6
        libclangDriver.so.6 => path_to_clang_package/lib/../lib/../lib/libclangDriver.so.6
        libclangEdit.so.6 => path_to_clang_package/lib/../lib/../lib/libclangEdit.so.6
        libclangParse.so.6 => path_to_clang_package/lib/../lib/../lib/libclangParse.so.6
            libLLVMMCParser.so.6 => path_to_llvm_package/lib/libLLVMMCParser.so.6
        libclangSerialization.so.6 => path_to_clang_package/lib/../lib/../lib/libclangSerialization.so.6
        libLLVMBitReader.so.6 => path_to_llvm_package/lib/libLLVMBitReader.so.6
        libLLVMOption.so.6 => path_to_llvm_package/lib/libLLVMOption.so.6
        libLLVMProfileData.so.6 => path_to_llvm_package/lib/libLLVMProfileData.so.6
    libclangIndex.so.6 => path_to_clang_package/lib/../lib/libclangIndex.so.6
        libclangFormat.so.6 => path_to_clang_package/lib/../lib/../lib/libclangFormat.so.6
        libclangToolingCore.so.6 => path_to_clang_package/lib/../lib/../lib/libclangToolingCore.so.6
            libclangRewrite.so.6 => path_to_clang_package/lib/../lib/../lib/../lib/libclangRewrite.so.6
    libclangLex.so.6 => path_to_clang_package/lib/../lib/libclangLex.so.6
    libclangSema.so.6 => path_to_clang_package/lib/../lib/libclangSema.so.6
        libclangAnalysis.so.6 => path_to_clang_package/lib/../lib/../lib/libclangAnalysis.so.6
    libclangTooling.so.6 => path_to_clang_package/lib/../lib/libclangTooling.so.6
        libclangASTMatchers.so.6 => path_to_clang_package/lib/../lib/../lib/libclangASTMatchers.so.6
    libclangARCMigrate.so.6 => path_to_clang_package/lib/../lib/libclangARCMigrate.so.6
        libclangStaticAnalyzerCheckers.so.6 => path_to_clang_package/lib/../lib/../lib/libclangStaticAnalyzerCheckers.so.6
            libclangStaticAnalyzerCore.so.6 => path_to_clang_package/lib/../lib/../lib/../lib/libclangStaticAnalyzerCore.so.6
    libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2
    libLLVMX86CodeGen.so.6 => path_to_llvm_package/lib/libLLVMX86CodeGen.so.6
        libLLVMAnalysis.so.6 => path_to_llvm_package/lib/../lib/libLLVMAnalysis.so.6
            libLLVMObject.so.6 => path_to_llvm_package/lib/../lib/../lib/libLLVMObject.so.6
            libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6
        libLLVMAsmPrinter.so.6 => path_to_llvm_package/lib/../lib/libLLVMAsmPrinter.so.6
            libLLVMDebugInfoCodeView.so.6 => path_to_llvm_package/lib/../lib/../lib/libLLVMDebugInfoCodeView.so.6
        libLLVMCodeGen.so.6 => path_to_llvm_package/lib/../lib/libLLVMCodeGen.so.6
            libLLVMBitWriter.so.6 => path_to_llvm_package/lib/../lib/../lib/libLLVMBitWriter.so.6
            libLLVMScalarOpts.so.6 => path_to_llvm_package/lib/../lib/../lib/libLLVMScalarOpts.so.6
                libLLVMInstCombine.so.6 => path_to_llvm_package/lib/../lib/../lib/../lib/libLLVMInstCombine.so.6
            libLLVMTransformUtils.so.6 => path_to_llvm_package/lib/../lib/../lib/libLLVMTransformUtils.so.6
        libLLVMGlobalISel.so.6 => path_to_llvm_package/lib/../lib/libLLVMGlobalISel.so.6
        libLLVMSelectionDAG.so.6 => path_to_llvm_package/lib/../lib/libLLVMSelectionDAG.so.6
        libLLVMTarget.so.6 => path_to_llvm_package/lib/../lib/libLLVMTarget.so.6
        libLLVMX86AsmPrinter.so.6 => path_to_llvm_package/lib/../lib/libLLVMX86AsmPrinter.so.6
        libLLVMX86Utils.so.6 => path_to_llvm_package/lib/../lib/libLLVMX86Utils.so.6
    libLLVMX86AsmParser.so.6 => path_to_llvm_package/lib/libLLVMX86AsmParser.so.6
    libLLVMX86Desc.so.6 => path_to_llvm_package/lib/libLLVMX86Desc.so.6
        libLLVMMCDisassembler.so.6 => path_to_llvm_package/lib/../lib/libLLVMMCDisassembler.so.6
    libLLVMX86Info.so.6 => path_to_llvm_package/lib/libLLVMX86Info.so.6
    libLLVMSupport.so.6 => path_to_llvm_package/lib/libLLVMSupport.so.6
        libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1
        libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0
    libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
```

*And that's with duplicated dependencies removed, the full tree is much
more fun.*

Points 2, and 4. are solved through one of the latest conan features,
[`python_requires()`](https://docs.conan.io/en/latest/extending/python_requires.html#python-requires) which allows packaging python modules and conan
recipes (base recipes, think of them as recipe templates) and reuse them
to write another recipes.

With `python_requires()`, the component and module recipes are built using
the following base recipes:

 - [`LLVMPackage`](https://gitlab.com/Manu343726/clang-conan-packages/blob/master/llvm-common/llvmpackage.py): Base recipe with basic common package properties such as
 package naming conventions, LLVM version, option configuration, etc.
 - [`LLVMComponentPackage`](https://gitlab.com/Manu343726/clang-conan-packages/blob/master/llvm-common/llvmcomponentpackage.py): Base recipe to build LLVM components from their
 source trees (LLVM, Clang, Compiler-RT). This recipe implements all the
 steps shown in "Building LLVM" at the beginning of the post.
 - [`LLVMModulePackage`](https://gitlab.com/Manu343726/clang-conan-packages/blob/master/llvm-common/llvmmodulepackage.py): This base recipe implements all the machinery to
 **copy a module library file to its own package**, require the parent
 component as a build dependency, and require all direct module
 dependencies through the corresponding component packages.

> See my previous blog post ["Writing reusable conan recipes"](https://manu343726.github.io/2018-11-17-conan-common-recipes/)
> for an in depth tutorial on base recipes.

This base recipes are written so that all packages follow some rules:

 - Each module is packaged from a base component.
 - Each module package contains **one** module library only.
 - Module recipes specify both component and module dependencies **by name** only.
   Dependencies always point to packages of the same author, channel, and
   version as the dependant package.

Let's take a look to the clang AST matchers package:

``` python
from conans import python_requires

common = python_requires('llvm-common/0.0.0@Manu343726/testing')

class ClangASTMatchers(common.LLVMModulePackage):
    version = common.LLVMModulePackage.version
    name = 'clang_ast_matchers'
    llvm_component = 'clang'
    llvm_module = 'ASTMatchers'
    llvm_requires = ['clang_headers', 'clang_ast', 'clang_basic', 'llvm_support']
```

Ignoring the `version` property, [which looks a bit redundant to
me](https://github.com/conan-io/conan/issues/4214), the recipe is as
concise as possible. With just five lines we see:

 - That there's a clang library named [`ASTMatchers`](https://github.com/llvm-mirror/clang/tree/master/lib/ASTMatchers) that we want to
 package as [`clang_ast_matchers`](https://bintray.com/beta/#/manu343726/conan-packages/clang_ast_matchers:Manu343726?tab=overview) package.
 - That AST matchers library depends on clangAST, clangBasic, and
 LLVMSupport libraries (See the full library dependency tree
 [here](https://gitlab.com/Manu343726/clang-conan-packages/blob/master/clang_6.0.1_lddtree.txt#L597)).

> Note that each component headers are packaged as a single header only
> conan package named `<component>_headers`. `LLVMModulePackage` contains
> `package()` logic a bit more complex than what's shown in this post to
> take care of corner cases like this.

Add [an horrendous CI
pipeline](https://gitlab.com/Manu343726/clang-conan-packages/pipelines/47557929)
to the mix and that's it, one conan package for each Clang and LLVM
libraries.

## Great, but show me an example please

Here you go: This is a [real cppast
hello-world example](https://gitlab.com/Manu343726/foonan/tree/master/cppast) using conan to get the
cppast library and its libclang dependency:

``` cmake
# CMakeLists.txt

cmake_minimum_required(VERSION 2.8.1)
project(cppast-example)

include(${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup(TARGETS)

add_executable(ast_printer ast_printer.cpp)
target_link_libraries(ast_printer PRIVATE CONAN_PKG::cppast)

file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/example.hpp" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
```

``` cpp
// Copyright (C) 2017-2018 Jonathan Müller <jonathanmueller.dev@gmail.com>
// This file is subject to the license terms in the LICENSE file
// found in the top-level directory of this distribution.

/// \file
/// This is a very primitive version of the cppast tool.
///
/// Given an input file it will print the AST.

#include <cppast/visitor.hpp> // visit()

#include "example_parser.hpp"

void print_ast(const cppast::cpp_file& file)
{
    std::string prefix;
    // visit each entity in the file
    cppast::visit(file, [&](const cppast::cpp_entity& e, cppast::visitor_info info) {
        if (info.event == cppast::visitor_info::container_entity_exit) // exiting an old container
            prefix.pop_back();
        else if (info.event == cppast::visitor_info::container_entity_enter)
        // entering a new container
        {
            std::cout << prefix << "'" << e.name() << "' - " << cppast::to_string(e.kind()) << '\n';
            prefix += "\t";
        }
        else // if (info.event == cppast::visitor_info::leaf_entity) // a non-container entity
            std::cout << prefix << "'" << e.name() << "' - " << cppast::to_string(e.kind()) << '\n';
    });
}

int main(int argc, char* argv[])
{
    return example_main(argc, argv, {}, &print_ast);
}
```

## What's next?

Well, I would say the first next step is to add windows workers to the CI
pipeline. I already did some earlier tests of this recipes on windows and
it seems to work ok, so I will push the work once I begin working on MSVC
support for tinyrefl (See tinyrefl roadmap
[here](https://github.com/Manu343726/tinyrefl/releases/tag/v0.3.3)).  
Oh, also packaging libtinfo with conan to get rid of that system
dependency. *Any volunteer? Please?*

Apart from getting rid of the thousand of CMake lines tinyrefl contained
to handle its dependencies, I learned a lot about conan packaging while
working on this, and now I'm eager to try a new challenge. Maybe modular
Qt? Modular Boost?
