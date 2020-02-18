---
title: Objects anywhere, anytime
date: 2016-04-17 10:51:34 -07:00
tags:
- c++
- reflection
- datatypes
layout: post
modified: 
comments: true
---

A long time ago, in [a post](http://manu343726.github.io/2016/03/13/the-road-for-reflection-tagging-types.html)
far far away... we seen how `constexpr` can be used to retrieve and store
type names at compile time, so C++ types can be tagged and indexed without
relying on RTTI.

The goal of this series is to build a simple reflection engine. Since the
engine is dynamic, we need a way to handle objects of heterogenous types
at runtime. A kind of Boost.Any, but focused on our future reflection
needs.

## Manu unchained

There are situations where I would like to live in that wonderful world of
JavaScript, where everybody can become everything they want: A string, an
integer, an elephant... Sounds great, but sadly I'm living as a slave of
the C++ type system.

Thankfully C programmers were getting rid of the C type system since the
very birth of the language, doing safe generic programming in terms of
`void*`. For us, `void*` is a simple way to instance and reference *any*
kind of C++ object in an homogeneous manner, regardless of its type:

``` cpp
template<typename T>
void* create()
{
    return new T();
}

template<typename T>
void destroy(void* object)
{
    delete reinterpret_cast<T*>(object);
}

int main()
{
    void* intObject = create<int>();
    void* boolObject = create<bool>();

    // Welcome to Java
    std::vector<void*> arrayList{intObject, boolObject};

    destroy<int>(intObject);
    destroy<bool>(boolObject);
}
```

Simple, like most C-like APIs look like. It has some caveats though:

 - **Type information was lost**: With `void*` we have lost type
   information, having to remember the type of each object so we
   manipulate them accordingly. What would happen if we treat a `bool` as
   a `std::string`? Maybe we're writing a page in the history of humanity
   by starting WWIII or something...

 - **Manual memory management**: As with type information, using `void*` means
   we've lost C++ automatic storage duration, having to reason about object
   (and its associated memory) lifetime ourselves. Which of course all of us
   love since it makes C++ programming even more challenging.

Anyway, the thing still holds. Let's see how we can improve this simple
idea with lightweight C++ abstractions.

## Metatype

The first issue with the simple API above is that by referencing everything
through a `void*` we have lost that precious information that a type system
gives to us. Well, not exactly. Maybe the compiler had lost that information,
**but we know exactly what type was used and could access that info at
runtime**.

The first abstraction on the object API is a class in charge of
manipulating objects of an specific type at runtime. That is, a class that
represents **a C++ type at runtime**. I call this class `MetaType`:

``` cpp
class MetaType
{
public:
    void* create();
    void destroy(void* object);
    const TypeInfo& typeInfo() const;

private:
    TypeInfo _typeInfo;
};

bool operator==(const MetaType& lhs, const MetaType& rhs);
bool operator!=(const MetaType& lhs, const MetaType& rhs);
```

A `MetaType` variable represents a particular instance of the API above,
keeping track of the type being used with that particular API:

``` cpp
MetaType intType{int}, boolType{bool};
assert(intType != boolType);

void* intObject = intType.create();
void* boolObject = boolType.create();

...

intType.destroy(intObject);
boolType.destroy(boolObject);
```

*But Manu, that doesn't event compile*. Ok, ok; I was cheating you. Sadly
C++ is not (yet) that cool, so we have to do some extra work. We have to
find a way to store all type behaviors we need (By behaviors I mean how we
manipulate instances of a type), but all that behaviors should be kept
**in variables of the same type**. `void*` is a way of type punning as
we've seen, but C++ has another cool one: Inheritance-based polymorphism:

``` cpp
class MetaType
{
    ...

private:
    class TypeBehavior
    {
    public:
        TypeBehavior(const TypeInfo& typeInfo);
        virtual ~TypeBehavior() = default;

        const TypeInfo& typeInfo() const;

        virtual void* create() = 0;
        virtual void destroy(void* object) = 0;

    private:
        TypeInfo _typeInfo;
    };

    std::shared_ptr<TypeBehavior> _behavior;

    template<typename T>
    class TypeBehaviorFor : public TypeBehavior
    {
    public:
        TypeBehaviorFor() : TypeBehavior{TypeInfo::get<T>()}
        {}

        void* create() override
        {
            return new T();
        }

        void destroy(void* object) override
        {
            delete reinterpret_cast<T*>(object);
        }
    };
};
```

My beloved templates to the rescue. What we have done here is to define
a common interface for type behaviors, then use a template to define the
behavior of each requested (instanced) type.

*`TypeBehavior` class above is rather minimalistic, you may want to
instance objects given constructor parameters, assign objects, copy them,
etc. The post only shows the most simple interface, but the real world
code both the reader and I would write includes more overloads of
`create()` method and others such as `copy()`, `assign()`, etc.*

*Also note I'm calling raw `new`/`delete`. Of course this is not optimal
but simple for the post too. I suggest you to use an object pool instead.
With a pool this works great since most allocations are just recycling old
dead objects.*

Now any `MetaType` object just asks its internal behavior about what to
do:

``` cpp
void* MetaType::create()
{
    return _behavior->create();
}

void MetaType::destroy(void* object)
{
    return _behavor->destroy(object);
}

const TypeInfo& MetaType::typeInfo() const
{
    return _behavior->typeInfo();
}

bool operator==(const MetaType& lhs, const MetaType& rhs)
{
    return lhs.typeInfo() == rhs.typeInfo();
}

bool operator!=(const MetaType& lhs, const MetaType& rhs)
{
    return !(lhs == rhs);
}
```

Finally write the usual factory template and we are done:

``` cpp
class MetaType
{
public:
    template<typename T>
    MetaType get()
    {
        return {std::make_shared<TypeBehaviorFor<T>>()};
    }

    ...

private:
    MetaType(const std::shared_ptr<TypeBehavior>& behavior) :
        _behavior{behavior}
    {}

    ...
};
```

The user code looks more or less like this:

``` cpp
auto intType = MetaType::get<int>();

void* integer = intType.create();
...
intType.destroy(integer);
```

### Metatype for reflection

`MetaType` class shown above is the minimal needed to write something in
the lines of `Boost.Any`. But for dynamic reflection one may need more
features, specifically, asking for C++ types given their name:

``` cpp
void* createObject(const std::string& typeName)
{
    auto type = MetaType::get(typeName);
    return type.create();
}
```

To do that, `MetaType` should be able to register known types. The
simplest way by holding a `typeName -> behavior` table as part of
`MetaType` class:

``` cpp
class MetaType
{
public:
    template<typename T>
    MetaType get()
    {
        static std::shared_ptr<TypeBehavior> behavior = []
        {
            auto behavior = std::make_shared<TypeBehaviorFor<T>>();
            _registry[ctti::unnamed_type_id<T>()] = behavior;
            return behavior;
        };

        return {behavior};
    }

    MetaType get(const std::string& typeName)
    {
        return {_registry[ctti::id_from_name(typeName)]};
    }

    ...

private:
    std::shared_ptr<TypeBehavior> _behavior;
    ...
    static std::unordered_map<
        ctti::unnamed_type_id_t,
        std::shared_ptr<TypeBehavior>
    > _registry;
}
```

Now as long as you've statically used a type once, you would be able to
ask for it by name:

``` cpp
MetaType::get<int>(); // "register" the type

auto intType = MetaType::get("int");
void* integer = intType.create();
intType.destroy(integer);
```

*If you ever worked with C++ libraries interacting with dynamic type
systems (Boost.Python and luabind come to my mind), you might have noticed
those always involve some kind of "registration" step. That registration
is a bootstrapping process where the library sets up all the information
that it needs to map, among other things, the C++ static type system to
its dynamic counterparts. Is in this boring registration code were an
external tool may help a lot. But that's a topic for following posts on
libclang API...*

## MetaObject

So far we've solved one of our two problems with the *simple C-like object
API*. But we still have to create and destroy objects manually (And don't
forget to use the correct `MetaType`! Was the first problem solved at
all?!!).

This is solved by reintroducing objects into the RAII principle, wrapping
them with a handle class in charge of managing their lifetime by means of
constructors and destructors:

``` cpp
class MetaObject // Aka "Any"
{
public:
    template<typename T>
    MetaObject(T&& value);

    template<typename T>
    const T& get() const
    template<typename T>
    T& get();

    template<typename T>
    operator const T&() const;
    template<typename T>
    operator T&();

    const MetaType& type() const;

private:
    MetaType _type;
    std::shared_ptr<void> _object;
};
```

`MetaObject` class keeps together both the object and its type, so objects
are always manipulated using the right type. A couple of `get()` methods
give access to the underlying object:

``` cpp
template<typename T>
const T& MetaObject::get() const
{
    assert(MetaType::get<T>() == _type);
    return *reinterpret_cast<const T*>(_object.get());
}

template<typename T>
T& MetaObject::get()
{
    assert(MetaType::get<T>() == _type);
    return *reinterpret_cast<T*>(_object.get());
}
```

Usage of metaobjects is simplified by automatic conversion operators:

``` cpp
template<typename T>
MetaObject::operator const T&() const
{
    return get<T>();
}

template<typename T>
MetaObject::operator T&()
{
    return get<T>();
}
```

``` cpp
int add(int a, int b)
{
    return a + b;
}

int main()
{
    MetaObject a{1}, b{2};

    f(a, b);
}
```

*"Simplified". Many people don't like automatic conversions, but I find
them useful in situations like this to make user code more pleasant to
write and read. I always try to be pragmatic instead of dogmatic and
pedant, but I'm sure I don't always act this way. In fact many of my best
friends are sick of me, in their words, "Oh shit let's run away, he's
starting a pedant C++ discourse again!" while having beers after the
work/university... I'm sorry guys :)*

A `std::shared_ptr` is used to manage object lifetime. We could have
written a custom destructor, constructors, and assignment operators; fut
I feel simpler to use a standar library smart pointer with a custom
deleter:

``` cpp
class Deleter
{
public:
    Deleter(const MetaType& type) :
        _type{type}
    {}

    void operator()(void* object)
    {
        _type.destroy(object);
    }

private:
    MetaType _type;
};

template<typename T>
MetaObject::MetaObject(T&& value) :
    _type{MetaType::get<std::decay_t<T>>()},
    _object{
        _type.create(std::forward<T>(value)),
        Deleter{_type}
    }
{}
```

*AFAIK `std::shared_ptr` doesn't give public access to its deleter, so
I had to store type twice (One for the `MetaObject` checks and other for
the deleter).*

## Conclusion

Having a generic type gives us the chance to manipulate any
C++ object in an homogeneous way, which will be very useful when accessing
and manipulating elements of classes with dynamic reflection features.

In next posts we will introduce the reflection runtime model, with the
classes needed to access class member functions, class fields, and
finally, a registry with all class information.

## Appendix: Playing with metaobjects

The conversion operators defined as part of `MetaObject` makes simple to
pass values back and forth this type, like we've seen in the example:

``` cpp
MetaObject integer{1};
int i = integer;
```

On top of these we can write some interesting tools:

### Heterogeneous vector container

Thanks to generic types `std::vector` could work as a container of
heterogeneous values. Like a type erased `std::tuple`:

``` cpp
template<typename... Ts>
std::vector<MetaObject> pack_to_vector(Ts&&... values)
{
    return { MetaObject{std::forward<Ts>(values)}... };
}
```

Using the indices trick a tuple alternative looks like this:

``` cpp
template<typename... Ts, std::size_t... Is>
std::vector<MetaObject> tuple_to_vector(const std::tuple<Ts...>& tuple,
                                        std::index_sequence<Is...>)
{
    return pack_to_vector(std::get<Is>(tuple)...);
}

template<typenbame... Ts>
std::vector<MetaObject> tuple_to_vector(const std::tuple<Ts...>& tuple)
{
    return tuple_to_vector(tuple, std::make_index_sequence_for<Ts...>{});
}
```

*You might want to add two overloads (tuple lvalue and tuple rvalue) to
reduce copies*.

A `vector_to_tuple()` function is left as an exercise for the reader.

### Invoking functions

Following with the heterogeneous `std::vector` container, we could write
an utility to invoke functions using a vector of arguments, in the line of
`tuple_call()`:

``` cpp
template<typename F, std::size_t... Is>
auto vector_call(const std::vector<MetaObject>& args, F function,
                 std::index_sequence<Is...>)
{
    return function(args[Is].get<function_argument_t<Is, F>>()...);
}

template<typename F>
auto vector_call(const std::vector<MetaObject>& args, F function)
{
    return vector_call(
        args,
        function,
        std::make_index_sequence<function_arguments<F>::size>{}
    );
}
```

Here we query the function signature expand vector elements to the
corresponding function argument types.

*Getting the signature if a function in a generic way is not a simple task, C++
supports many flavors of function-like entities. This interesting topic
deserves an entire post (Or a talk for a future Madrid C++ meetup as one our
members suggested!). Just for completeness I will leave here a link to the
utility header being used in my reflection engine: `#include <siplasplas/utility/[function_traits.hpp]
(https://github.com/GueimUCM/siplasplas/blob/master/include/siplasplas/utility/function_traits.hpp)>`*

Given the function signature (A type list with all the function arguments
types), to invoke the function with the vector we "unpack" it by getting the
value of each `MetaObject` using the corresponding type.

### Having fun

All the tools above can be combined to do some interesing things. Here's
a little spoiler/example: As part of the reflection engine I wrote an attribute
API for functions inspired in C# attributes syntax.

That API is based in an `Attribute` interface as follows:

``` cpp
class Attribute
{
public:
    virtual std::vector<MetaObject> processArguments(const std::vector<MetaObject>& args) = 0;
    virtual MetaObject processReturnValue(const MetaObject& value) = 0;
};
```

`Attribute` instances are injected before and after function invocation:

``` cpp
ReturnType invoke(const std::vector<MetaObject>& args, Attribute& attr, F f)
{
    auto processedArgs = attr.processArguments(args);
    auto returnValue = vector_call(processedArgs, f);
    return attr.processReturnValue(returnValue);
}
```

This works great to add decorators to member functions of a class, such as
precondition checking or logging function calls. You can also troll your
users by changing the sign of integer arguments, clearing strings, etc.
You're free.

That's enough. More on attributes in following posts!
