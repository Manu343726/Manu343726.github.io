---
title: 'The road for reflection: Tagging types'
date: 2016-03-13 08:35:00 -07:00
tags:
- c++
- reflection
layout: post
---

Something like a month ago I wrote [a post](http://manu343726.github.io/2016/01/30/reflection-intro.html) introducting the reflection
engine I'm writing as part of a C++ course for people in the game programming
master degree of my university. Why me, an undergrad that has the end of his CS
grade faaaaaaar away from his horizon, is giving such a course is a mater of
another post (A post on "*Why I hate the university sooooooo much*"...).

Writing a reflection engine has two primary goals:

 - Game programmers these days are used to a wide pipeline of tools that ease
   with game development, in a way most of them doesn't even write their own
   game engine but only do *the game*. That is, game programming has become a
   game itself. Regardless you think this is good or not (I think is not, game
   programming is not such a challenging/interesting task that was the old days.
   But others may argue that now everybody could write his own game, which is
   good.), the point is that most game programming courses focus on tools
   instead of game enignes/design, and people leave the course (the master in
   this case) without knowing how these tools work. This is true even in courses
   focused on engine programming, since most people don't have the required
   programming skills to deep into the techniques involved in the implementation
   of AAA engines nor frameworks such as unity.

 - It's fun. Nothing more, nothing less.

So the idea is to show people how systems like Unreal Engine's blueprint
integration ***could*** work, and have fun in the process.

*"Could" is an important word here, As I usually tell people in my classes,
what I show is just the way I did the thing, not THE WAY to do it. I took
decissions based on my own context (It should be "teachable", I'm lazy, etc),
but others might follow different approaches. That's what being an engineer
means, right?*

As the roadmap in the previous post shows, one of the first tasks we have
to aboard is the way we store C++ type information for our engine, in a way it
can be used at runtime to instance objects, check function signatures, etc.
Today, I will show you what I did to **store C++ type information at compile
time** and use that info for **tagging C++ types**.

## RTTI

Since what we are doning with runtime reflection is a kind of dynamic type
system for C++, the first thing we need is a way to store type information so it
can be checked at runtime, compare types for equality, etc.

``` cpp
Type intType = getType<int>();
Type charType = getType<char>();

assert(intType != charType);
```

C++ already ships a similar feature in its Standard Library by means of the
`<type_info>` header: **RunTime Type Information**, or RTTI for friends.

C++ RTTI works by storing type information (Its name, a "unique" identifier,
etc) as part of the program binary so we can query this info at runtime with the
[`typeid` operator](http://en.cppreference.com/w/cpp/language/typeid):

``` cpp
#include <type_info>

auto intType = typeid(int);

std::cout << intType.name(); // Prints "int" ?
```

However standard rtti has some caveats:

 - **It can be queried at runtime only**: Please read that again. I find so
   funny that a language that uses its "static type system" almost as an
   advertising motto cannot query the name of a type at compile time. Amazing.

 - **No practical guarantees**: The name that `.name()` gives is mangled, so
   it's almost useless in most scenarios. Of course you can demangle the name
   with your vendor API, but having to maintain a portable demangling API is
   something I would like to avoid. *I did once, and was... well, let's follow.*
   Also the supposed "unique id" that `.hash_code()` returns is only guaranteed
   to be the same for equal types, but **is not guaranteed that different types
   yield different hashes**. Kind of useful.

 - **Size bloat**: AAA game programmers often dislike RTTI since it bloats their
   executable by adding an entry in the symbol table for each type. RTTI data is
   usually backed in classes as a "negative" entry in the vtable of the type,
   like:

   ``` cpp

   class Foo
   {
       virtual void f();
   };

   class Bar : public Foo
   {
       void f() override;
       int i;
   };

   Foo* bar = new Bar();
                                    Bar vtable
                                 +---------------+
              Bar object         | Bar type info |
   bar ---> +------------+       +---------------+
            | vtable ptr | ----> |    &Bar::f    |
            +------------+       +---------------+
            |    int i   |
            +------------+
   ```

   But, really, I don't care. I'm sure deploying to such constrained systems
   like game consoles is hard. But, come on... I think this is a common case of
   bias like the "exceptions are slow" topic. Of course YMMV, but from my
   experience exceptions and RTTI worked pretty damn well on embedded devices.

## CTTI

To solve the two issues above (As I said, I don't consider the third an issue)
my friend [Jonathan "foonathan" Müller](https://foonathan.github.io/) and I
wrote the [ctti library](https://github.com/Manu343726/ctti).  CTTI (From
"*Compile Time Type Information*") aims to provide both demangled
type names and unique hashes at compile time, thanks to C++11 `constexpr:`

``` cpp
#include <ctti/type_id.hpp>

constexpr auto intType = ctti::type_id<int>();
constexpr auto charType = ctti::type_id<char>();

static_assert(intType != charType, "What???");

std::cout << intType.name(); // Gives "int"
```

### How it works

This morning as part of the pre-work to write this post I found myself checking
the code and documentation of Don Williamson's [clReflect
engine](https://github.com/Celtoys/clReflect), a clang-based reflection engine
very similar to this one. *Williamson was the lead engine programmer of games such 
as Fable and Splinter Cell Conviction. [Here](http://www.gamasutra.com/view/news/128978/Reflection_in_C_The_simple_implementation_of_Splinter_Cell.php)'s
a great gamasutra article where he shares the reflection API they wrote for
Conviction*.

[From clReflect
wiki](https://bitbucket.org/dwilliamson/clreflect/wiki/GetType%20Discussion):

> **Simulate C++ RTTI type name retrieval**
>
> To relieve the dependency on the typeinfo header, a function similar to this
> can be introduced:
>

> ``` cpp
> template <typename TYPE>
> const char* GetTypeName()
> {
> #ifdef _MSC_VER
>    return __FUNCSIG__;
> #else
>    return __PRETTY_FUNCTION__;
> #endif
> }
```

This trick is exactly what CTTI does.  Using a `constexpr` function CTTI
"parses" `__FUNCSIG__` and similar expressions at compile time to get the name
of the type out of the string.  "Parses" is a very generous word I think. What
we did is to wrap vendor-specific `__FUNCSIG__`-like expressions and check the
format of its output, so we can get an specific substring (The type name) out of
the string.

This is a bit tricky: Did you notice I never said "string literals" nor
"macros"? It's because such constructs (`__PRETTY_FUNCTION__`, `__FUNCSIG__`,
etc) are not macros **nor string literals** but implicitly declared identifiers
similar to standard `__func__` (From C99, added to C++ with C++11), which is probably 
one of the worst specified points of the standard...  
The weird part of CTTI was to write a `constexpr` string class able to build
substrings **at compile time, with C++11 constexpr only, supporting Visual
Studio**. *Cannot bold that last item enough. That was such a pain in the ass.
See [THE ISSUE](https://github.com/Manu343726/ctti/issues/5)*.

But, what's a `constexpr` class? What's `constexpr` ?

### constexpr

*Feel free to jump over this if you already know about `constexpr`.*

`constexpr` is feature available since C++11 which gives the option of
writing C++ code to be evaluated (Actually interpreted by the compiler) at
compile time. Here's an example:

``` cpp
constexpr float add(float x, float y)
{
    return x + y;
}

constexpr float FLOAT_CONSTANT = add(0.0f, 1.0f);
```
This has lots of benefits, since until C++11 the only way to do complex
computations at compile time was by using hard to read/maintain/easy-to-throw-up
template meta-programming techniques. And this only involved computations on
integral types. *I once wrote a floating point template for TMP, since I wanted
to do 3d transformations at compile time. Trust me, you wouldn't like to put
that kind of code in production...*

In the example above, `add()` is a **`constexpr` function** and `FLOAT_CONSTANT`
a `constexpr` constant. A `constexpr` function is guaranteed to be evaluated at
compile time as long as its arguments can, like in this case. Else, functions
are "downgraded" into a normal C++ functions, to be evaluated at runtime.
The `constexpr` constant there is just a way to force `constexpr`
evaluation of `add()`: These are constants that have to be initialized at
compile time, else compilation fails.

`constexpr` not only applies to plain C functions, but also to member functions, even
**constructors**. So you end up having the ability **to write classes and
instancing objects that are completely evaluated at compile time**. That's so
cool.

Here's an example of a useful `constexpr` member function: `std::array::size()`:

``` cpp
std::array<ichar, 1024> buffer;

read(file, &buffer[0], buffer.size());
```

No more `#define LENGTH (x) (sizeof(x)/sizeof(&x[0]))` tricks. No more C arrays
please. It's 2016 and there's still people introducing bugs in the linux kernel
because of this... *Hey, would you like to hear a TCP joke?*

A `constexpr` class is just a class that has at least one `constexpr` declared
contructor, so the compiler can make instances at compile time:

``` cpp
class string
{
public:
    template<std::size_t N>
    constexpr string(const char (&string_literal)[N]) :
        _str{string_literal},
        _size{N}
    {}

    constexpr std::size_t size() const
    {
        return _size;
    }
private:
    const char* _str;
    std::size_t _length;
};

static_assert(string("foo").size() == string("bar").size(), "???");
```

Also note that `constexpr` implies `inline` linkage. This may be useful when declaring constants, since you're guaranteed to not have [ODR](https://en.wikipedia.org/wiki/One_Definition_Rule) issues when defining the constant at the header. *Why one would care to both declare and define a constant in a header? Well, this just hapenned to me a couple of weeks ago at work, where we have `-Wall -Werror -pedantic` enabled by default and GCC gives you warnings when you don't initialize a constant at its declaration...*

For more info about `constexpr` clases, and `constexpr` in general, I recommend
[this series](https://akrzemi1.wordpress.com/2011/05/11/parsing-strings-at-compile-time-part-i/)
on compile time string parsing.

## Tagging types

Now that we have unique IDs and demangled names, let's pack all useful type info in a
`TypeInfo` `constexpr` class:

``` cpp
class TypeInfo
{
public:
    constexpr TypeInfo(ctti::type_id_t id, std::size_t sizeOf, std::size_t alignment) :
        _id{id},
        _sizeof{sizeOf},
        _alignment{alignment}
    {}

    constexpr const char* name() const
    {
        return _id.name().c_str();
    }

    constexpr ctti::type_id_t id() const
    {
        return _id;
    }

    constexpr std::size_t sizeOf() const
    {
        return _sizeof;
    }

    constexpr std::size_t alignment() const
    {
        return _alignment;
    }

    template<typename T>
    static constexpr TypeInfo get()
    {
        return {
            ctti::type_id<T>(),
            sizeof(T),
            alignof(T)
        };
    }

    friend constexpr bool operator==(TypeInfo lhs, TypeInfo rhs)
    {
        return lhs.id() == rhs.id();
    }

    friend constexpr bool operator!=(TypeInfo lhs, TypeInfo rhs)
    {
        return !(lhs.id() == rhs.id());
    }

private:
    const ctti::type_id_t _id;
    const std::size_t _sizeof;
    const std::size_t _alignment;
};

constexpr TypeInfo intType = TypeInfo::get<int>();
constexpr TypeInfo charType = TypeInfo::get<char>();

static_assert(intType != charType, "???");

std::cout << intType.name(); // prints "int"
```

Our `TypeInfo` class takes the type ID, the `sizeof()` of the type, and its alignment, information that will be useful when instancing objects in following posts. Note both its constructor and the `get<T>()` factory are `constexpr`.

### What's next?

Today we learnt how to write a compile-time type info class with all the
information we will need to follow with the reflection engine. In next posts we
will implement `MetaType`, the class that manages runtime types and knows how to
instance and destroy objects dynamically.

### Acknowledgements

 - Thanks to [Zedalos](http://www.zedalos.org/) for noticing the missing `constexpr` specifier in `TypeInfo` constructor.
 - Thanks to [sehe](http://stackoverflow.com/users/85371/sehe) for that entertaining trolling session on twitter ;)
