---
title: Writing AST matchers for libclang
date: 2017-02-11T15:46:14+01:00
tags: [c++,clang-api]
---

## Preface

It has been like one year or so since the last time I wrote a blog post.
I used to habve that common habit of writing posts about the latest
*supposedly* cool thing I've been working on. But this has been a weird
year which I've spent leaving my CS degree (*Why is a matter for another
post...*), working hard both at work, and at home with my
latest-but-yet-wip project: [siplasplas](https://github.com/Manu343726/siplasplas)

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
"C++" I mean Standard C++ full of magic macros, the usual issue is that the
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

finder.matchAst(tu);
```

We no longer have callbacks for each AST node, **but a callback that is
called only if an AST node matches a set of requirements**. In the
example, class declarations with at least one non-static member function.
The DSL also allows tagging parts of the predicate, so its easy to get
what nodes were involved in each part of the matched predicate.

Matching an structure like the example above is a very complex task using
visitors only, but ASTMatchers API is not exposed by libclang. So that's
it. You have a great tool to solve your problem but it is not in your
stable-between-versions toolbox.


If I were another person I would close vim, start crying, and blame
libclang for being such a limited API. But parsing C++ code for reflection
purposes is all about searching specific code structures, and doing it by
hand does not scale at all (My Python script only supported basic OOP
stuff and it already started to look terrible...). So I absolutely needed
something like ASTMatchers.


## Writing ASTMatchers ourselves

LLVM and Clang are both two great open source projects, and what that
means is that if you are bored enough you can end up studying their code.

*Yes, that's my definition of "Open Source Sofware"*

For me this was easy since I already had a local copy of clang sources in
the conan local cache. If you look carefully at the sources,
ASTMatchers API is just a bunch of headers:

 - **`ASTMatchers.h`**: Exposes the user API and matcher functions (Those
   `classDecl()`, `cxxMethod()`, etc above) that form the DSL.
 - **`ASTMatchersInternal.h`**: Internal types used by the implementation
   (The matcher interface, a bound nodes map, etc).
 - **`ASTMatchersMacros.h`**: Defines macros used to define matcher functions
 - **`ASTMatchFinder.h`**: Declaration of the `MatchFinder` class and its operations

*There's also the `Dynamic/` directory, where parsing routines are
declared so matchers can be defined at runtime from string descriptions.
This is what `clang-query` uses under the hood.*

First thing we note is that the Clang AST is typed, that is, there are
different node types for different categories of entities (Declarations,
statements, etc). Most of the complexity of ASTMatchers implementation
comes from handling this different types of nodes the right way. But since
we are using the Cursor abstraction from libclang, we have no such problem
and **our implementation can be simpler**.


### The matcher interface

First we must define what a matcher is. As we've seen in the overview
above, a matcher is basically a predicate on the AST. With this in mind we
can define a generic representation of a matcher as follows:

``` cpp
class MatcherInterface
{
public:
    virtual ~MatcherInterface() = default;

    virtual bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) = 0;
};
```

`MatcherInterface::matches()` basically states if a match is found on the
cursor currently being visited. For example:

``` cpp
class ClassDeclMatcher : public MatcherInterface
{
public:
    bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) override final
    {
        return current.kind() == CursorKind::Kind::ClassDecl;
    }
};

ClassDeclMatcher classDecl()
{
    return {};
}
```

The class above represents a class declaration matcher, which gives
a match on any cursor pointing to a class declaration.

The other two arguments, being unused by the class declaration matcher,
are tools for more complex matchers:

 - **`boundCursors`**: Allows registering the current cursor in a `name ->
   cursor` map. Used by matchers that allow binding names to its elements;
   such as `id(name, matcher)` in the first example.

 - **`AstMatchFinder`**: Provides an AST interface for the matcher, with
   operations to find more matches in other AST nodes. Used by complex
   matchers that need to check properties in siblings, descendants, or
   ancestors of the current node.


### A type erased matcher

In many situations we need to store multiple matchers in a common storage,
the simplest case `MatchFinder::addMatch()` function which internally
stores a matcher and its callback.

To do so, we define a `Matcher` class that does type-erasure of the given
`MatcherInterface` implementation:

``` cpp
class Matcher : public MatcherInterface
{
public:
    Matcher(std::shared_ptr<MatcherInterface> impl) :
        _impl{std::move(impl)}
    {}

    template<typename Impl>
    Matcher(Impl&& impl) :
        Matcher{std::make_shared<std::decay_t<Impl>>(std::forward<Impl>(impl))}
    {}

    bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) override final
    {
        return _impl->matches(current, boundCursors, finder);
    }

