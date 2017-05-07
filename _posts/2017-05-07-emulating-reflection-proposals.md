---
title: Emulating reflection proposals
date: 2017-05-07
---

Despite this year is being far less productive than I expected (*Well, that's not fair, past year I used to work from 09:00 to 18:00 + 3h of commuting, then play with C++ from 22:00 to 02:00-06:00 depending on the day. I was so exhausted by the end of the year I decided to slow down my C++ fever a bit*), I'm still throwing commits from time to time towards the refactorization of siplasplas (See [the previous post](https://manu343726.github.io/2017/03/13/lock-free-job-stealing-task-system-with-modern-c.html)), also, while not actively working on the project nor involved in the last cool C++ conferences, I try to be in sync of the latest work on this topic.

In the last few weeks I was so glad that Jackie Kay publicly joined the Official Group Of Folks Obsessed With C++ Reflection by starting [an interesting series](http://jackieokay.com/2017/04/13/reflection1.html) on the state of reflection in C++ and the two main flavors being proposed to the standard.

I agree with Jackie that the future of reflection must go for a value-based API. In its current state, the siplasplas static reflection API
uses a type-based meta-object model similar to the `reflexpr` operator proposal. Since I don't have language support for a custom operator with custom semantics but a libclang tool to hack into the user code, providing different templates (`Enum<T>`, `Class<T>`, etc) as metadata getters was a simple way to make an introspection API available at compile time. The instances of that templates contain the different metadata as typelists, static member functions, etc. Basically the API is designed with classic type-metaprogramming in mind (In fact, the generated code containing the metadata and the static reflection API are C++11 compatible, despite the library is advertised as "C++ 14 reflection"). To get an idea, this is the equivalent of Jackie's equality operator example using my API:

``` cpp
template<typename T>
bool equal(const T& lhs, const T& rhs)
{
    bool result = true;

    cpp::foreach_type<cpp::static_reflection::Class<T>::Fields>([&](auto type)
    {
        using Field = typename decltype(type)::type;

        result &= equal(Field::get(lhs), Field::get(rhs));
    });

    return result;
}
```

In the example `Class<T>` is the meta-object with the metadata of the class type `T`, and `Fields` is a
member typelist with instances of `Field<Type, Pointer>` in it. `cpp::foreach_type<>()` is an ugly template to
iterate over the elements of a typelist. Despite the code may look similar to the `for_each()` (Which seems to work on tuples
instead of typelists) from cpp3k, the example follows the same ideas of the `reflexpr` example using a different API structure.  
*Check [my Meeting C++ 2016 slides](https://github.com/Manu343726/meetingcpp2016) if you like, there are more examples like JSON and protobuf serialization*.

The advantages of a type-based reflection API are clear: The ubiquitous "Modern C++ Design style" of type metaprogramming that C++ wizards were
used to master for the last fifteen years would also serve to master the introspection of user defined types. But this also introduces an important
disadvantage: Not all people like to apply some arcane spells in order to figure out the public member variables of a class. Not all people want to have to know all the dark corners of C++ in order to use an API (I'm looking at you boost...). **We must design a so requested and powerful feature
like reflection with the average C++ programmer in mind**. That's why a value-based API, with dots instead of colons between metadata layers,
is the way to go.

There's another issue with the type-based API, one I experienced myself during the design and development of siplasplas: API cognitive-overhead.  
That's right, cognitive overhead. It happens that after three months getting used to all that template stuff you realize that loading user APIs at
runtime would be a good idea. So you would open cppreference to check the standard dynamic reflection API, and what you see? A completely different beast.
By definition a dynamic reflection API, even if built around the features of a type-based static reflection one, would not be type-based at all but
similar to what C# and Java do: Objects containing metadata, string-based queries, maps, etc. So you end up with a completely different API, to do
what is basically the same. **You end up having to learn two completely different models of the same metadata**.

*That's not chronologically exact to what it happened to me, but it goes to the point anyway...*

So, why is value-based reflection so good? Because it has the power to provide **exactly the same meta-object model for both static and dynamic reflection**.  
Since C++11 we have the right to abuse AST evaluation to do almost anything we want at compile time. `constexpr` enabled value-based compile-time computing in C++, which is just a convoluted way to say "Compile-time related code may look similar to your day-to-day runtime code", which is awesome. One not-so-known feature of `constexpr` is that you can write `constexpr` classes, that is, classes that may be instanced at compile time. For example:

``` cpp
struct Point
{
    int x, y;

    constexpr Point(int x = 0, int y = 0) :
        x{x}, y{y}
    {}
};
```

`Point` declares a user-defined `constexpr` constructor, which enables users to instance a `Point` object using that constructor in contexts evaluated at compile-time, such when setting the length of an array:

``` cpp
constexpr Point p{42, 1};
using MyArray = int[p.x*p.y];
```

While not so useful in another contexts, `constexpr` enabled entities (functions and classes) have the ability of switching to normal "runtime" functions/classes when the given parameters are not evaluable at compile time. That means `Point` could act as a common `struct` as well:

``` cpp
int main(int argc, char* argv[])
{
    if(argc >= 3)
    {
        Point p{std::atoi(argv[1]), std::atoi(argv[2])};

        std::cout << p.x << ", " << p.y;
    }
}
```
Imagine the following case:

``` cpp
namespace reflection
{

class Enum
{
public:
    constexpr Enum(std::size_t count, const char* names[], const std::int64_t values[]) :
        _count{count}, _names{names}, _values{values}
    {}

    constexpr bool hasWithName(const char* name) const
    {
        return constexpr_find(_names, _names + _count, name);
    }

    constexpr bool hasWithValue(std::int64_t value) const
    {
        return constexpr_find(_values, _values + _count, value);
    }

    constexpr const char* name(std::size_t i) const
    {
        return _names[i];
    }

    constexpr const char* value(std::size_t i) const
    {
        return _values[i];
    }

    constexpr std::size_t count() const
    {
        return _count;
    }

private:
    std::size_t _count;
    const char** const _names;
    const std::int64_t* const _values;
};

}
```

The above class represents the meta-object of an enumeration type. In an hypothetical `$` operator API, it would work as follows:

``` cpp
enum MyEnum
{
    One = 1,
    Two,
    Three
};

static_assert($MyEnum.count() == 3, "");
static_assert(constexpr_streq($MyEnum.name(1), "Two"), "");
static_assert($MyEnum.value(2) == 3, "");
``` 

While an hypothetical dynamic reflection API would be something like this:

``` cpp
reflection::dynamic::Runtime runtime{DynamicLibrary::load("libfoo.so")};

assert(runtime.has("MyEnum"));
assert(runtime.entity("MyEnum").kind() == reflection::dynamic::Entity::Kind::Enum);

Enum enumMetaObject = runtime.enum("MyEnum");

assert(enumMetaObject.count() == 3);
assert(enumMetaObject.name(1) == std::string{"Two"});
assert(enumMetaObject.value(2) == 3);
```

Where `Enum`, the class representing enumeration meta-objects, **is exactly the same from the static reflection API**.

In order to make this possible, we need a couple of things just to make sure we don't go nuts in the process:

 - **`constexpr` equivalents of Standard library algorithms and data structures**: I have [a siplasplas module just for this](https://manu343726.github.io/siplasplas/doc/doxygen/master/group__constexpr.html) (Master is way behind its current state, with arrays, strings, string related algorithms, etc). The idea is to have the same toolbox you are used to in C++. Else you end up writing the same code hundreds of times.

 - **C++14 extended `constexpr` support**: This could be optional, but having to write everything in terms of the ternary conditional operator and recursion could be rather insane, with the resulting algorithms being horribly slow at runtime. Sadly this is the only currently available path if you whish to support Visual Studio. *This is what I currently do and, trust me, I dream about dropping VS support everyday...*

This is going well as a kind of mental experiment, but what if we want to implement the `$` **now**, with C++14?

I have been thinking about this for a while. The first step is obvious, the operator must be defined as a macro:

``` cpp
#define $(T)
```

which changes the syntax a bit, but still looks pretty much like what's being proposed to the standard:

``` cpp
static_assert($(MyEnum).count() == 3, "");
```

*What would be great is that the proposal explicitly allowed parens in operator `$` expressions, so this new feature could replace the C++14 emulation by just undefining the macro... (If I do my work right implementing the same proposed API, of course)*

Keep in mind that the goal of `$` emulation is to have a common reflection API entry point, to substitute the per-kind entry points described at the beginning of the post (Those `Enum<T>`, `Class<T>`, etc templates). First, let's examine what information is available at the point of macro instantiation:

``` cpp
# 0 // foo.cpp
# 1
# 2
# 3 static_assert($(MyEnum).count() == 3, "");
# 4
# 5
```

From the macro point of view, there are three things clearly visible: The file where the macro is instantiated (`__FILE__` or equivalent), the line number (`__LINE__` or equivalent), and the type identifier token ("MyEnum").

Now the hard thing is to figure out the kind of meta-object to return from the `$` expression, which depends completely in the identifier passed to it. Let's sketch the C++ side (Out of the Preprocessor Land and back to the Magical Reign Of C++ Templates) of the API entry point:

``` cpp
template<typename EntityRef>
constexpr MetaObject<EntityRef> getMetaObject()
{
    return EntityData<EntityRef>::getMetaObject();
}
```

Let's examine that function line by line:

 - Its input is a type representing a reference to a C++ entity. How do we reference an entity? By its name (The identifier token) and the point from which we reference the entity (File and line). The location is needed to keep reference context, since the referenced entity may depend on this context (Such as when we reference a type by its non-qualified name).
 - `MetaObject<Ref>` is a metafunction returning the meta-object type (`Enum`, `Class`, etc) corresponding to the given
   entity reference. Of course we could rely in C++14 return type deduction, but the point is to show you that we could keep all this emulation C++11 compatible if we liked to.
 - `EntityData<>::getMetaObject()` takes the reference as input doing the corresponding lookup in the metadata, returning the right meta-object.

But, how we lookup the metadata given that "reference"? What metadata?

As explained before, siplasplas models meta-objects as templates with the metadata as type and static data members. A fixed set of different kinds of entities are supported, mainly `Enum<>` and `Class<>`. The library backend defines templates for each kind, and a code generator generates specializations of these templates for each entity found. The "key" to the API are the type identifiers themselves, lookup implemented by your C++ compiler as C++ template instantiation rules.

``` cpp
// foo.h

class Foo
{
    int i;
};
```

``` cpp
// foo.h generated code (reflection/foo.h)
#include <foo.h>

template<>
class Class<Foo>
{
public:
    using Fields = typelist<
        Field<
            SourceInfo<
                string<'i'>,                     // name
                string<'f', 'o', 'o', '.', 'h'>, // file
                4                                // line
            >,
            decltype(&Foo::i), &Foo::i, // Pointer
        >
    >;
};
```

``` cpp
// main.cpp
#include <foo.h>
#include <reflection/foo.h>

using FooFields = Class<Foo>::Fields;
```

The main problem with that approach is that those API entry points are not heterogeneous, since C++ lacks the concept of a "heterogeneous template parameter". Member pointers (Represented by `Method<>` and `Field<>`) take two parameters (Pointer type and the pointer itself), classes and enums take one (The user type), etc. But this has the advantage that the entity lookup rules is implemented by the compiler itself, which already knows the rules of C++ type name lookup.

In order to create an unified API entry point, we have to find a common representation of these entity references. The solution is simple for the line number (Which is an integer, supported by C++ templates), but what about the entity token and the filename? How we can pass those to the templates?  
The answer is to keep in mind that the original reference information could be lost, since all the interesting metadata is already kept by the generated templates, included these strings. We don't need to pass the strings to the templates, we can just hash them. That's the trick: Instead of specializing on the user types, specialize on entity references, which are a tuple `(hash(entity identifier), hash(filename), line number)`:

``` cpp
#define $(T) getMetaObject<EntityRef<hash(CPP_STRINGIFY(T)), hash(__FILE__), __LINE>>();
```

The tricky part is to make sure that the code generator uses the same hashing algorithm than the library frontend. This is easy if you embed the strings directly in the generated code and force the compiler to compute the hashes itself:

``` cpp
// foo.h generated code (reflection/foo.h)
#include <foo.h>

template<Hash FilenameHash, std::size_t LineNo>
class EntityData<EntityRef<hash("Foo"), FileNameHash, LineNo>>
{
public:
    using Fields = typelist<
        Field<
            SourceInfo<
                string<'i'>,                     // name
                string<'f', 'o', 'o', '.', 'h'>, // file
                4                                // line
            >,
            decltype(&Foo::i), &Foo::i, // Pointer
        >
    >;

    static constexpr Kind kind = Kind::Class;

    static constexpr Class getMetaObject()
    {
        return ...
    }
};
```

Note there's no longer one template for each entity kind, but a common `EntityData` which, in its simplest form, uses entity references as template parameters.

For value-based reflection, the actual `constexpr` meta-objects can be built from the template metadata, invoking `getMetaObject()`. `getMetaObject()` builds a meta-object from references to the metadata. The following would be the `MyEnum` example metadata:

``` cpp
template<Hash FileNameHash, std::size_t LineNo>
class EntityData<hash("MyEnum"), FileNameHash, LineNo>
{
    using Names = typelist<
        string<'O', 'n', 'e'>,
        string<'T', 'w', 'o'>,
        string<'T', 'h', 'r', 'e', 'e'>
    >;

    using Values = integral_sequence<std::int64_t,
        1,
        2,
        3
    >;

    static constexpr Kind kind = Kind::Enum;

    static constexpr Enum getMetaObject()
    {
        return Enum{
            typelistLength<Names>(),
            typelistToArray<Names>(),
            typelistToArray<Values>()
        };
    }
};
```

Putting it all together:

``` cpp
static_assert($(MyEnum).count() == 3, "");
```

Of course the lookup algorithm must be fixed, in its current state is so simple that it would lead to entity ambiguities very easy. Also, it only works with non-qualified references. To fix this problems, we could grasp a set of lookup rules:

 1. **All full-qualified references to an entity reference to the same entity, regardless of the context (location)**. This means the least-specialized template must point to the full qualified entity name, not the unqualified:

    ``` cpp
    template<Hash FileNameHash, std::size_t LineNo>
    class EntityData<hash("::MyEnum"), FileNameHash, LineNo>
    {
        ...
    };
    ```
 2. **Non full-qualified references are organized in code regions**. That is, for each non-full-qualified reference possible (If the entity is `::foo::Bar`, all the possible references are "Bar", "foo::Bar", "foo::Bar", and "::Foo::Bar") an specialization pointing to the full-qualified metadata is generated. These specializations are SFINAEd by the regions of code where that reference combination could apply. This is not simple since you have to examine all user code for cross-references to the entity in the AST computing all the AST sub-trees (with its corresponding regions of code) where that specific reference combination could point to the entity.

Rule 2, which basically tries to implement C++ type name lookup rules, sounds crazy but it can be done with enough patience. But there's another option: What if we could get the full qualified name of an entity at compile time? If that were possible we could point any identifier token (Such as `Foo`) to the full qualified entity (`::foo::Foo`), relying again in the compiler to implement the name lookup rules.  
The good news is that this can actually be done. [ctti](https://github.com/Manu343726/ctti), a library I wrote a couple of years ago, implements a set of non-standard (but cross-platform) tricks to have static equivalents of RTTI features at compile time. The trick consists in parsing the output of `__PRETTY_FUNCTION__` and its counterparts at compile time, getting the full qualified name of a type out of a template's `__PRETTY_FUNCTION__` output.  
If we rewrite our `$` operator as follows:

``` cpp
#define $(T) getMetaObject<EntityRef<hash(ctti::type_id<T>().name()), hash(__FILE__), __LINE__>>()
```

Now the `EntityRef` is always the full-qualified reference to the entity, regardless of the identifier passed to the `$()` macro by the user.

As you can see, we can get value-based `operator $` reflection for C++11/14. Cool isn't?