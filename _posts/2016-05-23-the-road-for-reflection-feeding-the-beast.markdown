---
title: The road for reflection: Feeding the beast
date: 2016-05-23T18:37:41+02:00
---

Once I had a conversation with a professor of my university about what he
called the "Pointer Abstraction" *or something like that*, regarding that
despite cache related issues in modern systems the **pointer** is a useful
abstraction that helps us reason about algorithms (Swap-based sorting
algorithms come to my mind), data structures (trees, linked lists,...),
etc. *WTF are you talking about? Doing a bit of retrospective that
conversation now sounds like a pair of drunk philosophers talking about
the metaverse. I should consider better narrative constructions to get
into posts...*

When writing a dynamic reflection engine, it's all about pointers.
Pointers to member functions, pointers to class fields, everything tied to
a string so we could say "Look, I don't need Python anymore, good old C++
can get class members by strings too!".  
The same principle applies to any C++ to *X* bindings library out there,
these APIs are usually pointer-centric:

``` cpp
#include <pybind11/pybind11.h>

int add(int i, int j) {
    return i + j;
}

namespace py = pybind11;

PYBIND11_PLUGIN(example) {
    py::module m("example", "pybind11 example plugin");

    m.def("add", &add, "A function which adds two numbers");

   return m.ptr(); } ```

