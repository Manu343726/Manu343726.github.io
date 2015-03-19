---
layout: post
title: "Haskellizing C++ template metaprogramming"
modified:
categories: C++
excerpt:
tags: [C++, metaprogramming]
image:
  feature:
date: 2015-03-19T11:25:20+01:00
comments: true
---

If you followed my crazy tales about [the Haskell Metaphor](http://isocpp.org/blog/2014/11/metaprogramming-with-modern-c-the-haskell-metaphor) you may think I have a great problem with myself and the way I waste my free time...  
I have been doing intensive C++ template metaprogramming for up to three years, developing my own tmp library called [Turbo](https://github.com/Manu343726/Turbo), with the idea of providing high-level facilities for C++ metaprogramming. 

You all know, the syntax is horrible, the compiler error messages are not better. But there's still a lot of people doing tmp. Why? Generic programming, libraries, etc. But that's another story. *Someone suggested me to submit a talk to Meeting C++ 2015 on the topic, stay tuned :)*

The point is that we use TMP. We have to deal with it. Great projects like Eric Niebler's Meta or Louis Idionne Boost.Hana reflect the rising effort on the C++ community to have true support of tmp, with standard libraries only for tmp. This is a big step forward, but these libs tend to treat TMP as C++, relying on C++ naming conventions, idioms, etc.  
I don't think TMP is C++. TMP is a functional language. Ugly syntax, but just a Turing complete functional language. I think we will go better if we treat it as a functional language and we apply directly idioms, features, and names from functional programming. Let's forget C++ and think in Haskell.


Lazy vs eager
-------------

Since the very beginning of Turbo, I picked an eager approach for metafunctions. As many of you already know, the basis of Turbo is the `tml::eval` metafunction, which is really a kind of monster expression evaluator. 
`eval` is designed to take any expression Turbo is capable of dealing with and return its value. For example, in the old days of tmp, think of Boost.MPL, you have to do this:

{% highlight cpp %}
using int_t = typename std::remove_const<
                  typename std::remove_pointer<const int*>::type
              >::type;
{% endhighlight %}

With `tml::eval` is as simple as:

{% highlight cpp %}
using int_t = tml::eval<std::remove_pointer<std::remove_const<const int*>>>;
{% endhighlight %}

In other words, Turbo provides support for **true metafunction composition**.   

The problem with this approach is that `eval` does eager expression evaluation. There's no place for lazy evaluation since `eval` processes the expression as is. This generates some issues, since the expression (The template which represents the expression) should be instantiable in two different ways, one during expression instantiation and other during expression evaluation (Substitution of expression arguments with their values).  

Eric Niebler took a better approach on Meta. Meta is eager as Turbo, but it doesn't have the classic concept of metafunction as a template. Instead Meta uses template aliases whenever possible, providing a simpler version of `eval` to extract the `::type` from old fashioned metafunctions like STL type traits.  
No templates, no composition problems and no double instantiation issues. Simple.

Despite the differences, both ways suffer from the same evaluation problem when dealing with lazyness or deferred evaluation. Consider lambda expressions:

{% highlight cpp %}
using f = lambda<_1, eval<std::decay<_1>>>;
{% endhighlight %}

The `f` function above doesn't do what we want, since `eval` does direct evaluation. We don't want to evaluate the lambda body until its parameters are substituted with their values, until lambda evaluation.
Both Eric and I come to the same solution: Place a fake `eval`-like expression whenever you need evaluation on non-eager contexts. I call it `tml::deval`:

{% highlight cpp %}
using f = lambda<_1, tml::deval<std::decay<_1>>>;
{% endhighlight %}

Even if the interface is almost the same, Meta and Turbo lambda implementations are very different. Mine is rather complex, a lambda is really a `let` expression with the lambda parameters bounded on it. `let` is applied when the lambda is evaluated. The deferred eval hack was just another specialization on the `let` implementation to take care of that "tag", substituting it with `tml::eval`.  
I was looking mindfully along all Meta development, acting as a vulture flying around Eric's work... I had most of Turbo features working almost a year before he announced Meta, so I felt it was a great opportunity to compare my work with one of the top C++ gurus. The day he released lambda expressions I though *"Great!, let's look how he solved the problem..."* and then *"Oh, fuck, his lambdas are exactly the same of mine. He didn't solve this neither :("*

Whenever we want to play lazy, we have to enter a custom entity to deal with it. 
Another problem when representing functions as templates/template aliases, is that functions are not first class citizens by default. You cannot store a template as a typedef, nor passing it as parameter directly. *Forget template-template parameters, metafunctions usually work with type parameters only to make things easier*. 

{% highlight cpp %}
template<typename T>
struct identity
{
    using type = T;
};

l1 = map<identity, list<int,char,bool>>;            //ERROR: map expects type
l2 = map<identity<_>, list<int,char,bool>>;         //OK, _ is a placeholder
l3 = map<tml::lazy<identity>, list<int,char,bool>>; //OK, lazy wraps templates
{% endhighlight %}

Even if `meta::defer`/`tml::lazy` work, these are not the best way to deal with this problem.  **We really have to wrap every metafunction just to be able to pass it to another context? We really have to add another layer just to apply lazy/deferred evaluation?** I'm not that sure, should be better ways to play with this.

Metafunction classes
--------------------

Boost.MPL solved the problem many years ago through *metafunction classes*. A metafunction class is just another representation of a metafunction: Despite getting the result directly via a `::type` member and passing function parameters as template parameters, metafunction classes hold the computation on a nested template-based metafunction, usually called `apply` by convention, **actually decoupling function evaluation from instantiation**.   

This is the identity function above in the metafunction class way:

{% highlight cpp %}
struct identity
{
    template<typename T>
    struct apply
    {
        using type = T;
    };
};
{% endhighlight %}

Note the difference: **Now `identity` is a type instead of a template**, a first class citizen in our TMP toolbox which we can pass to other templates, store as a typedef, etc. 

The last month I have been working hard for a week to add metafunction classes support for `tml::eval`. Now Turbo is able to evaluate metafunction classes too, which makes `eval` even more powerful:

{% highlight cpp %}
struct identity
{
    template<typename T>
    struct apply
    {
        using type = T;
    };
};

using l1 = map<identity, list<int,char,bool>>; //OK, identity is a mc
{% endhighlight %}

And that power is not only a matter of being a type. Since we are not using template parameters as function parameters anymore, we can use those for whatever we want. For example, `tml::lazy` was rewritten as a metafunction class:

{% highlight cpp %}
template<template<typename...> class F>
struct lazy
{
    template<typename... Args>
    struct apply
    {
        using type = tml::eval<F<Args...>>;
    };
};

using remove_reference = lazy<remove_reference>;
using int_t = tml::eval<remove_reference, int&>;
{% endhighlight %}

Also you can bound complex metafunction implementation inside its definition, that is, partially specialize the `apply` instead of polluting the namespace:

{% highlight cpp %}
struct remove_pointers
{
    template<typename T>
    struct apply
    {
        using type = T;
    };

    template<typename T>
    struct apply<T*>
    {
        using type = T;
    };

    template<typename... Ts>
    struct apply<std::tuple<Ts...>>
    {
        using type = std::tuple<tml::eval<remove_pointers, Ts>...>;
    };

    template<typename T, std::size_t N>
    struct apply<T[N]>
    {
        using type = tml::eval<remove_pointers, T>[N];
    };

    ...
};
{% endhighlight %}

Since the computation is not done by the type/template itself but the nested `apply` metafunction, **any type with an `apply` nested metafunction is considered a metafunction class, therebefore callable**.

Consider the usual hello world for modern template metaprogramming, the variadic typelist template:

{% highlight cpp %}
template<typename... Ts>
struct list {};
{% endhighlight %}

We can easily hold multiple types on that structure, traverse the list recursively, etc. Useful, but a bit boring.   
What if lists were callable?

{% highlight cpp %}
template<typename... Ts>
struct list
{
    template<typename Function>
    struct apply
    {
        using type = list<tml::eval<Function, Ts>...>;
    };
};

using input    = list<int,char,bool>;
using function = tml::lazy<std::add_pointer>;
using output   = tml::eval<input, function>;

static_assert(std::is_same<output, list<int*, char*, bool*>>::value, "Cool");
{% endhighlight %}

That was just a mappable list, a list that belongs to the Functor typeclass, following Haskell terminology. What about a foldable list?

{% highlight cpp %}
template<typename... Ts>
struct foldable
{
    template<typename F, typename E>
    struct apply
    {
        template<typename S, typename... Args>
        struct foldl;

        template<typename S, typename Head, typename... Tail>
        struct foldl<S,Head,Tail...>
        {
            using type = typename foldl<tml::eval<F,S,Head>,Tail...>::type;
        };

        template<typename S>
        struct foldl<S>
        {
            using type = S;
        };

        using type = typename foldl<E,Ts...>::type;
    };
};
{% endhighlight %} 

With a foldable thing we can write a simple continuation type in a couple of lines:

{% highlight cpp %}
template<typename... Fs>
struct Continuation
{
    struct Fn{};
    struct State{};

    template<typename Start>
    struct apply
    {
        using list = foldable_list<Fs...>;
        using type = tml::eval<list, tml::lambda<State,Fn, tml::deval<Fn,State>>, Start>;
    };
};
{% endhighlight %}

And here's the crown jewel: Do-like notation for tmp:

{% highlight cpp %}
template<typename Start, template<typename...>class... Fs>
using Do = tml::eval<Continuation<tml::lazy<Fs>...>, Start>;

using int_t = Do<const int[10]&,
                 std::remove_const,
                 std::remove_reference,
                 std::decay,
                 std::remove_pointer
                >;
{% endhighlight %} 

Type classes
------------

As you may noticed, I'm not a big fan of partial template specialization and traits-based constraints. The same for tag dispatching. Of course they work great, but pollute the library namespace with sparse partial specializations and related code that make the code even more hard to read. *Maintainance first, even in funadic metaprogramming!*  
On the metafunction classes part we seen how features can be bound to the type they belong to. Let's extend that to a new level: Let's try to mimic Haskell type classes. 

To simplify the process, consider a *type class* a set of features related to a set of types. A type `T` is considered part to a type class `C` if `T` has the features that identify `C`. By features I usually mean metafunctions.   

In that way, the most simple typeclass that comes to my mind is [Functor](https://wiki.haskell.org/Functor). Functor identifies those types that are mappable, that is, that have a working `fmap` function applicable on them. This is how `Functor` may look like in TMP:

{% highlight cpp %}
template<typename T, typename F>
using fmap = tml::eval<typename T::fmap, F>;
{% endhighlight %}

Given the definition of type class above, any type `T` part of the `Functor` class has a `fmap` function. Then, the global `fmap` metafunction is applicable, **and looks like a normal metafunction**, for any `T` functor:

{% highlight cpp %}
template<typename... Ts>
struct list
{
    struct fmap
    {
        template<typename F>
        struct apply
        {
            using type = list<tml::eval<F, Ts>...>;
        };
    };
};

using ptrs = fmap<list<int,char,bool>, tml::lazy<std::add_pointer>>;
{% endhighlight %}

Functor was simple. What about Monad? Again, let's get simple and say something is a monad if it has `Bind` and `Return` metafunctions:

{% highlight cpp %}
template<typename A, typename B>
using Bind = tml::eval<typename A::Bind, A, B>;

template<typename T>
using Return = tml::eval<typename T::Return, T>;
{% endhighlight %}

 This is `Maybe` monad:

{% highlight cpp %}
template<typename T>
struct Just;

struct Nothing;

struct Maybe
{
    struct Return
    {
        template<typename U>
        struct apply
        {
            using type = Just<U>;
        };
    };

    struct Bind
    {
        template<typename M, typename G>
        struct apply;

        template<typename G>
        struct apply<Nothing, G>
        {
            using type = Nothing;
        };

        template<typename M, typename G>
        struct apply<Just<M>, G>
        {
            using type = tml::eval<G, M>;
        };
    };
};

template<typename T>
struct Just : public Maybe
{};

struct Nothing : public Maybe
{};
{% endhighlight %}

The forward declarations of `Just` and `Nothing` are needed since there's no easy way to simulate Haskell algebraic datatypes. Instead, we define `Maybe` using `Just` and `Nothing` inside its implementation, then both become part of `Maybe` typeclass via inheritance.

Here's an example of Maybe monad usage:

{% highlight cpp %}
struct remove_stars
{
    template<typename T>
    struct apply
    {
        using type = Just<T>;
    };

    template<typename T>
    struct apply<T*>
    {
        using type = Just<T>;
    };

    template<typename T, std::size_t N>
    struct apply<T[N]>
    {
        using type = Nothing;
    };
};

//binary function for fold (Sequence element is ignored)
struct run
{
    template<typename Current, typename Elem>
    struct apply
    {
        using type = Bind<Current, remove_stars>;
    };
};

using array_t = int[10];

//Run 10 times
using result = tml::apply_for<run, Just<int*****>, tml::size_t<0>, tml::size_t<10>>;
using error  = tml::apply_for<run, Just<array_t****>, tml::size_t<0>, tml::size_t<10>>;

int main()
{
    std::cout << tml::to_string<result>() << std::endl;
    std::cout << tml::to_string<error>() << std::endl;
}
{% endhighlight %}

The result is:

{% highlight text %} 
Just<int>
Nothing
{% endhighlight %}

More haskell?
-------------

Of course there are more things we can do. My point is that if we focus on the functional spirit of TMP, instead of thinking on templates, writing metaprograms can be easy, even funny, for any programmer with minimal experience in functional programming.

*As usually, I'm not a Haskell expert so far, please feel free to write a comment if there are mistakes. Thanks!*

> Written with [StackEdit](https://stackedit.io/).

