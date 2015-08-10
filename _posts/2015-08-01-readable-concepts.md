---
layout: post
title: "Readable Concepts"
modified:
categories: 
excerpt:
tags: [c++]
image:
  feature:
date: 2015-08-01T02:05:16+02:00
---

Before [the acceptance of Concepts Lite TS into C++17](https://www.reddit.com/r/cpp/comments/3dzv6i/eric_niebler_on_twitter_the_concepts_ts_was_voted/), there was a lot of effort out there in the community to make a C++11/14 compatible implementation of Concepts, at least an emulation wrapping the usual SFINAE tricks.  
Concepts provide nice syntax for concept definition type constraining. No more SFINAE! (I hope). This will make C++ syntax clean in one of the very few points I think C++ still needs a "lifting": templates.   
Concepts Lite also defines the hierarchical organization of different type concepts that had always been with us, but only as [documentation notes](http://en.cppreference.com/w/cpp/concept). 

## Previous work

In this widespread C++ community, taking template meta-programming seriously introduces you into a very little niche of people as-crazy-as-you, currently leaded by what I call "The Gang Of Tmp": Louis Dionne, Paul Fultz II, and Eric Niebler. There are other players that are rising and I would like to see more of their work in the future, like Peter Dimov's series on Simple C++11 Meta-programming, and the amazing work from Filip Roséen on stateful meta-programming.

Anyone else noticed the fun fact that each one of the gang above has implemented his own version of a C++11/14 Concepts library? Fultz has [Tick](https://github.com/pfultz2/Tick), [Boost.Hana](https://github.com/ldionne/hana) has another one built in its internals, Niebler's [range-v3](https://github.com/ericniebler/range-v3) proposal is based heavily on Concept specification. We C++ers really have a serious problem on code sharing and dependency management... 

I have tried all those libs. Tick provides straightforward trait specification and checking, the best alternative for defining traits and concepts in your libraries these days: Just clone Tick into your include directories and you are ready. No dependencies, no headaches. I like range-v3 concepts since Niebler did a lot of work providing a wide set of concepts for the library. Almost any Standard Library concept is already defined there, plus the ones about ranges. Finally is Hana from Dionne: You really guys should try Boost.Hana. Hana is C++ The Right Way. Seriously. Concepts implementation in Hana are a mere implementation detail, but I'm confident Louis is currently working on modularizing the library, so it could be available as a standalone module in the future.

## The problem

One of the greatest gifts given by the Concepts Lite proposal is to reduce template diagnostic boilerplate when a concept is not satisfied: Instead of showing the whole "call stack" of instantiation failures, Concepts Lite will provide a meaningful message at the uppermost level:

{% highlight cpp %}

struct foo
{
	foo() = delete;
	foo(int) {}	
};

void f(const DefaultConstructible& e);

int main()
{
	f(foo{});
}

{% endhighlight %}

> **foo** does not satisfy the **DefaultConstructible** concept  

Compare that to its SFINAE based alternative:

{% highlight cpp %}

template<typename T, 
         typename = typename std::enable_if<std::is_default_constructible<T>::value>::type>
void f(const T& e);

int main()
{
	f(foo{});
}

{% endhighlight %}

> No member named "type" in "std::enable_if<std::is_default_constructible<foo>::value>"  

clang at least identifies this pattern and gives you something similar to "Specialization disabled by enable_if".

So far so good. But consider a more complex concept, one that's the aggregation of multiple properties and/or refines some other concepts. Take for example `TotallyOrdered` concept [from range-v3](https://github.com/ericniebler/range-v3/blob/a9c05cd3f879d6b363d60d26ebfc8c55948acb04/include/range/v3/utility/concepts.hpp#L479):

{%highlight cpp %}
struct TotallyOrdered
  : refines<EqualityComparable, WeaklyOrdered>
{
    template<typename T>
    void requires_(T);

    template<typename T, typename U>
    auto requires_(T t, U u) -> decltype(
        concepts::valid_expr(
            concepts::model_of<TotallyOrdered>(val<T>()),
            concepts::model_of<TotallyOrdered>(val<U>())
        ));
};

struct foo {};

template<typename T, CONCEPT_REQUIRES(TotallyOrdered<T>())>
void f(const T& e);

int main()
{
	f(foo{});
}

{% endhighlight %}

`CONCEPT_REQUIRES()` macro will tell you that `foo` does not satisfy the `TotallyOrdered` concept, **but not why**. This issue was [already noticed](https://www.reddit.com/r/cpp/comments/3g3lsy/c_concepts_support_merged_into_gcc_trunk/ctx4vwt) in the recent addition of Concepts Lite branch into GCC trunk. 

I faced this problem writing a custom range type while playing with range-v3:

{% highlight cpp %}
struct MyRange
{
    struct Iterator{ ... };

    Iterator begin() const;
    Iterator end() const;
};

void odds_only(const MyRange& r)
{
    return r | ranges::view::filter([](int e)
    {
        return i % 2 == 0;
    });
}
{% endhighlight %}

I might have made a mistake with the `Iterator` class since the internal concept checks that range-v3 has said that `MyRange` was not a range, so I couldn't apply a view on it. *`MyRange` doesn't satisfy the "`Range`" concept*. But why? I had to deep dive into the range-v3 concepts hierarchy sources to figure out what was wrong with my iterators.

## When a C++ programmer is frustrated on X, writes yet another X library

That experience left me a bad taste in the mouth. That wasn't fine, Niebler took a lot of work to get the concepts right, every function and datatype from range-v3 is constrained on the types it works with. But a frustrated user could just throw away range-v3 at first chance if he is not able to figure out the problems that may arise. 

The C++11/14 concepts implementations described above are focused on success, on passing the type across all the concept properties to look if those are fulfilled, grabbing out the result for SFINAEing functions, classes, etc. Those concepts are just functions computing a result, whether the concept was satisfied by a given type or not.   

But what if we design concepts that are not mere sets of properties that should be true, but a bag of meta-information about this properties? 
Since the properties are formulated and checked at compile time, there should be a way to, when checking if a type `T` meets a concept `C`, collect enough information to give a meaningful message to the user about what was wrong. That is, instead of just carrying boolean information about the application of `T` in `C`, also store information about **how `T` acted at each property of `C`**.

All the guys above have their own concepts checking library! I have to write mine too...

## Worm

Worm is the codename of my readable concepts library. *I'm open for name suggestions*. 

It is focused on defining concepts in a declarative way, plus storing information about the requirements of the concept applied to a type.

{% highlight cpp %}
BEGIN_CONCEPT(Allocatable)
    REQUIRES_EXPR_EXPECTED(new T_, T_*)
    REQUIRES_EXPR(delete std::declval<T_*>())
    REQUIRES_EXPR_EXPECTED(new T_[666], T_*)
    REQUIRES_EXPR(delete [] new T_[666])
END_CONCEPT(Allocatable)
{% endhighlight %}

That's the `Allocatable` concept directly translated from range-v3 sources. The preprocessor macros make the code look like ugly COBOL, but hide a lot of sorcery to the user.

Any worm concept has a `::value` boolean member with the result of the instantiation of the concept on a given type or types. Is the member that type traits usually have, the one you use for SFINAE and related things. But in addition, this concepts have a `message` member, **a `constexpr` string with detailed information about the instantiation of the concept**.  
If you print that string, you will see how exactly the type behaved in the concept:

{% highlight cpp %}
int main()
{
    std::cout << Allocatable<int>::message << "\n";
}
{% endhighlight %}

> Allocatable requires:  
> "new T_" of type "T_*" [SUCCEED]  
> "delete std::declval<T_*>()" [SUCCEED]  
> "new T_[666]" of type "T_*" [SUCCEED]  
> "delete [] new T_[666]" [SUCCEED]  

## Defining a concept

Worm concepts start with a "*concept block*", a pair of `BEGIN_CONCEPT()` `END_CONCEPT()` macros with the name of the concept as parameter.

{% highlight cpp %}
BEGIN_CONCEPT(OurFirstConcept)

END_CONCEPT(OurFirstConcept)
{% endhighlight %}

`BEGIN_CONCEPT()` also supports additional extra parameters that are **the existing concepts that may be refined** by our concept. For example, this is the translation of `Regular` concept from range-v3:

{% highlight cpp %}
BEGIN_CONCEPT(Regular, Semiregular<T>, EqualityComparable<T>)
END_CONCEPT(Regular)
{% endhighlight %}

Worm concepts can take any number of types as parameters, and those are exposed in multiple ways to make the concept the most readable possible on each scenario:

 - First parameter: `T`, `First`, `Lhs`, or `Head`. For optimal readability in unary, binary, and n-ary concepts. Concepts without parameters are not valid, so the first parameter is always defined.

 - Second Parameter: `U`, `Second`, `Rhs`. If the concept was instanced with one parameter only, its value is undefined.

 - Rest: `Tail`, a variadic pack packing all parameters except first. `Head` and `Tail` are suited for working with recursive concepts in an easy way. If there is one parameter only, the pack is empty.

 - All: `Ts`, a variadic pack.

As an example, imagine you want to write an n-ary equivalent of `Regular`, a concept that checks if all the types passed are `Regular`:

{% highlight cpp %}
BEGIN_CONCEPT(Regulars, Regular<Head>, Regular<Tail>...)
END_CONCEPT(Regular)
{% endhighlight %}

There's no awkward syntax, just simple variadic pack expansion. Let's see the `message` from an instance of `Regulars`:

{% highlight cpp %}
int main()
{
    std::cout << Regulars<int, char>::message << "\n";
}
{% endhighlight %}

> Regulars requires:  
> While refining (Regular<Head>, Regular<Tail>...) with Regulars:  
> Regular requires:  
> While refining (Semiregular<T>, EqualityComparable<T>) with Regular:  
> Semiregular requires:  
>  "&lvalue<T_>" of type "const T_*" [SUCCEED]  
> While refining (DefaultConstructible<T>, CopyConstructible<T>, Destructible<T> >, CopyAssignable<T>) with Semiregular:  
> DefaultConstructible requires:  
>  "(std::is_default_constructible<Ts...>)" giving "true" [SUCCEED]  
> CopyConstructible requires:  
>  "(std::is_copy_constructible<Ts...>)" giving "true" [SUCCEED]  
> Destructible requires:  
>  "(std::is_destructible<Ts...>)" giving "true" [SUCCEED]  
> CopyAssignable requires:  
>  "(std::is_copy_assignable<Ts...>)" giving "true" [SUCCEED]  
> EqualityComparable requires:  
>  "std::declval<T_>() == std::declval<T_>()" convertible to "bool" [SUCCEED]  
>  "std::declval<T_>() == std::declval<T_>()" convertible to "bool" [SUCCEED]  
> Regular requires:  
> While refining (Semiregular<T>, EqualityComparable<T>) with Regular:  
> Semiregular requires:  
>  "&lvalue<T_>" of type "const T_*" [SUCCEED]  
> While refining (DefaultConstructible<T>, CopyConstructible<T>, Destructible<T>, CopyAssignable<T>) with Semiregular:  
> DefaultConstructible requires:  
>  "(std::is_default_constructible<Ts...>)" giving "true" [SUCCEED]  
> CopyConstructible requires:  
>  "(std::is_copy_constructible<Ts...>)" giving "true" [SUCCEED]  
> Destructible requires:  
>  "(std::is_destructible<Ts...>)" giving "true" [SUCCEED]  
> CopyAssignable requires:  
>  "(std::is_copy_assignable<Ts...>)" giving "true" [SUCCEED]  
> EqualityComparable requires:  
>  "std::declval<T_>() == std::declval<T_>()" convertible to "bool" [SUCCEED]  
>  "std::declval<T_>() == std::declval<T_>()" convertible to "bool" [SUCCEED]  

### Requirements

Now that we have learned how to declare a new concept and make it refine one or more existing ones, it's time to write the requirements for our concept.

Requirements are defined just inside the concept block. For example:

{% highlight cpp %}
BEGIN_CONCEPT(HasFoo)
    REQUIRES_EXPR(std::declval<T_>().foo())
END_CONCEPT(HasFoo)
{% endhighlight %}

That's the `REQUIRES_EXPR()` requirement, which checks if an expression is valid. In this case, it checks if the type has a member function `foo()`. All the type parameters are accessible by requirements, but these have an extra underscore at the end. *Matters of sorcery limitations...*
Any number of requirements is supported inside a concept.

Currently worm provides the following requirements:

 - `REQUIRES_EXPR(<expression>)`: Checks if `<expression>` is valid.
 - `REQUIRES_EXPR_EXPECTED(<expression>, <expected>)`: Checks if `<expression>` is valid and yields a value of type `<expected>`.
 - `REQUIRES_EXPR_CONVERTIBLE(<expression>, <expected>)`: Checks if `<expression>` is valid and yields a value convertible to type `<expected>`.
 - `REQUIRES_TRAIT_EXPECTED(<trait instance>, <expected>)`: Checks if a given type trait/concept instance yields the value (`::value`) `<expected>`.
 - `REQUIRES_TRAIT(<trait instance>)`: Checks if a given type-trait/concept instance is fulfilled. Just an alias of `REQUIRES_TRAIT_EXPECTED(<trait instance>, true)`.
  
### Porting existing type-traits and concepts

Wrapping Standard Library type traits is a tiresome task, so worm also provides a simple utility `CONCEPT_FROM_TRAIT(<name>, <trait>)` to help with it.  
`CONCEPT_FROM_TRAIT()` is a simple macro defined as:

{% highlight cpp %}
#define CONCEPT_FROM_TRAIT(Concept, Trait) \
    BEGIN_CONCEPT(Concept)                 \
      REQUIRES_TRAIT(Trait<Ts...>)         \
    END_CONCEPT(Concept)
{% endhighlight %}

Had you noticed the `Ts...` in the `Regulars` message shown above? That's why all the Standard Library concepts were defined from its equivalent type traits through this macro:

{% highlight cpp %}
CONCEPT_FROM_TRAIT(DefaultConstructible, std::is_default_constructible)
{% endhighlight %}

## Future work

Ignoring the format of the messages, which I'm still not fully satisfied with, there's a point that bothers me: **The message doesn't show the values of the type parameters**. The reason is simple: When I started with worm, I had no way to take a `constexpr` string with the name of a given type. 

That's why I have been working on an independent project, [ctti](https://github.com/Manu343726/ctti). ctti provides compile-time type information similar to what `std::type_info` and `std::type_index` give through RTTI, but at compile time. We have ctti working on GCC, Clang, and Visual Studio 2015, so I hope I could use it in worm in a couple of weeks. This is the kind of message I'm pursuing:

> Regulars requires:  
> While refining (Regular<Head>, Regular<Tail>...) with Regulars: [With Head = int, Tail = (char)]  
> Regular requires:  
> While refining (Semiregular<T>, EqualityComparable<T>) with Regular: [With T = int]  
> Semiregular requires:  
>  "&lvalue<T_>" of type "const T_*" [SUCCEED] [With T_ = int]  
> While refining (DefaultConstructible<T>, CopyConstructible<T>, Destructible<T> >, CopyAssignable<T>) with Semiregular: [With T_ = int]  
> DefaultConstructible requires:  
>  "(std::is_default_constructible<Ts...>)" giving "true" [SUCCEED] [With T_ = int]  

## Next posts

Worm was not released yet, I would like to explain how it works in detail before sharing it with the community. It's all about macros and tricks pushing compiler limits, so I will not show you the codez until I'm sure you (and me xD) understand how it works.