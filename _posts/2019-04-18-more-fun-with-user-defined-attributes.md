---
title: (More) Fun with user defined attributes
date: 2019-04-18
tags: [c++]
comments: true
---

## If C++ was a modern language...

For one reason or another I find myself doing a lot of python lately. The
experience I'm gaining with it brings some ideas and inspiration when I'm
lucky enough to touch C++ again. One example is my latest project,
[`unittest`](https://github.com/Manu343726/unittest).

`unittest` is one use case of reflection that's being around my mind since
I started working on [`tinyrefl`](http://github.com/Manu343726/tinyrefl)
two years ago. The idea was to try to write unit tests with as less wiring
code as possible, like what's possible in modern languages that use
reflection and decorators to declare unit tests.

> Yeah, until C++ gains reflection and a library reuse system (The term
> *"Package manager"* sounds too mainstream to me) it could not be
> considered modern. C++11 is "Modern" compared to C++98/03, but that's
> like calling the steam engine "Modern" because it's being compared with
> Egiptian slaves pushing giant stone blocks on ramps to build piramid
> looking structures.

Look, live preview!

Note that markdown features such as *italic* or **bold** are hidden if not
being edited.

The point of the `unittest` project is not whether the
`assert_called_with()` syntax is the best alternative for unit testing,
but **to know if we could actually write the same thing with C++**, and
learn what's needed in the process.

Here's an example of a dummy unit test written in Python3 with the
official [`unittest`
framework](https://docs.python.org/3/library/unittest.html):

``` python
import unittest, unittest.mock
import mynamespace

class ExampleTestCase(unittest.TestCase):

    @unittest.mock.patch('mynamespace.ExampleClass.identity', return_value=42)
    def test_another_one_bites_the_dust(self, identity):
        object = mynamespace.ExampleClass()

        self.assertEqual(object.methodThatCallsIdentity(), 42)
        identity.assert_called_once_with(43)
```

The above program outputs this:

```
test_another_one_bites_the_dust (test_example.ExampleTestCase) ... FAIL

======================================================================
FAIL: test_another_one_bites_the_dust (test_example.ExampleTestCase)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/lib/python3.7/unittest/mock.py", line 1195, in patched
    return func(*args, **keywargs)
  File "/home/manu343726/Documentos/unittest/examples/python_equivalent/test_example.py", line 11, in test_another_one_bites_
the_dust
    identity.assert_called_once_with(43)
  File "/usr/lib/python3.7/unittest/mock.py", line 831, in assert_called_once_with
    return self.assert_called_with(*args, **kwargs)
  File "/usr/lib/python3.7/unittest/mock.py", line 820, in assert_called_with
    raise AssertionError(_error_message()) from cause
AssertionError: Expected call: identity(43)
Actual call: identity(42)

----------------------------------------------------------------------
Ran 1 test in 0.002s

FAILED (failures=1)
```

Compare that example with this C++14 code using `unittest`:

``` cpp
#include <unittest/unittest.hpp>
#include <libexample/example.hpp>
#include <libexample/example.hpp.tinyrefl>

namespace test_example
{

struct ExampleTestCase : public unittest::TestCase
{
    [[unittest::patch("mynamespace::ExampleClass::identity(int) const", return_value=42)]]
    void test_another_one_bites_the_dust(unittest::MethodSpy<int(int)>& identity)
    {
        mynamespace::ExampleClass object;

        self.assertEqual(object.methodThatCallsIdentity(), 42);
        identity.assert_called_once_with(43);
    }
};

}
```

and its output:

```
test_another_one_bites_the_dust (test_example::ExampleTestCase) ... FAIL

=======================================================================
FAIL: test_another_one_bites_the_dust (test_example::ExampleTestCase)
-----------------------------------------------------------------------
Stack trace (most recent call last):
#0    Source "/home/manu343726/Documentos/unittest/examples/test_example.hpp", line 16, in test_another_one_b
ites_the_dust
         13:         mynamespace::ExampleClass object;
         14: 
         15:         self.assertEqual(object.methodThatCallsIdentity(), 42);
      >  16:         identity.assert_called_once_with(43);
         17:     }
         18: };

AssertionError: Expected call: mynamespace::ExampleClass::identity(43)
Actual call: mynamespace::ExampleClass::identity(42)

-----------------------------------------------------------------------
Ran 1 tests in  0.002s

FAILED (failures=1)
```

## Unleash the beast!

`unittest` does a lot of black magic behind the scenes to get an example
that concise (Generates a `main.cpp` file that includes all your unit test
headers, runs `tinyrefl-tool` to parse your code and generate reflection
data, uses [`elfspy`](https://github.com/Manu343726/elfspy) to monkey
patch the test executable link table to add spies to your library
functions, etc), but for me the most interesting of its secrets is the
emulation of the
[`patch()`](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.patch)
decorator from Python's
[`unittest.mock`](https://docs.python.org/3/library/unittest.mock.html)
library.

Python's `patch()` allows to override the given target entity (a function,
a class, a method, etc) with a custom mock object during the context of
the function that's being decorated (Usually a test). Since I just wanted
a proof of concept, I restricted my `patch()` counterpart to mock member
functions only. To do that, I needed to:

1. Find out if the current test function is being "decorated" with
   a `[[unittest::patch()]]` user defined attribute.

2. Find out if the attribute has any argument. If that's the case, try to
   parse the first argument as a string literal containing the full
   display name of the method to mock

   > Full display name means the full qualified name of the method
   > ("namespace::class::method") with its exact signature ("(int, char,
   > bool)")

3. Search in the reflection metadata a function matching the given display
   name.

4. If exists, get the pointer to the function and pass it to `elfspy` to
   do the mocking magic. If it doesn't exists, raise an error.

Note every step except the monkey patching is fully static, run at compile
time. For example, in the case the target of `patch()` does not exist, an
`static_assert()` is fired.

The whole thing looks like this:

``` cpp
template<typename TestCase>
void runTestCase()
{

TestCase testCase;

// Loop through all the public member functions 
// of the test case class, visiting only the ones
// we consider a unit test:
tinyrefl::visit_class<TestCase>(
[](auto /* testName */, auto /* depth */, auto method,
    TINYREFL_STATIC_VALUE(tinyrefl::entity::MEMBER_FUNCTION))
    -> std::enable_if_t<
        is_test_method<decltype(method)>()
    >
{
  using Method = decltype(method);
  constexpr Method constexpr_method; // constexpr args for C++23 pls!

  if constexpr (constexpr_method.has_attribute("patch") &&
    constexpr_method.get_attribute("patch")
        .namespace_.full_name() == "unittest" &&
    constexpr_method.get_attribute("patch").args.size() >= 1)
  {
    // Get the full display name.
    // pad(1, 1) needed to remove the quotes from the string literal
    constexpr auto target_id =
        constexpr_method.get_attribute("patch").args[0].pad(1,1);

    // Do we know of any entity named that way?
    if constexpr (tinyrefl::has_entity_metadata<target_id.hash()>())
    {
      using Target = tinyrefl::entity_metadata<target_id.hash()>;
            
      if constexpr (Target::kind != tinyrefl::entity::MEMBER_FUNCTION))
      {
        static_assert(sizeof(TestCase) != sizeof(TestCase),
          "[[unittest::patch()]] target is not a member function");

        // static_assert() with constexpr string parameter would be
        // great here
      }

      // Tell elfspy to do its magic during this scope
      MethodSpyInstance<Target> spy; 

      // Invoke the test method with the spy as parameter
      method.get(testCase, spy);
    }
    else
    {
      static_assert(sizeof(TestCase) != sizeof(TestCase),
        "[[unittest::patch()]] target not found");
    }
  }
  else
  {
    // No patch, call the method with no spy
    method.get(testCase);
  }
});

}
```

The problem with this approach is that **the parsing of the attribute
looks awful**. It's just a chain of edge cases completely hardcoded, full
of asumptions (*What if the first parameter is not an string literal?*),
and as verbose as it could possibly get. 
Having user defined attributes accesible through reflection is great, but
having to work this way is not. Once attributes are not mere tags we're
doomed.

## What I would like to see

Let's take another example, very common in the gamedev world: A value
exposed to the engine GUI that is clamped in some way:

``` cpp
struct Camera
{
    float x, y, z;
    float eye_x, eye_y, eye_z;

    [[editable::range(0.0f, 1.0f)]]
    float fov;
};
```

In the ideal world, where we have unicorns instead of project managers and C++
has modules that are actual modules, that expression will just return us the
exact information we want: That the field `fov` can be edited from the IDE in
the range `(0, 1)`. With actual modern languages like C# this is easy, since
attributes are classes you implement and the attribute syntax is an invocation
to the class constructor:

``` csharp
using System.ComponentModel.DataAnnotations

public class Camera
{
    [Range(0, 1)]
    public float fov;
}
```

Then if you check the attributes of `fov` using C#'s reflection you will get
a pretty instance of the `Range` class.

So: **How could we make C++ user defined attributes be classes too?**

## Attribute classes for C++

With my `tinyrefl` library and a bit of imagination I think attribute classes
could be implemented for C++14 following three "simple" steps:

### 1. Find a class named as the attribute

This is similar to what we've done with `unittest`, use `tinyrefl` global
reflection info to find a class with that name:

``` cpp
template<typename Metaobject>
constexpr auto get_attribute(const Metaobject&)
{
    constexpr Metaobject info;

    static_assert(info.get_attributes().size() > 0);

    constexpr auto attribute_info = info.get_attributes()[0];
    constexpr auto attribute_class_id =
        attribute_info.name.full_name().hash();

    if constexpr(tinyrefl::has_entity_metadata<attribute_class_id>() &&
                 tinyrefl::entity_metadata<attribute_class_id>::kind ==
                 tinyrefl::entities::CLASS)
    {
        using attribute_class = typename tinyrefl::entity_metadata<
            attribute_class_id>::class_type;

        ...
    }
    else
    {
        // error
    }
}
```

### 2. Check if the class is an attribute

We could do it the C# way and inherit from an `attribute` class:

``` cpp
class attribute {};

template<typename T>
constexpr bool is_attribute_v = std::is_base_of<attribute, T>::value;
```

But let's get a bit more meta and use an attribute to tag attribute classes:

``` cpp
template<typename T>
constexpr bool is_attribute_v =
    tinyrefl::has_metadata<T>() &&
    std::is_class<T>::value &&
    tinyrefl::has_attribute<T>("attribute");

// Here's an attribute:
namespace editable
{
    [[attribute]]
    struct range
    {
        float begin, end;

        constexpr range(float begin, float end) :
            begin{begin},
            end{end}
        {}

        constexpr bool value_in_range(float value) const
        {
            return begin >= value && value <= end;
        }
    };
}
```

Back at `get_attribute()` function we use it as follows:

``` cpp
template<typename Metaobject>
constexpr auto get_attribute(const Metaobject&)
{
    ...

    if constexpr(...)
    {
        ...

        if constexpr(is_attribute_v<attribute_class>)
        {

        }
    }
    else
    {
        // error
    }
}
```

### 3. Parse attribute arguments

The ideal would be to generate a tuple (`std::tuple` is `constexpr`!) from
the attribute arguments. I think both [Hana
Dusíková](https://github.com/hanickadot/compile-time-regular-expressions)
and [Jonathan Müller](https://github.com/foonathan/lex) would have some
suggestions for this.

> **UPDATE:** [Hana gave me a lot of
> feedback](https://twitter.com/hankadusikova/status/1121104154481635329) and
> we decided a C++ 14 backport of her CTLL library will be the best option.
> Thanks again Hana!

Once you get the tuple of arguments, it's just
a [`std::apply()`](https://en.cppreference.com/w/cpp/utility/apply) call
on the attribute class constructor:

``` cpp
template<typename Metaobject>
constexpr auto get_attribute(const Metaobject&)
{
    ...
    
    return std::apply([](auto&&... args) constexpr
    {
        return attribute_class{std::forward<decltype(args)>(args)...};
    }, parse_args(attribute_info));
}
```

## The result

The final user side snippet would look something like this:

``` cpp
void set_value_from_ide(Camera& camera, const std::string& field, float
value)
{
    tinyrefl::visit_class<Camera>( 
        [&](const std::string& name, /* depth */, auto variable_metadata,
        TINYREFL_STATIC_VALUE(tinyrefl::entity::MEMBER_VARIABLE))
    {
        constexpr decltype(variable_metadata) metadata;
        
        if(field == name)
        {
            constexpr auto attribute = get_attribute(metadata);

            if constexpr (std::is_same<
                decltype(attribute), editable::range>::value)
            {
                if(attribute.value_in_range(value))
                {
                    throw std::out_of_bounds{
                        "Invalid value, not in range of the field"};
                }
            }

            metadata.get(camera) = value;
        }
    });
}
```

Maybe C++ could be modern after all.