private:
    std::shared_ptr<MatcherInterface> _impl;
};
```

*I've omitted an SFINAE check in the second constructor that checks if
`Impl` inherits from `MatcherInterface`, wich is neccesary to remove
ambiguities (It also gives "better" error messages if you instance
`Matcher` with something else).*

*If you look at `ASTMatchersInternal.h` from Clang, `MatcherInterface` is
similar to Clang's `MatcherInterface`, and `Matcher` has the role of
`Matcher<T>` (Where `T` is the Clang AST node type supported).*

### Boolean matchers

Before we continue writing more complex matchers, we need some building
blocks to do basic logical operations with our matchers.

Doing n-ary and/or is basically the same fold operation changing the
operator to apply, so let's be a bit smart and write a `VariadicMatcher`
template that handles this in a generic way:

``` cpp
template<typename Op, bool Seed, typename... InnerMatchers>
class VariadicMatcher
    : public MatcherInterface,
    : public cpp::MemberFunctor<Op>
{
public:
    template<typename... InnerMatchers_>
    VariadicMatcher(InnerMatchers_&&... innerMatchers) :
        _innerMatchers{std::make_tuple(std::forward<InnerMatchers_>(innerMatchers)...)}
    {}

    bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) override final;

private:
    std::tuple<InnerMatchers> _innerMatchers;
};
```

*Of course I'm not that smart, Clang's implementation also uses
a `VariadicMatcher` template, but with a vector of type-erased matchers
instead of a tuple.*

The implementation of `VariadicMatcher::match()` is pretty
straightforward, if you have the right tools:

``` cpp
bool VariadicMatcher::matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
)
{
    return cpp::foldTuple([&](bool previous, auto& innerMatcher)
    {
        return this->cpp::MemberFunctor<Op>::invoke(
            previous,
            innerMatcher.matches(current, boundCursors, finder)
        );
    }, Seed, _innerMatchers);
}
```

`cpp::MemberFunctor<Op>` is basically a template representing a member
functor variable, such as an allocator, applying EBO if possible.
`cpp::foldTuple()` applies a fold operation over the elements of
a `std::tuple()`.

*I always called these things "functors"...*

*I recommend watching Vittorio Romeo's "Implementing static control flow
in C++14" talk to get a general idea about how to implement constructions
like `cpp::foldTuple()`.*


Once we have `VariadicMatcher`, n-ary and/or are just a couple of aliases:

``` cpp
template<typename... InnerMatchers>
using AllOfMatcher = VariadicMatcher<
    std::logical_and<>,
    true,
    InnerMatchers...
>;

template<typename... InnerMatchers>
AllOfMatcher<std::decay_t<InnerMatchers>...>
allOf(InnerMatchers&&... innerMatchers)
{
    return { std::forward<InnerMatchers>(innerMatchers)... };
}

template<typename... InnerMatchers>
using AnyOfMatcher = VariadicMatcher<
    std::logical_or<>,
    false,
    InnerMatchers...
>;

template<typename... InnerMatchers>
AnyOfMatcher<std::decay_t<InnerMatchers>...>
anyOf(InnerMatchers&&... innerMatchers)
{
    return { std::forward<InnerMatchers>(innerMatchers)... };
}
```

There's an special case, the not operator, represented by the
`UnlessMatcher` class:

``` cpp
template<typename InnerMatcher>
class UnlessMatcher : public MatcherInterface
{
public:
    UnlessMatcher(const InnerMatcher& innerMatcher) :
        _innerMatcher{innerMatcher}
    {}
    UnlessMatcher(InnerMatcher&& innerMatcher) :
        _innerMatcher{std::move(innerMatcher)}
    {}

    bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) override final;

private:
    InnerMatcher _innerMatcher;
};

template<typename InnerMatcher>
UnlessMatcher<std::decay_t<InnerMatcher>>
unless(InnerMatcher&& innerMatcher)
{
    return { std::forward<InnerMatcher>(innerMatcher) };
}
```

What makes `UnlessMatcher` special is that it "inverts" the result of the
inner matcher, so we have to be very careful about the side effects of the
matcher:

``` cpp
bool UnlessMatcher::matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
)
{
    BoundCursors dummy;
    return !_innerMatcher.matches(current, dummy, finder);
}
```

Note how we pass a dummy `BoundCursors` to the inner matcher. The idea is
that binding matchers do bind when they find a match, but since an inner
bind matcher is not aware of being surrounded by an `unless()`, we have to
ignore this potential binds in some way.

*Again, this is exactly what Clang implementation does.*

We have `and`, `or`, and `not`; in the form of `allOf()`, `anyOf()`, and
`unless()` matchers. So far so good.

### Complex matchers

Let's visit (no pun intended) `ClassDeclMatcher` again. If you check out
the example again, you will notice that `classDecl()` and the
`ClassDeclMatcher` constructor had no arguments.  
Well, the reason for this is that boolean matchers were introduced later
in the post... Let's fix it now:

``` cpp
class ClassDeclMatcher : public MatcherInterface
{
public:
    template<typename... InnerMatchers>
    ClassDeclMatcher(InnerMatchers&&... innerMatchers) :
        _innerMatchers{allOf(std::forward<InnerMatchers>(innerMatchers)...)}
    {}

    bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) override final
    {
        return current.kind() == CursorKind::Kind::ClassDecl &&
               _innerMatchers.matches(current, boundCursors, finder);
    }

private:
    Matcher _innerMatchers;
};

template<typename... InnerMatchers>
ClassDeclMatcher classDecl(Matchers&&... innerMatchers)
{
    return { std::forward<InnerMatchers>(innerMatchers)... };
}
```

Now you can write a complete class declaration matcher like in the first
ASTMatchers example we seen at the beginning of the post:

``` cpp
classDecl(has(cxxMethod(unless(isStatic()))))
```

*In the real implementation, matchers like `classDecl()` are implemented
through a `KindMatcher<Kind>` template and a macro to generate the matcher
function. Clang does something similar, since there are dozens of
different kinds of nodes you may want to match.*

### AstMatchFinder

All the matchers we implemented work on properties of the current cursor
only. But if a matcher needs checking properties of other surrounding
nodes, such as a property of the parent or a child, it needs some way to
access the AST.

Instead of manually jumping the AST with visitors, Clang defines the
`AstMatchFinder` interface (With a beaufitul `TODO: Find a better name`),
which gives functions to check if one of the surrounding nodes match
a given matcher:

``` cpp
class AstMatchFinder
{
public:
    virtual ~AstMatchFinder() = default;

    virtual bool matchesDescendant(
        MatcherInterface& matcher,
        const Cursor& root,
        BoundCursors& boundCursors,
        std::size_t maxDepth
    ) = 0;

    bool matchesChild(
        MatcherInterface& matcher,
        const Cursor& root,
        BoundCursors& boundCursors
    );

    virtual bool matchesAncestor(
        MatcherInterface& matcher,
        const Cursor& root,
        BoundCursors& boundCursors,
        std::size_t maxDepth
    ) = 0;

    bool matchesParent(
        MatcherInterface& matcher,
        const Cursor& root,
        BoundCursors& boundCursors
    );
};
```

As you can see, what the `AstMatchFinder` class does is to provide
operations to test if a given matcher matches on an *ancestor* or
*descendant* of a given `root` cursor (Where `matchesChild()` and
`matchesParent()` are special cases of the other two).

With this interface, one can implement hierarchical matchers, like
`has()`:

``` cpp
template<typename... InnerMatchers>
class ChildMatcher : public MatcherInterface
{
public:
    template<typename... InnerMatchers_>
    ChildMatcher(Innermatchers_&&... innerMatchers) :
        _innerMatchers{allOf(std::forward<InnerMatchers_>(innerMatchers)...}
    {}

    bool matches(
        const Cursor& current,
        BoundCursors& boundCursors,
        AstMatchFinder& finder
    ) override final
    {
        return finder.matchesChild(
            _innerMatchers,
            current,
            boundCursors
        );
    }

private:
    AllOfMatcher<InnerMatchers...> _innerMatchers;
};

template<typename... InnerMatchers>
ChildMatcher<std::decay_t<InnerMatchers>...>
has(InnerMatchers&&... innerMatchers)
{
    return { std::forward<InnerMatchers>(innerMatchers)... };
}
```

Bringing back our initial ASTMatchers example:

``` cpp
classDecl(has(cxxMethod(unless(isStatic()))))
```

What that DSL expression is really doing is:

 1. Set up a `ClassDeclMatcher`.
 2. Set a `ChildMatcher` as its inner matcher.
 3. Set a `CxxMethodMatcher` as the inner matcher of the `has()`.
 4. Set an `UnlessMatcher` as the inner matcher of `cxxMethod()`.
 5. Set an `IsStaticMatcher` as the inner matcher of `unless()`.

Which in plain English means more or less:

> Match a cursor with kind `ClassDecl` which has at least one child cursor
> with kind `CxxMethod`, unless the found method is static.

That's a lot of information in just one line, isn't?

### Implementing ASTMatchFinder

Since we've started with ASTMatchFinder API we have not seen any visitor
at all. So, there should be one point in the implementation of the API
that actually visits the AST right?

`ASTMatchFinder` implementation is one of these points.

Check `ASTMatchFinder` declaration again. How we can check if a descendant
node of a given node `root` matches a given matcher? Simple: Run
a recursive AST visitor until it finds a match, or reaches the maximum
recursion depth (The maximum distance allowed from the root to a matching
node):

``` cpp
class MatchChildVisitor : public RecursiveVisitor
{
public:
    MatchChildVisitor(
        MatcherInterface& matcher,
        BoundCursors& boundCursors,
        AstMatchFinder& finder,
        std::zize_t maxDepth
    ) :
    _matcher(matcher),
    _boundCursors(boundCursors),
    _finder(finder),
    _maxDepth{maxDepth},
    _matches{false},
    _root{Cursor::null()]
    {}

