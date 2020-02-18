---
title: Reflections On User Defined Attributes
date: 2019-07-14 00:00:00 -07:00
tags:
- c++
comments: true
---

This week there's a ISO C++ meeting in Cologne, and after having the Spanish
national body meeting this past week this looks like a perfect moment to
reflect, one more time, on reflection in C++.

<!-- vim-markdown-toc GitLab -->

* [1. Scope](#1-scope)
* [2. Motivation](#2-motivation)
* [3. Minimal intro to the Reflection TS](#3-minimal-intro-to-the-reflection-ts)
* [4. Minimal attributes support for the Reflection TS](#4-minimal-attributes-support-for-the-reflection-ts)
* [5. Minimal attribute parsing support](#5-minimal-attribute-parsing-support)
    - [Name, full name, and display name](#name-full-name-and-display-name)
    - [Attribute namespace](#attribute-namespace)
    - [Attribute arguments](#attribute-arguments)
    - [Argument tokenization](#argument-tokenization)
* [6. Future](#6-future)
    - [Attribute search API](#attribute-search-api)
    - [Full attribute parsing API](#full-attribute-parsing-api)
    - [Attributes as classes](#attributes-as-classes)

<!-- vim-markdown-toc -->

## 1. Scope

This post tries to be the basis of a future ISO C++ paper discussing the
possible addition of attribute reflection to the Reflection Technical
Specification.

Specifically, here I discuss a possible API to reflect and query
attributes applied to entities already reflected by the Reflection TS.
I also explore different use cases and future directions.

> **Disclaimer**: I know we are in the process of closing C++ 20. We are fixing
> late issues and discussing design and features is no longer allowed (Or
> should not be). This post, and if things go well the future paper that will
> come from it, is not targeted for C++20 in any way. It's just my way to open
> a discussion about the next step on reflection after the TS is published and
> we gain implementation experience.

## 2. Motivation

[Attributes](https://en.cppreference.com/w/cpp/language/attributes) are
basically a way to tag information to a C++ entity. Whether this
information is usefull for the user or the compiler does not matter from
the point of view of reflection, but the respective uses cases are
different.  

In case of bult-in attributes, that information is usually about code
correctness and potential optimizations, such as
[`[[nodiscard]]`](https://en.cppreference.com/w/cpp/language/attributes/nodiscard)
or
[`[[fallthrough]]`](https://en.cppreference.com/w/cpp/language/attributes/fallthrough):

``` cpp
int [[nodiscard]] fopen(const char* filename);
```

User defined attributes are different in that the compiler ignores them,
but those attributes may have an special meaning or carry useful
information for the user of an specific API. For example:

``` cpp
class [[tinyrefl::rename("AwesomeClass")]] AwfulClass
{
public:
  void doSomething();
};
```

Here the `[[tinyrefl::rename()]]` user defined attribute tells the
[tinyrefl](https://github.com/Manu343726/tinyrefl) reflection system to
ignore the original name of the class and instead register it using
a custom name `"AwesomeClass"`. This attribute is ignored by the compiler,
and only has any meaning in the context of the system or API that
implements or "reads" this attribute. In this specific case, changing the
behavior of an external parsing tool.

Another use case for user defined attributes is a generic reflection-based
serialization system that uses attributes to configure the serialization
behavior of some members of a class:

``` cpp
struct vector
{
  float x, y, z;

  [[serializer::ignore]]
  float length;
};
```

This same mechanism can be extended to more complex cases: [unittest]() is
a proof of concept of a unit test and mocking framework that uses user
defined attributes to select functions and classes to mock at compile
time, monkey-patching those when entering the scope where the
`[[unittest::patch()]]` user defined attribute was applied:

``` cpp
[[unittest::patch("Socket::sendBytes(const std::vector<char>&)")]]
void test_dataSentThroughSocketOnAssignment(
    unittest::MethodSpy<int(const std::vector<char>&)>& sendBytes)
{
  ObjectProxy<int> proxy{"localhost:12345"};

  proxy = 42;

  sendBytes.assert_called_once_with(toNetworkEndianessBytes(42));
  assertEqual(sendBytes.calls[0].result, sizeof(int));
}
```

[One can also imagine
a way](https://manu343726.github.io/2019-06-20-raytracer-runtime-postmortem/)
to generate the argument parser of a C++ cli application by reflecting
a `settings` structure:

``` cpp
template<typename Settings>
void add_options(cxxopts::Options& options)
{
    tinyrefl::visit<Settings>(tinyrefl::member_variable_visitor(
        [&](const auto& var) { add_option(var, options); }));
}

template<typename MemberVariable>
void add_option(MemberVariable, cxxopts::Options& options)
{
    constexpr MemberVariable var;
    using value_type = typename MemberVariable::value_type;

    constexpr auto name        = option_name(var);
    constexpr auto short_name  = option_short_name(var);
    constexpr auto description = option_description(var);

    if constexpr(
        std::is_class_v<value_type> && tinyrefl::has_metadata<value_type>())
    {
        add_options<value_type>(options);
    }
    else if constexpr(is_istream_readable<value_type>())
    {
        options.add_option(
            "",
            short_name.str(),
            name.str(),
            description.str(),
            cxxopts::value<value_type>(),
            "");
    }
}

template<typename MemberVariable>
constexpr auto option_description(MemberVariable)
{
    constexpr MemberVariable var;

    if constexpr(var.has_attribute("rt::description") &&
       var.attribute("rt::description").arguments().size() == 1)
    {
        return var.attribute("rt::description").arguments()[0].pad(1, 1);
    }
    else
    {
        return tinyrefl::string{""};
    }
}
```

As you can see user defined attributes open a world of possibilities for domain
specific properties, nothing new if we consider that developers have been
implementing similar use cases for years in the means of macros or custom
parsers.  

Having reflection built into the language is a great step forward, but the same
users that currently use reflection extensively in their codebases (Game
engines, ORMs, UI frameworks, etc) also have some some form of attribute
tagging. Supporting reflection of attributes **is the only way we will
eliminate the need for custom parsers for most of the cases**.

## 3. Minimal intro to the Reflection TS

> Note this post does not include a final wording diff draft. Here I will
discuss changes built upon the [Reflection TS working draft
N4818](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/n4818.pdf). All
that follows are possible additions to what's being proposed in that draft.

In short, the proposed reflection API works by querying reflection information
of any supported expression through the `reflexpr` operator. This `reflexpr`
call returns a type, called a *"meta-object type"*, which contains all
reflection metadata of the entity referenced by its operand. For example:

``` cpp
using int_metaobject_type = reflexpr(int);

constexpr auto name = std::experimental::reflect::get_name_v<
    int_metaobject_type>;

fmt::print("\"{}\"", name); // prints "int"
```

So the Reflection TS has two sides: A new language feature, the `reflexpr`
operator, and a library of meta-object types and property querying traits. Note
that **the meta-object types are not specified** but the TS instead specifies
a library of [concepts](https://en.cppreference.com/w/cpp/language/constraints)
describing the properties and rules a type must implement to be considered
a meta-object type of a given entity. This, in addition to trait-based queries,
**allows for flexible implementation oportunities**. 

## 4. Minimal attributes support for the Reflection TS

> To follow this section I recomend having [N4818 section 21.12.2
> [reflect.synopsis]](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/n4818.pdf)
> around. This section will do a lot of references to the concepts and traits
> specified there.
>
> Also note most (all) examples assume we are working with the
> `std::experimental::reflect` namespace.

To support attributes (Both built-in and user-defined) we first need
a meta-object type that will contain the information of an attribute. Let's
define a new meta-object concept named `Attribute` to represent this:

``` cpp
template<class T> concept Attribute = ...; // Refines Object
```

The simplest information we could store and query about an attribute is
its full attribute string: Given an attribute `[[hello]]` return the
string `"hello"` and for a more complex attribute like
`[[mylib::attribute("arg")]]` (which includes a namespace and attribute
args) return `"mylib::attribute(\"arg\")"`. We will call the corresponding
trait `get_attribute`:

``` cpp
template<Attribute T> struct get_attribute;
template<Attribute T> constexpr auto get_attribute_v = 
    get_attribute<T>::value;
```

The goal is not to reflect attributes directly, like
`reflexpr([[noreturn]])`, but to know if a given entity has an specific
attribute, or list the set of attributes applied to an entity. To
implement this we need a trait that returns a sequence of `Attribute`s for
the given entity. Let's call this trait `get_attributes`:

``` cpp
template<Attribute T> struct get_attributes; // Returns ObjectSequence
template<Attribute T> using get_attributes_t =
    typename get_attributes<T>::type;
```

We could use it as follows:

``` cpp
[[nodiscard]] int f();

using f_t = reflexpr(f);
using f_attributes = get_attributes_t<f_t>;

static_assert(
    (get_size_v<f_attributes> > 0) && 
     get_attribute_v<get_element_t<0, f_attributes>> == "nodiscard");

template<Function T> concept NoDiscardFunction =
    (get_size_v<get_attributes_t<T>> > 0) &&
    get_attribute_v<get_element_t<0, get_attributes_t<T>>> == "nodiscard";
```

## 5. Minimal attribute parsing support

While the minimal attribute support works, most use cases of user defined
attributes involve some kind of parsing. For example, how do you
differentiate if an attribute is built-in or user defined? The quick and
dirty way would be to parse the attribute to see if it contains
a namespace qualifier.

While we could delegate all parsing to the user, ideally we would like
this parsing to be done at compile time and to be as simple as possible
(i.e. with as few template tricks as possible).

### Name, full name, and display name

If we stick to naming the attribute only, there are four different ways we
could want to name an attribute:

 - By its full string, i.e. `R"(tinyrefl::rename("AwesomeClass"))"`
 - By its non-qualified name plus arguments, i.e.
 `R"(rename("AwesomeClass"))"`
 - By its full qualified name, i.e. `"tinyrefl::rename"`
 - By its non-qualified name, i.e. `"rename"`

> Note all but the first could be ambiguous

[tinyrefl 0.5.0 (wip)](https://github.com/Manu343726/tinyrefl/tree/refactoring-api) implements a base [`Entity`](https://github.com/Manu343726/tinyrefl/blob/refactoring-api/include/tinyrefl/entities/entity.hpp) meta-object class which
among other things introduces a common naming interface for all reflected
entities (attributes included): There's a `full_display_name()`,
a `display_name()`, a `full_name()`, and a `name()`, which correspond to
the four naming criteria described above. However, only the full display
name is guaranteed to be unique, and **this is the only identifier allowed
to be used as unique identifier of an entity**.

> In the `unittest` example shown in section "*2. Motivation*", note how
the mocked function is referenced by its full display name.

The Reflection TS includes the concept of a `Named` entity, and implements
two traits to get the name of a `Named` reflected entity:

 - **`get_name(_v)`**: Returns the non qualified name of the entity.
 - **`get_display_name(_v)`**: Returns the non qualified display name of
 the entity.

One option to implement attribute naming for searches would be to make
`Attribute` refine `Named` instead of `Object`, but I think this feels a bit
forced (I think that was not the original purpose of the `Named` concept) and
also note there's no equivalent on the proposed TS for the full qualified
naming alternatives.

Another option is to have attribute-dedicated traits for naming:

``` cpp
template<Attribute T> struct get_attribute_name;
template<Attribute T> constexpr auto get_attribute_name_v =
    get_attribute_name<T>::value;

template<Attribute T> struct get_attribute_display_name;
template<Attribute T> constexpr auto get_attribute_display_name_v =
    get_attribute_display_name<T>::value;

template<Attribute T> struct get_attribute_full_name;
template<Attribute T> constexpr auto get_attribute_full_name_v =
    get_attribute_full_name<T>::value;

template<Attribute T> struct get_attribute_full_display_name;
template<Attribute T> constexpr auto get_attribute_full_display_name_v =
    get_attribute_full_display_name<T>::value;
```

> I still think having limited naming support will be a problem even for
non attribute entities, and that implementing all the four alternatives
for `Named` must be considered.

### Attribute namespace

In adition to attribute identifiers, in some cases users want to classify
attributes depending on which API they belong to. For example:

``` cpp
template<Attribute T> concept OmpAttribute =
    get_attribute_namespace_v<T> == "omp";
```

To do so the extended attribute support would provide the
`get_attribute_namespace` trait that returns the namespace of the
attribute if it is a user defined attribute, or an empty string if the
attribute is built in:

``` cpp
template<Attribute T> struct get_attribute_namespace;
template<Attribute T> constexpr auto get_attribute_namespace_v =
    get_attribute_namespace<T>::value;
```

Now we can finally check if the attribute is built-in or user defined:

``` cpp
template<Attribute T> concept BuiltInAttribute =
    get_attribute_namespace_v<T>.empty();

template<Attribute T> concept UserDefinedAttribute = !BuiltInAttribute<T>;
```

> Note I'm assuming the trait is returning a `std::string_view` instance.
This would be the optimal choice since `std::string_view` is already
`constexpr` enabled and it provides a useful `std::string` like interface
(Useful for parsing, searching, etc). Also, it makes sense that
`Attribute` components are returned as views to parts of the full
`get_attribute_full_display_name` string.

### Attribute arguments

To help process attribute arguments, we define the trait
`get_attribute_arguments_string` that returns the string containing only
the attribute arguments, commas between args included. **The enclosing
parens are not included**:

``` cpp
template<Attribute T> struct get_attribute_arguments_string;
template<Attribute T> constexpr auto get_attribute_arguments_string =
    get_attribute_arguments_string<T>::value;
```

### Argument tokenization

To help users even more, we could implement lexer support of attribute
arguments and return them already processed as a sequence of tokens. To do
so, let's implement a `AttributeArgumentToken` concept hierarchy as
follows:

``` cpp
template<class T> concept AttributeArgumentToken = ...; // Refines Object
    // and represents an attribute argument token

template<AttributeArgumentToken T> concept AttributeArgumentInteger = ...;
template<AttributeArgumentToken T> concept AttributeArgumentFloat = ...;
template<AttributeArgumentToken T> concept AttributeArgumentBoolean = ...;
// And so on...
...
```

> See [here]() for an hyperlinked BNF grammar of the C++11 attribute
specifier sequence. See
[`attribute-argument-clause`](https://www.nongnu.org/hcb/#attribute-argument-clause)
for the set of tokens supported as attribute arguments.

Now, for any `AttributeArgumentToken` we provide two traits:

 - `get_attribute_argument_token_string`: Returns the string
 representation of the token, i.e. `"\"hello\""`, `"42"`, `"true"`, etc.
 - `get_attribute_token_value`: Returns the parsed value of the token if
 the token is a literal: `"hello"`, `42`, `true`. If the token is not
 a literal it should be referencing an existing entity (A variable, for
 example). If that's the case, return an `Alias` to the referenced entity.

``` cpp
template<AttributeArgumentToken T> struct get_attribute_token_string;
template<AttributeArgumentToken T> constexpr auto get_attribute_token_string_v =
    get_attribute_token_string<T>::value;

template<AttributeArgumentToken T> struct get_attribute_token_value;
template<AttributeArgumentToken T> constexpr auto get_attribute_token_value_v =
    get_attribute_token_value<T>::value;
```

Finally, provide a trait that returns the sequence of tokenized arguments:

``` cpp
// Returns ObjectSequence
template<Attribute T> struct get_attribute_tokenized_arguments;
template<Attribute T> using get_attribute_tokenized_arguments_t =
    get_attribute_tokenized_arguments<T>::type;
```

## 6. Future

### Attribute search API

On top of the low level attribute and sequence traits high level functions and
traits can be implemented to cover common use cases of matching, searching, and
checking attributes:

``` cpp
template<class MetaObject, auto Id> struct has_attribute;
template<class MetaObject, auto Id> constexpr bool has_attribute_v =
    has_attribute<MetaObject, Id>::value;

template<class MetaObject, auto Id> struct get_attribute_by_name;
template<class MetaObject, auto Id> constexpr bool get_attribute_by_name_t =
    typename get_attribute_by_name<MetaObject, Id>::type;
```

Using them as follows:

``` cpp
class [[tinyrefl::ignore]] InternalClass { ... };

static_assert(has_attribute_v<reflexpr(InternalClass), "tinyrefl::ignore">);
static_assert(
    get_size_v<
        get_attribute_tokenized_arguments_t<
            get_attribute_by_name_t<reflexpr(InternalClass), "tinyrefl::ignore">
        >
    > == 0);
```

### Full attribute parsing API

The previous section only describes minimal argument tokenization features for
attributes, ignoring full parsing of them and their arguments.  

From my experience working with [`libclang` and
`clangAST`](https://clang.llvm.org/docs/Tooling.html) I would say current C++
compiler frontends do not parse attributes as part of their AST. Actually,
[cppast](https://github.com/foonathan/cppast) (A C++ libclang wrapper),
implements custom tokenization of C++ attributes since **those are not exposed
to the `clang` AST API nor the `libclang` one**.

> `libclang` is a stable wrapper
over `clangAST`, so it makes sense that if `clangAST` does not expose
attributes neither does `libclang`.

However, this could be fixed from the C++ side at the expense of "some"
compile-time performance. [CTRE](https://compile-time.re/) has demonstrated
that building a performant compile-time lexer and parser generator is possible,
so maybe a pure library implementation of attribute parsing is possible with
APIs similar to CTRE.

### Attributes as classes

With full attribute parsing we could go one more step forward and implement
attribute classes **as a full library feature**.

First we declare a class that will act as attribute:

``` cpp

namespace math
{

class [[attribute]] range
{
public:
    constexpr range(const float begin, const float end);

    constexpr bool between(const float x) const;
};

} // namespace math
```

Use it to tag an entity, say a class member variable:

``` cpp
struct Point
{
    [[math::range(0.0f, 1.0f)]]
    float x;  

    [[math::range(0.0f, 1.0f)]]
    float y;  
};
```

And the reflection system parses the attribute, finds a matching attribute
class constructor among all the reflected entities in the translation unit, and
returns a `constexpr` instance of the attribute class as `get_attribute` value
instead of a string. See [this
post](https://manu343726.github.io/2019-04-18-more-fun-with-user-defined-attributes/)
for details.
