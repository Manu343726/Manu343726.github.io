---
title: Writing AST matchers for libclang
date: 2017-02-11T15:46:14+01:00
---

## Preface

It has been like one year or so since the last time I wrote a blog post.
I used to habve that common habit of writing posts about the latest
*supposedly* cool thing I've been working on. But this has been a weird
year which I've spent leaving my CS degree (*Why is a matter for another
post...*), working hard both at work, and at home with my
latest-but-yet-wip project: siplasplas

Siplasplas started as a C++ course I gave to some people of a game
programming masters degree in Madrid. The idea of the course was to
introduce students to the inner deep concepts of C++ required to become
a C++ expert. Since students were coming with the idea to become game
programmers, I focused topics and examples on bare metal concepts related
to game programming.  We spent almost three months learning the
intricacies of writing basic specialized allocators, such as stacks,
pools, etc; then learning how to use them with standard containers *and
why the standard allocator model is so flawed in the process*. I've to say
that I followed the great blog.molecular-matters.com to prepare the
content for that classes, and I'm very grateful for having such
a reference. *I also planned to write a task engine following molecular
tutorial, but we finally had no time enough*.

My metodology to prepare the classes was to write my own implementation of
a topic two or three months ahead of the course, so while I was teaching
allocators I started working on the implementation for the next topic:
**What if we write a C++ reflection engine from scratch?**

My goals were clear: These students are used to hear about Unreal Engine
and its "blueprints", but they usually have no idea how these things work
and what what they are really doing when writing "C++" with Unreal. By
"C++" I mean Standard C++ full of magic macros, he usual issue is that the
average game programming student doesn't know where C++, the standard
language stuff, ends and where the tricky Unreal features begin. It's
a blurred line for them.  So the idea was to show them that such features
are not magic, but a system (very common in their industry) that anyone
can undestand and implement.


The implementation of the reflection engine started with the runtime
representation of the entities we wanted to query: **Classes, public
member functions, and public member variables**. Then a kind of *runtime*
that indexes that entities.  Manually registering sourcecode entities is
a boring, error prone, and repetitive task, so the next step was to write
a tool that could parse our C++ code and register the entities for us:
What I called the "*Dynamic Reflection Library Parser"*, or
[DRLParser](https://github.com/biicode/common/blob/develop/edition/parsing/cpp/drl_parser.py)
for friends. DRLParser was a "simple" python script that used libclang
bindings to parse C++ sourcecode and generate C++ code with all the
metadata. Then reflection system could be feed with all that metadata and
store it in a dynamic reflection runtime (The index above).

Months, classes, and weird C++ code followed; then *someone* suggested
this story would be a good topic for a meetingcpp talk. The months that
followed the talk proposal were a race to document the library, write some
tests, and useful examples so people would not think I have beein wasting
my time for a year...  

During the talk I detailed some points (and issues) of the current
siplasplas implementation:

 - **Only supports OOP entities**
 - **Relies on an external python script**
 - **Dependencies were a hell to maintain**
 - **CI builds are (and still are...) broken**

### Improving the parser

I already solved the third issue by moving all my dependencies to
[conan.io](https://www.conan.io), but the first two were inherent to the
design of the parser itself. So after meetingcpp I started woring on a C++
replacement for DRLParser. The new reflection parser is organised as
follows:

 - **A libclang C++ wrapper**: Maps almost 1 to 1 libclang features, using
   RAII to manage libclang resources. The wrapper is extended on demand,
   I will only add new features as I need them.

 - **A core parsing API**: Uses the libclang wrapper to build the basics
   of the parsing engine, such as collecting all required entities from
   a translation unit, checking `#include` dependencies of headers,
   tracking changes of translation units (So you only reparse if
   required), etc. **I'm currently working on this layer**.

 - **Entity processing and metadata generation**: This layer will take the
   processed AST information from the layers above and process it,
   generating the metadata we need for reflection.

 - **Code generation**: Takes the metadata and generates a readable
   representation as C++ code.

All the layers above will be provided as an independent API, and a client
(The new DRLParser) will be written on top of the API. *I mostly based the
new design on the feedback from Jonathan Muller and his [Standardese
project](https://github.com/foonathan/standardese)*.

## Let's go!

Such a long preface, isn't? *As I said, it was a long time since I wrote
my last post. So much to say!"*

This post will be focused on the core parsing API above, specifically, how
we can write AST matchers using the libclang API

## libclang AST

The libclang AST abstracts the C++ recursive AST visitor of Clang into
a C cursor-based API. Basically a cursor is a generic representation of an
AST node. libclang works by giving functions to recurse the AST,
installing callbacks on node visitation, and a lot of functions to ask
different properties of the entity a cursor points to.

``` cpp
class MyAstVisitor : public VisitorInterface::Make<RecursiveAstVisitor>
{
public:
    void onCursor(const Cursor& current, const Cursor& parent) override
    {
        if(cursor.kind() == CursorKind::Kind::ClassDecl)
        {
            std::cout << "Class found: " << current.spelling();
        }
    }
};

Index index;
TranslationUnit tu = index.parse(
    "helloworld.cpp",
    CompileOptions().std("c++14")
);
MyAstVisitor visitor;
visitor.visit(tu.cursor());
```

*The above example uses my libclang wrapper, not the libclang C API*

This works great for simple cases, but once you start trying to find
complex structures in your C++ code, walkind the AST directly with
visitors becomes more and more complex. To solve this issue, the Clang
C++ API provides [the Matchers
API](http://clang.llvm.org/docs/LibASTMatchers.html), a simple C++ DSL to
create predicates on the AST. You can register multiple *matchers* in
a *match finder*, and Clang will automatically notify you if some of these
predicates matched:

``` cpp
class MyMatchCallback : public MatchCallback
{
public:
    void onMatch(const MatchResult& result) override
    {
        std::cout << "Found a class named "
            << result.boundNodes.get("class").spelling()
            << " with " result.boundNodes.getAll("methods").size()
            << " methods\n";
    }
};

TranslationUnit tu = ...;
MyMatchCallback callback;
MatchFinder finder;
finder.addMatch(
    id("class", classDecl(
        id("methods", has(cxxMethod(unless(isStatic()))))
    )),

    callback
);
```

We no longer have callbacks for each AST node, **but a callback that is
called only if an AST node matches a set of requirements**. In the
example, class declarations with at least one non-static member function.
The DSL also allows tagging parts of the predicate, so its easy to get
what nodes were involved in each part of the matched predicate.

While matching an structure like in the example above is a very complex
task using visitors only, ASTMatchers API is not exposed by libclang. So
that's it. You have a great tool to solve your problem but it is not in
your stable-between-versions toolbox.


If I were another person I would close vim, start crying, and blame
libclang for being such a limited API. But parsing C++ code for reflection
is all about searching specific code structures, and doing it by hand does
not scale at all (My Python script only supported basic OOP stuff and it
already started to look terrible). So I absolutely needed something like
ASTMatchers.


## Writing ASTMatchers ourselves

LLVM and Clang are both two great open source projects, and what that
means is that if you are bored enough you can end up studying their code.

*Yes, that's my definition of "Open Source Sofware"*

For me this was easy since I already had a local copy of clang sources in
the conan local cache. If you look at carefully at the sources,
ASTMatchers API is just a couple of headers:

 - *`ASTMatchers.h`*: Exposes the user API and matcher functions (Those
   `classDecl()`, `cxxMethod()`, etc above) that form the DSL.
 
