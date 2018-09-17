---
layout: post
title: "High Level Template Metaprogramming, Step by Step: Simple Expressions"
modified:
categories: C++
excerpt:
tags: [C++,template-meta-programming,biicode]
image:
  feature:
date: 2014-10-27T11:52:35+01:00
---

I you are one of who have been following our [post series](http://blog.biicode.com/template-metaprogramming-cpp-ii/) about template metaprogramming with modern C++, at this time you should have become a C++ template Guru. At least thats what I expect ;).   

You know about class templates, function templates, value parameters, type parameters, variadic templates... Your template metaprogramming toolbox is full of great things to play with. Thats good, but you want to start playing with your compiler, writting some cool metaprograms. 

Lets start the game!

*"Good"* Old template metaprogramming
-------------------------------------

In the old days of C++98/03 there were no variadic templates, no template aliases, no `std::enable_if`. Metaprogramming with C++ was a hard and ugly task.  
But it was a *neccesary* task. Most of the time library implementers used template metaprogramming to parametrize and automatize code generation for the library, instead of writting multiple duplicates or derivatives of the same code just to cover all the cases. This was, and it is, a common practice eve
n for Standard Library vendors.  
Template metaprogramming was used to improve perfomance on high-computing libraries too, with some clever code transformations done thanks to tmp. The best example of this is the [blitz++ library](http://en.wikipedia.org/wiki/Blitz%2B%2B), one of the first examples of a real use case of template metaprogramming.

But such codebases where hard to read and maintain, so for most common C++ programmers tmp was just *"crazy stuff for nerds"*.


Since C++11 the language has evolved to support some ways of metaprogramming as a common and useful thing. Metaprogramming become a first class citizen in C++, instead of the obscure, magical, and freaking way to abuse the compiler it was at the beginning.   
Look at the `<type_traits>` header. It provides the so called `type traits`, class templates designed to provide some useful information about a given type.

{% highlight cpp %}

#include <type_traits>

constexpr bool is_ptr = std::is_pointer<int*>::value;

{% endhighlight %}

Language features like `static_cast` and variadic templates help a lot too when doing tmp. 

But the syntax is still too ugly. Consider the implementation of `std::decay`:

{% highlight cpp%}

template< class T >
struct decay {
    typedef typename std::remove_reference<T>::type U;
    typedef typename std::conditional< 
        std::is_array<U>::value,
        typename std::remove_extent<U>::type*,
        typename std::conditional< 
            std::is_function<U>::value,
            typename std::add_pointer<U>::type,
            typename std::remove_cv<U>::type
        >::type
    >::type type;
};

{% endhighlight %}

Too many nested `typename`s, its hard to follow and undertand the code. 

**Could that syntax be improved?** I think thats possible, and thats exactly what we will learn today.

Lets get simpler
----------------

As we seen in our introduction to tmp, the C++ template system can be seen as a pure functional language. In that way, suppose that you are working with a weird version of Haskell.   
There are no templates, there are no types. You have expressions and values. Expressions and values that you can evaluate, manipulate, etc. Call this abstraction *"The Haskell Metaphore"*.

In our functional language, a C++ type is really a value we work with. And templates are expressions that take values (C++ types) as parameters:

{% highlight cpp %}

template<typename T>
using identity = T;

using i = identity<int>;

{% endhighlight %}

In the example above, the `identity` alias is a *metafunction* in our Haskell Metaphore: Takes a value and returns it. The alias `i` is only a value with a name, consider it a *(meta)variable*. 

To get this metaphore simpler, our metafunctions will get C++ type parameters only. If you need to pass a C++ value, use boxing though `std::integral_constant`:

{% highlight cpp %}

using one = identity<std::integral_constant<int,1>>;

{% endhighlight %}

Play with simple expressions
----------------------------

Here is and example of our *Haskell Metaphore*, using my Turbo metaprogramming library for C++11. Turbo is designed to be used trough biicode, which makes Turbo completely platform independent and easy to use. For most of the cases, just include `<manu343726/turbo_core/turbo_core.hpp>` and you are ready:

{% highlight cpp %}

#include <manu343726/turbo_core/turbo_core.hpp>


using one = tml::Int<1>;
using two = tml::Int<2>;

using three = tml::eval<tml::add<one,two>>;

int main()
{
    std::cout << tml::to_runtime<three>() << std::endl;    
}

{% endhighlight %}

Lets examine the example step by step:

 - `tml::Int` declares an integer value. Is just and alias to `std::integral_constant<int>`.

 - `tml::add` is a metafunction to perform addition.

 - `tml::eval` is the magic wand of Turbo. It takes any expression and evaluates it. The addition expression in this case.

 - `tml::to_runtime` is the *bridge* between the compile-time and the runtime world. This function templates takes a type (A value in the Haskell metaphore) and returns the C++ equivalent value. This work is done completely at runtime and has zero runtime overhead.

After compiling (Running the metaprogram), you can run the resulting C++ program:

{% highlight bash %}
$ bii cpp:build
$ ./bin/example
3
{% endhighlight %}

Thats all! Simple, isn't? Here's `std::decay` in the Turbo way:

{% highlight cpp%}

template< class T >
struct decay {
    using U = tml::eval<std::remove_reference<T>>;
    
    using type = tml::eval<
                 tml::conditional<
                                  std::is_array<U>,
                                  tml::eval<std::remove_extent<U>>*,
                                  tml::conditional< 
                                                   std::is_function<U>,
                                                   std::add_pointer<U>,
                                                   std::remove_cv<U>
                                                  >
                                 >
                          >;
};

{% endhighlight %}

I found it much more readable. What do you think?

Summary
-------

 Today we have seen that treating tmp as a functional language is a simpler way to do metaprogramming. We introduced what we call *The Haskell Metaphore* as a way to see and work with metaprogramms: Look at that code as Haskell, instead of the C++ type system.

 In the next posts we will see more complex examples of this metaphore, with high level metaprogramming constructions such as high-order metafunctions, lambda expressions, etc. Stay tuned!