    Visitor::Result onCursor(
        RecursiveVisitor::Tag,
        const Cursor& current,
        const Cursor& parent
    ) override final;

    bool findMatch(const Cursor& root);

private:
    ...
};

bool MatchChildVisitor::findMatch(const Cursor& root)
{
    _matches = false;
    _root = root;
    Visitor::visit(root);
    return _matches;
}
```

*You may notice that `onCursor()` has a tag, that's why my visitors are
hierarchical, in a sense you can compose new visitors by overriding the
behavior of others. This is done to easily change the traversal algorithm
(the "route" the visitor follows across the AST) of the visitor.*

So basically `MatchChildVisitor` is a recursive AST visitor with
a `findMatch()` method that says if a match is found on the given cursor.
`findMatch()` justs starts visiting the AST from the given `root` cursor
and returns if a match was found while visiting this subtree.

All the magic happens in the `RecursiveVisitor` callback, `onCursor()`:

``` cpp
Visitor::Result MatchChildVisitor::onCursor(
    RecursiveVisitor::Tag,
    const Cursor& current,
    const Cursor& parent
)
{
    if(current.distanceToAncestor(_root) <= _maxDepth)
    {
        if(_matcher.matches(current, _boundCursors, _finder))
        {
            _matches = true;
        }

        return Visitor::Result::Continue;
    }
    else
    {
        // Do not go deeper
        return Visitor::Result::Break;
    }
}
```

Basically what we do is test if we match in the current node and continue
searching, or not to continue down the AST if we already reached the max
depth. If you look carefully, with this implementation `_matches` could be
set to true multiple times, which may seem as a waste of time. This is
intended, to let binding matchers to find multiple matches and cache all
the results.

*In the Clang implementation this works different: What Clang does is to
have an equivalent of `BoundCursors` for each level of the AST, linking
all together with a so called `BoundNodesTreeBuilder`. After processing
the AST, Clang traverses this tree builder with a custom visitor, each
visited entry representing a partial result that is notified with its own
callback call. In other words, the Clang implementation has the hability
to callback all submatches. Since I don't need that feature, I simplified
handling of bound cursors and match results.*

Now that we have a visitor that checks if a given matcher matches in one
or more of the descendants of a node, implementing
`AstMatchFinder::matchesDescendant()` is pretty straightforward:

``` cpp
class MatchingAstVisitor
    : public RecursiveVisitor
    , public AstMatchFinder
{
private:
    bool matchesDescendant(
        MatcherInterface& matcher,
        const Cursor& root,
        BoundCursors& boundCursors,
        std::size_t maxDepth
    ) override final
    {
        MatchChildVisitor visitor{
            matcher,
            boundCursors,
            *this,
            maxDepth
        };

        return visitor.findMatch(root);
    }
};
```

Note the third argument of the `MatchChildVisitor` constructor: This
`MatchingAstVisitor` object is a `AstMatchFinder` (We were implementing
one of the operations of that interface, right?), so **we can pass this
same object as finder for the inner search**.

*`AstMatchFinder::matchesAncestor()` implementation does not use an AST
visitor but goes up in the hierarchy (from parent to parent) trying to
match. I leave its implementation as an excercise for the reader.*

As its name says, `MatchingAstVisitor` is not only a `AstMatchFinder` but
also a recursive visitor. Why? Because **this is the recursive visitor
that will be used for the global (outermost) traversal of the AST**.

## The outermost traversal of the AST

So far we have an AST, matchers, and something that finds inner matches
from a point in the AST. The only thing left is something to take a bunch
of matchers, an AST, and try to find as much matches as possible. That's
the second role of `MatchingAstVisitor`:

``` cpp
class MatchingAstVisitor
    : public RecursiveVisitor
    , public AstMatchFinder
{
public:
    MatchingAstVisitor(
        ArrayView<std::pair<Matcher, MatchCallback*>> matchers
    ) :
        _matchers{matchers}
    {}

private:
    void match(const Cursor& cursor);

    Visitor::Result onCursor(
        RecursiveVisitor::Tag,
        const Cursor& current,
        const Cursor& parent
    ) override final;

    ArrayView<std::pair<Matcher, MatchCallback*>> _matchers;
};
```

At the beginning of the post we saw that the whole point of ASTMatchers
API was to define matchers and to be notified whenever a match is found.
`MatchingAstVisitor` takes pairs `(matcher, callback)` and for each node
in the AST all matchers are tested. If a match is found, the corresponding
callback is invoked:

``` cpp
void VMatchingAstVisitor::match(const Cursor& root)
{
    for(auto& pair : _matchers)
    {
        auto& matcher  = pair.first;
        auto& callback = *pair.second;

        BoundCursors boundCursors;

        if(matcher.matches(root, boundCursors, *this))
        {
            MatchResult matchResult{root, matcher, boundCursors};
            callback.onMatch(matchResult);
        }
    }
}

Visitor::Result MatchingAstVisitor::onCursor(
    RecursiveVisitor::Tag,
    const Cursor& current,
    const Cursor& parent
)
{
    match(current);
    return Visitor::Result::Continue;
}
```

### The user interface: MatchFinder

We are finally here, at the final piece of the API: `MatchFinder`.
`MatchFinder` class is basically a container of matchers and its
associated callbacks, an API where the user can register matchers and
start a search.

``` cpp
class MatchFinder
{
public:
    template<typename UserDefinedMatcher>
    void addMatch(UserDefinedMatcher&& matcher, MatchCallback& callback)
    {
        _matchers.emplace_back(
            std::forward<UserDefinedMatcher>(matcher),
            &callback
        );
    }

