---
published: false
---

Future Template Metaprogramming
=============================

Let's be clear: I do C++ mostly for fun. Some days I try to dust off my maths and graphics knowledge to write a software raster. Some other days I "just" try to write a polymorphism-ready allocator. *Oh God, the C++ allocator interface is so creepy, kudos to anyone who ever tried to write one (Hey [Jonathan](https://github.com/foonathan/memory) :P)*.
But most of the time I try to catch bugs in my compiler, diving into the dark side of C++ where programs are not run but compiled...

*"Oh guy, just get this madness out and buy a haskell live interpreter"*. Nah, that sounds fairly easy. Instead, I have been working for two years to get the ideal workflow for doing metaprogramming with C++. [We all agree](http://pdimov.com/cpp2/simple_cxx11_metaprogramming.html) that modern C++ features like variadic templates and template aliases completely dominated this field and changed the way we do meta with C++. No more macros, `<type_traits>`, `static_assert()`. The average C++ programmer is now able to do most of the simple meta tasks without melding his/her brain in the process. *Maybe later, when the compiler gives its diagnostics*.

I'm mostly glad with the current state of the art, but there are a few points that still puzzle me:

 - **Eager vs lazy?**: Eager tmp is simple and composable, but what about conditionals? What about lambdas?
 - **Metafunction based?**: Looking at Hana, I'm no more that sure.
 - **(Learn LISP?)**: For true meta-programmers since the 60's...

I suggest you to enter Boost mailing list or the issues pages of the most relevant tmp libraries. There's a lot of debate on such topics. But I want to go one step forward: **How metaprogramming will look like in the next years?** This is what I want to see in the near future.

## Concepts Lite

I have been working with [Concepts Lite]() for a while. This feature and the proposals that come from it focus on the Standard Library: How STL algorithms should look like when that concepts and preconditions are really checked by the language instead of being just a "documentation detail". Despite Concepts Lite proposal discard powerful features like *"concept maps"* defined by the original C++0x proposal, I think CL still has the power to give a semantic-rich and composable type system, similar to the Haskell one.

For me many programming languages, C++ included, made the mistake of thinking in types as mere "tags", names: An entity belongs to a type because its tagged in that way. Then many tools raised to add tag-specific behavior to entities: Function overloading, generics/templates, etc. But the worst approach in my opinion is what most OO languages do: To give behavior to an entity, you should implement the interface that provides that behavior. Your entity is `Addable` since it implements the `addable` interface. 

Forget coupling, forget class hierarchies. I'm not talking about that kind of problems. I'm talking about the *philosophy of the type system*. Shouldn't it be the other way around? **I say the type `int` is `Addable` since I can actually add two ints**.

``` cpp
template<typename T, typename = void>
struct is_addable : std::false_type {};

template<typename T>
struct is_addable<T, void_t<decltype(std::declval<T>() + std::declval<T>())>> :
	std::true_type
{};
```

*I strongly recommend [Tick](https://github.com/pfultz2/Tick) library for trait and concept definition.*

I find more useful to think about the features a type gives to me instead of the concrete type itself.  The "*Type Class*" using Haskell terms. C++ had this before Concepts: Template-based static duck typing or `auto` are examples of this *"expect behavior, not type name"* way of thinking.
The beauty of Concepts Lite is the ability **to declare** the set of rules that identify certain set of types, then **constraint** types that follow that rules in a simple way. Using [gcc-clite](http://concepts.axiomatics.org/~ans/) syntax:

``` cpp
template<typename T>
constexpr bool Addable()
{
	return is_addable<T>::value;
}

template<Addable T>
auto dup(const T& e)
{
    return e + e;
}
```

That is, instead of adding types to a "family" (i.e. class hierarchy), you classify types by their features in a non-intrusive way. Types are completely free and unrelated, then some rules are provided to classify types using the criteria you want. *Note that with CL you cannot add a type to a typeclass/concept in a explicit manner. That was what concept maps were partly about. However, you can still simulate them using good old tag dispatching, type traits, etc:*

``` cpp
template<typename T>
constexpr bool MyConcept()
{
    return /* T expected characteristics */ &&
           specialize_to_add_T_to_MyConcept<T>::value;
}
```

But, what this have to do with metaprogramming? I have been working the last three months on the design and specification of a port of the Haskell language to C++ metaprogramming. Let's be truly functional when doing TMP. Or at least that's the idea...  
The plan is to develop the basis of a Haskell-like algebraic type system for our metaprograms (actually a meta-type system), then start to work on basic data structures and high order metafunctions. In the long term, a full port of the prelude.

I'm still working following the metafunction paradigm (See Hana metaprogramming bellow), so my main tools are C++ types, metafunctions, and metafunction classes. The first question is: How a type constructor should look like? This is what I ended up with:

``` cpp
struct List
{
    struct Nil
    {
		using type = HaskellType<List, Nil, std::tuple<>>;
	};

	struct Cat
	{
		template<typename Head, Class<List> Tail>
		
	};
};
```




> Written with [StackEdit](https://stackedit.io/).## A New Post

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
