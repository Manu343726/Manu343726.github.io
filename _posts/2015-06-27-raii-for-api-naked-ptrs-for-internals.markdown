---
layout: post
title: "RAII for API, Naked Ptrs for Internals?"
modified:
categories: 
excerpt:
tags: [c++]
image:
  feature:
date: 2015-06-27T14:27:10+02:00
---

It's well stablished that having raw pointers as part of a C++ API is not a good practice since it's not clear who (The user or the API internals) manages the resource. Consider:

{% highlight cpp %}

namespace int_buff
{
    int* factory();
    void have_fun(int*);
}

{% endhighlight %}

Let's say you have an API that operates on `int` buffers. The first function, `factory()`, returns a new buffer ready to play with. The latter `have_fun()` does something with a buffer.  
When reading such declarations, I have no idea if when requesting a new buffer via a `factory()` call if I'm in charge of deallocating it, or the return value from `factory()` is just a handle to an internal resource managed by the library. 

So, what's the solution? Documentation? That's what C has been doing for years: *"RTFM, you shouldn't call `memcpy()` with negative numbers, so never use `int` for sizes please."* Then we have heartbleed and similar bugs. **Docs don't prevent your code to compile**. [begin rant] That's why I prefer a language with a static type system, instead of `void*` and cast-land [end rant].

Instead, using C++ resource handlers, owning semantics are clear:

{% highlight cpp %}

std::shared_ptr<int> factory();

{% endhighlight %}

*"Oh, the library owns the buffer but it's being shared with me."* That makes sense. But what about `have_fun()`? Passing it as a `std::shared_ptr<int>` has no much sense. Adding potential mutex lock for something that will not change the handle in any way. `std::shared_ptr<int>&`? Too much syntactic nosise to just say "Hey, you don't own the ptr ok? It's RAII-managed by others". So I will continue with the naked `int*`.

## C++ classes = RAII + properties

In general, I'm agree with the rule "RAII-ed resources for the API, naked ones for internals" not just because the (neigible?) performance gains but to reduce noise. But there's something I like of owning resources trough objects: **Properties**. We usually wrap resources into objects to do RAII avoiding leaks thanks to our friend `}`. But objects (classes) also provide member functions, which can be used to ask for the state and other properties of the resource with a simple and concise syntax: I'm the only one that feels `array.size()` is far more readable than `sizeof(array)/sizeof/T)`?.

Is there any intermediate solution? Raw resources with properties?

## Raw resources, C++ version

Consider `std::vector`. It manages a dynamic array. `vector` has three kind of member functions: Special member functions (Constructors, destructor, etc to manage array lifetime), mutable functions (`push_back()`, `reserve()`, etc), and finally member functions that do not mutate the array itself but may change its data: `at()`, `front()`, back()`, etc.

What I propose is to split the implementation of resource handlers like `vector` into two different classes: The one which contains the resource and non-mutating methods, and the one which does RAII and guarantees proper management of the resource (So mutating members belong here):

{% highlight cpp %}

template<typename T>
struct raw_vector
{
    using value_type = T;
    using iterator = T*;

    raw_vector(T* buffer, std::size_t capacity, std::size_t size = 0) :
        data_{buffer},
        size_{size},
        capacity_{capacity}
    {}

    value_type& at(std::size_t i)
    {
        if(i < size())
            return (*this)[i];
        else
            throw std::out_of_range{};
    }

    value_type& operator[](std::size_t i)
    {
        return data_[i];
    }

    value_type& front()
    {
        return (*this)[0];
    }

    value_type& back()
    {
        return (*this)[size() - 1];
    }
    
    bool empty() const
    {
        return size() == 0;
    }

    iterator begin() const
    {
        return data_;
    }

    iterator end() const
    {
        return begin() + size();
    }

    std::size_t size() const
    {
        return size_;
    }

    std::size_t capacity() const
    {
        return capacity_;
    }

    const raw_vector<T>& raw() const
    {
        return *this;
    }

protected:
    T* data_ = nullptr;
    std::size_t size_ = 0, capacity_ = 0; 
};

{% endhighlight %}

Note that `raw_vector` does not own the dynamic array. It can be seen as a C++ *view* of a raw C dynamic array, with its same memory and performance footprint plus some C++ syntactic sugar. **You can actually use this class directly as vector view** (See bellow).  
`raw_vector` provides the non-mutable facilities, and on top of that `vector` owns the resource through RAII and implements all the operations that need to mutate the array:

{% highlight cpp %}

template<typename T>
struct vector : public raw_vector<T>
{
    using raw_t = raw_vector<T>;

    vector(std::size_t capacity) :
        raw_vector{new T[capacity], capacity, 0}
    {}

    vector(vector&& rhs) :
        raw_vector{rhs.data_, rhs.capacity_, rhs.size_}
    {
        rhs.data_ = nullptr;
        rhs.capacity_ = 0;
        rhs.size_ = 0;          
    }

    vector(const raw_vector<T>&) = delete;
    vector(raw_vector<T>&&) = delete;

    ... (Copy ctor, assigment operators)

    ~vector()
    {
        delete[] data_;
    }

    void insert(typename raw_t::iterator at, const T& e)
    {
        std::size_t pos = at - raw_t::begin(); //Beware of realloc

        if(raw_t::size() == raw_t::capacity())
        {
            T* buff_ = new T[raw_t::capacity() * 2];

            for(std::size_t i = 0; i < pos; ++i)
                buff_[i] = std::move(raw_t::data_[i]);

            buff_[pos] = e;

            for(std::size_t i = pos + 1; i < raw_t::size(); ++i)
                buff_[i] = std::move(raw_t::data_[i]);

            delete raw_t::data_;
            raw_t::data_ = buff_;
            raw_t::size_ += 1;
            raw_t::capacity_ *= 2;
        }
        else
        {
            for(std::size_t i = raw_t::size() - 1; i > pos; ++i)
                raw_t::data_[i] = raw_t::data[i-1];

            raw_t::data_[pos] = e;
            raw_t::size_ += 1;
        }
    }

    ... (erase(), push_back(), etc)

    raw_vector<T> view(typename raw_t::iterator begin, typename raw_t::iterator end) const
    {
        return {begin, 
                raw_t::capacity() - (begin - raw_t::begin()),
                end - begin};
    }
};

{% endhighlight %}

Note that `vector` has two deleted constructors, those that accept a raw vector as parameter. **Constructing a vector from a raw one should be discouraged since we don't know who owns that buffer**.

Now we are able to avoid RAII overhead in contexts where we know we are not owning nor changing a resource, like API internals, while having a RAII interface:

{% highlight cpp %}

namespace impl
{
    template<typename T>
    void wipe(raw_vector<T> v) //By value, like a pointer. No syntax noise. 
    {
        if(!v.empty()) //Raw, but with C++ sugar
        {
            for(std::size_t i = 0; i < v.size(); ++i)
                v[i] = i;
        }
    }
}

namespace my_api
{
    template<typename T>
    void wipe(vector<T>& v)
    {
        impl::wipe(v.raw());
    }
}

{% endhighlight %}