    void matchAst(const TranslationUnit& tu)
    {
        MatchingAstVisitor visitor{_matchers};
        visitor.visit(tu.cursor());
    }
private:
    std::vector<std::pair<Matcher, MatchCallback*>> _matchers;
};
```

Putting all together:

``` cpp
class MyMatchCallback : public MatchCallback
{
public:
    void onCallback(const MatchResult& result) override final
    {
        for(const auto& method : result.boundCursors().getAll("methods"))
        {
            std::cout << "Class '" << result.root().spelling()
                      << "', public method '" << method.displayName()
                      << "'\n";
        }
    }
};

Index index;
TranslationUnit tu = index.parse(...);

MyMatchCallback callback;
MatchFinder finder;

finder.addMatcher(
    classDecl(
        has(id("methods", cxxMethod(isPublic())))
    ),
    callback
);

matcher.matchAst(tu);
```

## Summary

Briefly, today we have learned that:

 - I should start working on the blog again to avoid writing walls of text
 - Working with AST visitors directly can be complex, and the Clang C++
   API provides a higher level alternative: ASTMatchers
 - ASTMatchers is not exposed by libclang, but we can replicate the Clang
   ASTMatchers implementation using libclang AST visitors

I should note I left some classes out of the post (For not being specially
complex). Some cases:

 - **The `MatchCallback` interface**: We've seen at least two examples of
   its usage, so I thing anyone can figure out its definition.
 - **`MatchResult`**: `MatchResult` is just an encapsulated aggregation of
   all the stuff involved in the match: The root cursor of the matching
   sub-tree, the matcher involved, and the index of cursors bound to the
   match.
 - **Matchers checking properties of cursors**: In the implementation
   I have a set of matchers that represent cursor properties from libclang
   API functions with signature `Result(Cursor)`. Those matchers declare
   one or more expected values test the property against cursors. Examples
   of such matchers are `isStatic()`, `isPublic()`, `hasName()`, etc.
 - **`BoundCursors`**: Meh, it is basically
   a `std::unordered_map<std::string, std::vector<Cursor>>`.
 - **Binding matchers**: I left the implementation of the `id()` matcher
   out of the post. Basically what matchers like `id()` do is to take
   a name and an inner matcher, and check if the inner matcher matches in
   the cursor. If it does, `id()` registers the current cursor in
   `boundCursors` with the given name.

Also, I intentionally used my libclang C++ wrapper instead of the libclang
C API in the examples. The point is that you can build a matchers API for
any kind of visitation API, regardless of the backend.

I think that's all. I hope you like this, it has been a loooooot of work
in the last two months.

Happy AST traversal!
