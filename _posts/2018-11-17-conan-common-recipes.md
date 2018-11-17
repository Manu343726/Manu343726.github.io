---
title: Writing reusable base recipes for conan.io packages
date: 2018-11-17
tags: [c++,conanio]
comments: true
---

One of my problems when doing C++ is that I actually like C++ (Yep, that's
a problem), so I end up spawning new projects like every two monthsh or
something. *Which is almost nothing compared to some people in this community*.

These days writing a C++ library, either be it a header only library or
something that takes two hours to build, involves not only sharing your library
on github, twitter, etc but also write a [conan.io](https://conan.io/) recipe
for it; so that other can resuse your library like this was Javascript.

A dependency manager for C++! What a time to be alive!


But, as with everything in the software engineering industry, using conan means
maintaining more code. Writing, testing, and versioning recipes. This is
specially frustrating since conan approach to packaging is so simple (Get your
sources, configure them, build, package the results) that **you end up writing
almost the same recipes for all your packages**. Like, clone a repo from github,
configure and build it with cmake, tell conan what binaries and headers to keep:

``` python
import conans, os
from conans import ConanFile, CMake, tools

class CppastConan(ConanFile):
    name = "cppast"
    version = "master"
    license = "MIT"
    url = "https://gitlab.com/Manu343726/foonan"
    description = "Library to parse and work with the C++ AST"
    settings = "os", "compiler", "build_type", "arch"
    options = {"shared": [True, False]}
    default_options = "shared=False"
    generators = "cmake_find_package","cmake"
    requires = ("clang/6.0.0@Manu343726/testing",
                "cxxopts/2.1.0@Manu343726/stable",
                "type_safe/0.3@Manu343726/testing",)

    def source(self):
        git = tools.Git(folter='cppast')
        git.clone("https://github.com/foonathan/cppast.git", self.version)

    def build(self):
        cmake = CMake(self)
        cmake.configure(defs={
            "CMAKE_VERBOSE_MAKEFILE": True,
            "CPPAST_BUILD_TEST": False,
            "CPPAST_BUILD_EXAMPLE": False,
            "CPPAST_BUILD_TOOL": False
            },  source_dir='cppast')
        cmake.build()

    def package(self):
        self.copy("*.h", dst="include", src=os.path.join("cppast", "include"))
        self.copy("*.hpp", dst="include", src=os.path.join("cppast", "include"))
        self.copy("*.lib", dst="lib", keep_path=False)
        self.copy("*.dll", dst="bin", keep_path=False)
        self.copy("*.so", dst="lib", keep_path=False)
        self.copy("*.dylib", dst="lib", keep_path=False)
        self.copy("*.a", dst="lib", keep_path=False)

    def package_info(self):
        self.cpp_info.libs = ["cppast", "_cppast_tiny_process"]

        if self.settings.os == "Linux":
            self.cpp_info.libs.append("pthread")
```

Since past year conan supports packaging python modules, literally using conan
as a python package manager, which could be used to write some hacks and reuse
python code for our recipes. But since python packages are just normal conan
packages, the dependencies are resolved **after the recipe itself is parsed**,
which limits the ways you can import your module utilities into the recipe.


But don't worry, not all is lost. With the addition of [python
requires](https://docs.conan.io/en/latest/extending/python_requires.html#python-requires)
the conan teamgave us the exact piece missing from this puzzle: A way to import
python dependencies at recipe parsing time!

# Using python requires

As the conan docs tell us, common recipe dependencies can be imported by calling
the `python_requires()` function **instead of writing the dependency as a C++ or
build dependency of our package**:

``` cpp
from conans import python_requires

common = python_requires('conan_common_recipes/0.0.1@Manu343726/testing')

class OptionalLite(common.HeaderOnlyPackage):
    name = "optional-lite"
    url = "https://gitlab.com/Manu343726/conan-nonstdlite-packages"
    description = "A single-file header-only version of a C++17-like optional, a nullable object for C++98, C++11 and later"
    version = "3.1.1"
    scm = {
        "type": "git",
        "subfolder": "optional-lite",
        "url": "https://github.com/martinmoene/optional-lite",
        "revision": "v" + version
    }
```

As you can see, this recipe pulls my package of common recipes as a recipe
dependency with `python_requires()`. After the `python_requires()` call, **every
entity (class, function, enum, etc) exported by the module is imported into the
recipe**, as if we were doing just a python `import` statement. In the example,
we inherit from a base conan recipe class that implements all the common stuff
for header only libraries: Packaging the includes in the right paths, etc.

Following this approach, package recipes become just **a declarative set of
package properties**: Name, version, sources, etc. **Nothing more**.

# Writing a python package with common recipes

To write your own package for sharing common recipes, first just write your
python modules as usual, like we were not doing conan at all:

```
common/
  headeronly.py
  cmake.py
  autotools.py
  __main__.py
```

Now, write a conan recipe that **exports any python sources**, so that any
python module you write will be part of the conan package:

``` python
from conans import ConanFile

class ConanCommonRecipes(ConanFile):
    name = "conan_common_recipes"
    version = "0.0.1"
    url = "https://gitlab.com/Manu343726/conan-common-recipes"
    license = "MIT"
    description = "Common recipes for conan.io packages"
    exports = "*.py"
```

The above recipe will package all your python modules, but still the modules
**will not be available to the dependant recipes** (Like `OptionalLite` above).
The trick here is to tell python to **import the content of those modules into
the recipe module**, so when conan `python_requires()` the package, the recipe
module of that package (the conanfile.py with `ConanCommonRecipes` in the
example) **and everything that it imports** will be imported into
your package recipe:

``` python
from common.headeronly import *
from common.cmake import *
from common.autotools import *

class ConanCommonRecipes(ConanFile):
   ...
```

To recap a bit, the final layout of the common recipes package sources will be:

```
common/ # the python modules
  headeronly.py
  cmake.py
  autotools.py
  __main__.py
conanfile.py # the conan package recipe
```

and the final recipe for the common recipes package is:

``` python
from conans import ConanFile
from common.headeronly import *
from common.cmake import *
from common.autotools import *

class ConanCommonRecipes(ConanFile):
    name = "conan_common_recipes"
    version = "0.0.1"
    url = "https://gitlab.com/Manu343726/conan-common-recipes"
    license = "MIT"
    description = "Common recipes for conan.io packages"
    exports = "*.py"
```

# Writing a common recipe class

How a common recipe class looks like? **Like every other conan recipe class**.
Reusing the recipe is just simple python class inheritance, so all recipe
methods (`source()`, `build()`, etc) are available in your derived recipe class.

This is the implementation of `HeaderOnlyPackage` common recipe:

``` python
from conans import ConanFile

class HeaderOnlyPackage(ConanFile):
    @property
    def _package_uses_git_scm(self):
        return hasattr(self, 'scm') and self.scm['type'] == 'git'

    def package(self):
        import os

        if not self._package_uses_git_scm:
            raise Exception("Recipe for package \"{}\" is not using scm with type git for its sources".format(self.name))

        include_dir = os.path.join(self.scm['subfolder'], 'include')

        if not os.path.exists(include_dir):
            raise Exception("Source repository of package \"{}\" has no include/ directory. Expected: {}".format(self.name, include_dir))

        for ext in ['*.h', '*.hpp', '*.hxx', '*.hcc']:
            self.copy(ext, dst='include', src=include_dir)

    def package_id(self):
        self.info.header_only()
```

It just implements `package()` which copies the headers from the `include/` dir
of the sources, and `package_info()` to tell conan that our package is header
only.

# Write your own!

Now go, fire up your laptop, and start writing common recipes with your conan
packaging patterns!
