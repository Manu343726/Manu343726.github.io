---
layout: post
title: "Resurce Handles and Value Semantics"
modified:
categories: 
excerpt:
tags: [c++]
image:
  feature:
date: 2015-06-30T02:05:16+02:00
---

Before C++11, implementing correct value semantics for a class was a complete pain: You must deal with the constructor, copy constructor, (copy) assignment operator, and then the destructor. Also you had to take some care since those semantics are tightly coupled between that special member functions, so the compiler may reject to provide a default implementation for any of these if you break the semantics of at least one. As you might know, this rule of thumb is known as [*"The Rule Of Three"*](http://stackoverflow.com/questions/4172722/what-is-the-rule-of-three).

Since C++11 this is event worse. Now you have move semantics, so there are the three ingredients above plus two more, move constructor and move assignment operator, added to the mix. The Rule Of Three [becomes The Rule Of Five](http://stackoverflow.com/questions/4782757/rule-of-three-becomes-rule-of-five-with-c11) in C++11.

Hopefully, the programming world is full of smart and nonconformist guys who always come with a solution. In this case, R. Martinho Fernandes came with the Great Idea of [The Rule Of Zero](http://flamingdangerzone.com/cxx11/rule-of-zero/). In short: Only a few classes should deal with resource management, in most of cases relying on such resource handlers for resource ownership is enough and, in that way, **default semantics given by the compiler work like a charm**.   
As the author notices, the point is that the C++ Standard Library already ships handlers for the two most common cases of resource management in C++: Dynamic memory and ownership management, the former through the [containers library](http://en.cppreference.com/w/cpp/container), the later with smart pointers `std::unique_ptr` and `std::shared_ptr`. We have reach the point that raw `new/delete` are not needed any more, even at the point [I think they should be deprecated in the long term](https://www.reddit.com/r/cpp/comments/3b4qx7/whats_the_deal_with_wt_why_doesnt_it_have_more/cske5mx) (But that's another story...).

Many of us adopted The Rule Of Zero as the main way to do C++ classes, in a way you might never deal with destructors and all that stuff except in some special circumstances (aka [tricks](https://en.wikipedia.org/wiki/Expression_templates) ;). But sadly **this model is flawed**.

## Non-RAII to RAII resources: Standard Library version

Consider the most usual case: You love RAII and the magic from destructors and `}`, but there's no handler in the Standard Library to directly deal with your resource; say a handle from a C API. Even fighting with a C API, you expect some RAII sorcery at the C++ side:

{% highlight cpp %}
class Surface
{

private:
	SDL_Surface* srf_;
}
{% endhighlight %}

*"Great, I know about the Rule Of Zero"* you may think. Wrap the handle with a `std::shared_ptr` and a custom deleter and you are done.

{% highlight cpp %}
std::shared_ptr<SDL_Surface>(SDL_LoadBMP(....), SDL_FreeSurface);
{% endhighlight %}

That could be the case for this example, I don't expect expensive resources like surfaces to be easily copyable but created on demand and shared across the engine. But what if the resource should be copyable? That's the flaw: **`std::shared_ptr` does not do value semantics but sharing semantics instead**. 

That's the point: From the two alternatives provided by the Standard Library, one is non-copyable, and the other does sharing semantics. There's a common practice of going ahead and picking `std::shared_ptr` *just to make my class copy* without realizing the semantics that will give to your objects.  
The answer from the Standard Library is: Stop programming by Rules and write all the member functions, or waste your time writing a value-semantics adaptor.

I love time wasting.

## value_wrapper

As usually, I'm not a proficient person at naming things: There's a `std::reference_wrapper` that wraps a reference to make it copyable, and I want to wrap something to make it look like a value. That "simple" reasoning is what my brain did to pick the name.

What's `value_wrapper` ? A template that, given a handle type and a description of the required value semantics, builds a wrapper around the handle that makes it look like a value following the semantics.

{% highlight cpp %}
template<typename Semantics>
struct value_wrapper
{
    using value_type = typename Semantics::value_type;
    using handle_type = typename Semantics::handle_type;

    template<typename... Args>
    value_wrapper(Args&&... args) :
        handle_{semantics_.construct(std::forward<Args>(args)...)}
    {}

    template<typename... Args>
    value_wrapper(const Semantics& semantics, Args&&... args) :
        semantics_{semantics},
        handle_{semantics_.construct(std::forward<Args>(args)...)}
    {}

    value_wrapper(const value_wrapper& v) :
        semantics_{v.semantics_},
        handle_{semantics_.copy(v.handle_)}
    {}

    value_wrapper(value_wrapper&& v) noexcept :
        semantics_{std::move(v.semantics_)},
        handle_{semantics_.move(v.handle_)}
    {}

    value_wrapper& operator=(const value_wrapper& v)
    {
        semantics_ = v._semantics;
        semantics_.copy_assign(handle_, v._handle);

        return *this;
    }

    value_wrapper& operator=(value_wrapper&& v) noexcept
    {
        semantics_ = std::move(v.semantics_);
        semantics_.move_assign(handle_, std::move(v.handle_));

        return *this;
    }

    template<typename T>
    value_wrapper& operator=(const T& value)
    {
        semantics_.copy_assign(handle_, value);

        return *this;
    }

    template<typename T>
    value_wrapper& operator=(T&& value)
    {
        semantics_.move_assign(handle_, std::move(value));

        return *this;
    }

    ~value_wrapper()
    {
        semantics_.destroy(handle_);
    }

    const value_type& get() const
    {
        return semantics_.deref(handle_);
    }

    value_type& get()
    {
        return semantics_.deref(handle_);
    }

    operator const value_type&() const
    {
        return get();
    }

private:
    Semantics semantics_;
    handle_type handle_;
};
{% endhighlight %}

As you can see, what `value_wrapper` is doing is to just bypass the special member function calls to the `Semantics` operating on the handle. `Semantics` has a member function with a verbose name for each special member function of a class. That's the way you customize the value semantics, by implementing a policy class with that functions.

## An example: `ptr_semantics`

Let's look at a simple example: Pointer semantics. That is, to give value semantics to a dynamically allocated object.

{% highlight cpp %}
template<typename T, typename Allocator = std::allocator<T>>
struct ptr_semantics : public default_value_semantics<ptr_semantics<T,Allocator>,T*>
{
    using value_type = T;
    using handle_type = T*;

    ptr_semantics(const Allocator& alloc = Allocator{}) :
        alloc_{alloc}
    {}

    template<typename... Args>
    handle_type construct(Args&&... args)
    {
        auto ptr_ = alloc_.allocate(1);
        alloc_.construct(ptr_, std::forward<Args>(args)...);
        return ptr_;
    }

    handle_type move(handle_type& handle)
    {
        handle_type ptr = std::move(handle);
        handle = nullptr;
        return ptr;
    }

    handle_type& move_assign(handle_type& handle, handle_type& other)
    {
        handle = std::move(other);
        other = nullptr;
        return handle;
    }

    void destroy(handle_type handle)
    {
        if(handle == nullptr) return;

        alloc_.destroy(handle);
        alloc_.deallocate(handle, 1);
    }

    const value_type& deref(handle_type handle) const
    {
        return *handle;
    }

    value_type& deref(handle_type handle)
    {
        return *handle;
    }

public:
    Allocator alloc_;
};
{% endhighlight %}

Given an allocator `Alloc` and a type `T`, `ptr_semantics<T,Alloc>` implements value semantics for pointers to objects allocated by `Alloc`. It inherits from `default_value_semantics` which implements `copy`, `move`, `copy_assign`, and `move_assign` operations in terms of `construct` and `deref`:

{% highlight cpp %}
template<typename Semantics,
        typename HandleType
>
struct default_value_semantics
{
    using handle_type = HandleType;

    default_value_semantics() :
            This{static_cast<Semantics*>(this)}
    {}

    handle_type copy(const handle_type& handle)
    {
        return This->construct(This->deref(handle));
    }

    handle_type move(handle_type&& handle)
    {
        return This->construct(std::move(This->deref(handle)));
    }

    handle_type& copy_assign(handle_type& handle, const handle_type& other)
    {
        This->deref(handle) = This->deref(other);
        return handle;
    }

    template<typename T>
    handle_type& copy_assign(handle_type& handle, const T& other)
    {
        This->deref(handle) = other;
        return handle;
    }

    ...

private:
    Semantics* This;
};
{% endhighlight %}

*Note we override `move` and `move_assign` in `ptr_semantics` since when working with pointers swapping them is enough. This is applicable to any situation where handles are cheap, but that has not to be always the case.*

Now we are able to avoid `new/delete` while having proper value semantics:

{% highlight cpp %}
int main()
{
	value_wrapper<ptr_semantics<int>> i = 0;

	std::cout << i.get();
}
{% endhighlight %}

No memory leaks, it copies when the wrapper is copyed, etc.

That's a fairly non useful example. Who might want to allocate `int`s in that way? Well, remember you can pass the allocator you want, so this type can be used to organize dynamic allocation of entities while preserving RAII-correctness and value semantics.  
But consider a more interesting example. Given an allocator that stores objects from a class hierarchy, doing the usual task of storing a set of polymorphic widgets is simple:

{% highlight cpp %}
struct base
{
    virtual ~base() = default;
    virtual void hello() const = 0;
};

struct derived1 : base
{
    derived1(int i = 0) : i_{i}
    {}

    void hello() const override
    {
        std::cout << "derived1! i=" << i_ << std::endl;
    }

private:
    int i_ = 0;
};

struct derived2 : base
{
    derived2(char c = 0) : c_{c}
    {}

    void hello() const override
    {
        std::cout << "derived2! c=" << c_ << std::endl;
    }

private:
    char c_ = 'a';
};

using value_t = value_wrapper<ptr_semantics<base, poly_allocator<base>>>;

int main()
{
    constexpr std::size_t n = 10;
    std::vector<value_t> v;
    
    v.emplace_back(derived1{1});
    v.emplace_back(derived2{'2'});
    v.emplace_back(derived1{3});
    v.emplace_back(derived2{'4'});

    for(const auto& e : v)
        e->hello();
}
{% endhighlight %}

Unlike when using the Standard Library, that vector is copyable and actually copies the objects it holds.

*Note: Writing such `poly_allocator` class worths another post just to cover that topic. I leave it as an exercise for the reader. Just a tip: [This](http://bannalia.blogspot.com.es/2014/05/fast-polymorphic-collections.html) may inspire you.*