*A [pybind11](https://github.com/pybind/pybind11) binding example from
[official docs](http://pybind11.readthedocs.io/en/latest/basics.html).*

But the fact that a *"dynamic function"*
(`cpp::dynamic_reflection::Function` in my case) is just a glorified
`std::pair<std::string, R(*)(Args...)>` doesn't mean one should pollute
the API with pointers. Frankly, I have no specific problem with pointers
in general, but I would like to avoid the horrible pointer to member
syntax if possible.

Even [early
versions](https://github.com/GueimUCM/siplasplas/blob/7e3c125927275d66b9a1ec7dabed5059b5e1ef33/examples/reflection.cpp)
of siplasplas engine were based on a raw-pointer registration API, what's
worse, `MACRO_BASED()`. *As I said, I hate member pointer syntax, back
then wrapping the thing in a macro to get/hide the pointer and the string
looked like a good idea...*

Of course you cannot hide the fact that everything the reflection engine,
binding library, whatever, are doing is to map names to pointers.
***Cannot you?***

## Feeding as an implementation detail

Ok, so everything you do is to map pointers. *End of post. Thanks for
coming!*.

Despite the engine model is a set of abstractions on top of pairs `(name,
pointer to thing)`, I would like to ignore that fact and focus on
**entities** instead: Imagine for a moment that C++ has builtin support
for reflection (I hope not [on this
way](https://www.google.es/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&ved=0ahUKEwiNm76k2vDMAhXJERQKHWnlDO4QFggxMAI&url=http%3A%2F%2Fwww.open-std.org%2Fjtc1%2Fsc22%2Fwg21%2Fdocs%2Fpapers%2F2016%2Fp0194r0.pdf&usg=AFQjCNHQtS-Ue65kttbKjAKNwhQp-2Fz9g&sig2=a2f0-OzQ99yXryeu1KajzQ)...).
If that was the case I would expect to have access to **class metadata**,
with information such as its list of members:

``` cpp
class MyClass
{
public:
    int foo()
    {
        return 42;
    }
};

MyClass myObject;
constexpr auto method = meta<MyClass>().method("foo");

// I've never trusted Deep Thought, but...
int result = method(myObject);
```

That is, I don't bother whether the underlying data uses pointers or
whatever else, **I just asked for a method and the engine returned an
entity representing it**.

For binding APIs a *low level* registration API is fine since the set of
types to register is usually short. But for reflection it's exactly the
opposite, you expect to have the right to do type introspection on
(almost) any type your project declares, metadata registration becoming
a painful task.

This is the reason why I think the process of *feeding* the engine with
metadata should be transparent to the user. And since the C++ language
currently lacks introspection, "transparent" means writing an external
preprocessing tool.

## The tool

For the siplasplas reflection engine I wrote a tool I call **DRLParser**
(*"Dynamic Reflection Library Parser"* or *"Diego Rodriguez Losada
Parser"* for friends), a python command line tool that parses your
sourcetree and generates C++ headers with metadata encoded as templates
(More on this later).

The tool works roughly as follows:

 1. **Collect sources information**: Given a set of search directories,
    DRLParser searches source files to process. It supports globs,
    filters, directory blacklists, etc. The idea is that the tool
    is designed to parse all your codebase and maintain upated metadata.
    For each file parsed it generates one header in a `[buildtree root]/path/to/your/file/` 
    directory in your buildtree.

 2. **Collect metadata**: For each file, it parses the file and collects
    metadata of **types and functions declared in that file**. What this
    stage generates is a processed version of the file's AST.

 3. **Generate C++ code**: Given the processed AST generated in the
    previous stage, the tool uses a set of [jinja2]() templates to write
    the collected metadata as C++ templates in a generated header file.

Let's ilustrate the process with an example: Imagine you're developing
a library `libfoo` that relies on reflection. The `libfoo` project looks
like this:

```
+--libfoo
|  +--include/libfoo/
|  |  foo.h
|  |
|  +--src/
|  |  foo.cpp
|  |  CMakeLists.txt
|  |
|  +--build/
|     ...
```

To get reflection metadata for `libfoo`, you run DRLParser as follows:

``` bash
$ cd libfoo
$ python2 DRLParser --source-dir . --searchdirs include/ --extensions *.h --compile-options "-std=c++11"
-o build/output/reflection/
```

which tells DRLParser to search for `*.h` files at `include/` directory,
parse them using C++11, and putting output at `build/` directory. *In real
world code you may need more parameters. For example, the template file
used for code generation is not hardcoded but passed as parameter (Which
could mean DRLParser could be used for other purposes further
reflection...). The interface may be complex, but note this is not
designed to be invoked by humans but by build systems. I actually use
DRLParser through a CMake script that invokes DRLParser inferring settings
from a given library target (Include directories, compile options,
etc).*

### Parsing sources

Hopefully we live in an age where writing your custom C++ "parser",
cappable of reading only a subset of C With Classes (MOC, anybody?), is no
longer needed. Thanks to the LLVM project we have APIs that help with
every compilation related tasks, from AST processing to code generation.

For DRLParser I picked the [libclang Python bindings](). Why libclang?

 - **It understands C++**: libclang is not a hand rolled C++-like parser
   but an interface to the internals of a true C++ compiler. Whatever
   Clang can parse, libclang parses it to. *Almost, more on this bellow.*

 - **It's stable**: libclang is designed as a C interface to the Clang C++
   APIs, aimed to provide API and ABI stable access to AST processing. It
   may not be as powerful as Clang C++ APIs such as [libtooling](), but it
   gives you the guarantee that you won't have to rewrite your parser
   whenever Clang gets a new release.

 - **It's Python**: Yes, I'm using the Python bindings instead of the
   C API directly. My good friend
   [Joonathan](https://github.com/foonathan/standardese) may not agree
   with me, but there are situations where C++ is not The Right Tool, and
   other languages with fast prototyping, simple and powerful string
   handling, and cross platform dependency management may do the work
   better. *Of course this is just my opinion, Jonathan is already doing
   a great work with Standardese. I often find interesting to compare his
   solutions with my own.*

The libclang "Hello, World" in Python looks more or less like this:

``` python
from clang.cindex import Index

index = Index.create()
translation_unit = index.parse('myheader.h', args = ['-std=c++11'])

for c in translation_unit.get_children():
    print c.spelling
```

First we get an `Index` instance. An `Index` represents a set of
translation units that may become a final executable or library after
linking. We ask `index` to parse a source file, giving us the
`translation_unit` of that parsed file.

libclang works by giving acess to AST entities through `Cursor` class,
which represents an specific entity (i.e. node) of the AST. *Here Python
bindings ease things a lot, since `Cursor` (`CXCursor` in the original
C API) hides all the AST visitation stuff (Technically a CXCursor` is not
a node but a kind of visitor on the underlying AST data structure).*

So in order to transverse the full syntax tree you've to do the usual
recursion using `Cursor.get_children()` to get the list of children
cursors of a given cursor. Consider an example C++ code, `foo.h`:

``` cpp
#ifndef LIBFOO_FOO_H
#define LIBFOO_FOO_H

class Foo
{
public:
    void f(int i);
};

Foo make_foo();

#endif // LIBFOO_FOO_H
```

If you parse this file, the generated AST will be similar to this:

```
+-- spelling = 'libfoo/include/libfoo/foo.h', kind = CursorKind.TRANSLATION_UNIT
|   +-- spelling = '', kind = CursorKind.NAMESPACE
|   |   +-- spelling = 'Foo', kind = CursorKind.CLASS_DECL
|   |       +-- spelling = 'f', kind = CursorKind.CXX_METHOD
|   |           +-- spelling = 'i', kind = CursorKind.PARAM_DECL
|   +-- spelling = 'make_foo', kind = CursorKind.FUNCTION_DECL
```

`Cursor` class has properties and functions to ask for whatever property
an entity may have (Is a static function?, is const?, get the aliased type
if the cursor is a typedef, etc). The most important are the two I'm using
above:

 - **spelling**: Gives the name the entity is declared with in the C++
   code. Note that spellings are not unique, there could be multiple
   cursors with same spelling, such as those pointing to overloaded
   functions. `Cursor` also exposes the `displayname` property which gives
   the full name of the enity ("f(int)").

 - **kind**: Value of `CursorKind` enumeration stating what kind of entity
   the cursor points to: A class declaration, a template type parameter,
   a function definition, a const qualifier, an integer literal, etc.
   `CursorKind` enumeration has one value for each kind of entity
   supported by libclang API. *libclang is not designed to do full AST
   processing but to help building applications in the way of refactoring
   tools and completion systems. Since in that cases not all information
   is needed, libclang AST doesn't export all AST information, tagging
   declarations of unsupported entities as `CursorKind.UNEXPOSED_DECL`.

If you look carefully at the graph above, you may notice the root cursor
of the AST represents the translation unit itself. Then comes the global
namespace, with `Foo` class and `make_foo()` function.
