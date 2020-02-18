---
title: Bitchunks
date: 2015-08-24 00:47:25 -07:00
tags:
- c++
- abstractions
- datatypes
layout: post
---

*I would have liked to talk about Concepts Lite, but I'm still working on contiguous integration support for the examples I want to share. Maybe in a couple of weeks.*

Today I will talk about a simple data structure, what I call "bitchunks".

## `std::bitset`

The C++ Standard Library ships with a template called `std::bitset`, which represents fixed-size sequences of bits. It works great for bit twiddling, giving a clean and sane "container like" interface instead of clever C macros:

{% highlight cpp %}

std::bitset<8> bits = 0xFF;

bits[0] = false; // Unsets LSB bit

auto foo = (bits >> 2) | bits;

{% endhighlight %}

It has however some caveats:

 - **It's statically sized**: You cannot declare a `bitset` which size is known at runtime. There are some alternatives like `boost::dynamic_bitset` that solve this issue.
 - **Gives access to individual bits only**: The `bitset[bit]` syntax is nice, but there are situations where you want to manipulate more than one bit at once, like getting the first 52 bits off a 64 bit number.

The `bitchunk` template I show here is designed to solve the second issue.

## `bitchunk`

While both host a value supposed to be manipulated at bit level, the goals of `bitchunk` are different from `std::bitset`:

 - The main goal is to, given a value of a basic type (Like `int`, `float`, etc), being able to **extract specific** ***chunks*** **of bits from the value**, manipulate that chunks; manipulating the value itself.

 - bitchunks should be "chunkable" too, in a way transparent to the user. That is, if we got the first three bits from a number, it should be possible to get the last two bits from that chunk (which are the bits 2-1 from the original number) maintaining bit ownership on the original number:

  {% highlight text %}

auto i = make_bitchunk(0x0F0F0); // 00001111000011110000
auto j = i(0,3); //          000 ------------------> ^^^
auto k = j(1,3); // 00 ----> ^^

k = 0b11;
assert(i == 0x0F0F6);

{% endhighlight %}

## Bit manipulation tools

First I will write a set of simple C-like functions to do bit twiddling. This way we encapsulate all the bit manipulation code in a separated `utility.hpp` header simple to read and debug.

### Raw data manipulation

Since the `bitchunk` template is supposed to work with any integral/floating-point type, I decided to declare an alias to the types I will use for bit twiddling:

{% highlight cpp %}
namespace detail {
  using raw_data_t = std::uint64_t;
  using bit_index_t = std::size_t;
}
{% endhighlight %}

The first declares the type we will work with when doing byte manipulation. The second is the type used for accessing bits by index.

Floating point values cannot be reinterpreted directly as integrals, so we should handle floats and integrals in different ways. Instead of having ugly casts across the code, I wrote a template that handles different cases through specialization, then a simple get/set interface on top of that:

{% highlight cpp %}
template<typename T, typename = void>
struct raw_data_accessor
{
    static raw_data_t get(T* data_ref)
    {
        return (raw_data_t)(*data_ref);
    }

    static void set(T* data_ref, raw_data_t newdata)
    {
        *data_ref = (T)(newdata);
    }
};

template<typename T>
struct raw_data_accessor<T, std::enable_if_t<std::is_floating_point<T>::value>>
{
    static raw_data_t get(T* data_ref)
    {
        return *(reinterpret_cast<raw_data_t*>(data_ref));
    }

    static void set(T* data_ref, raw_data_t newdata)
    {
        *data_ref = *(reinterpret_cast<T*>(&newdata));
    }
};

namespace raw_manip
{
    template<typename T>
    raw_data_t get(const T& value)
    {
        return raw_data_accessor<T>::get(&value);
    }

    template<typename T>
    raw_data_t get(T& value)
    {
        return raw_data_accessor<T>::get(&value);
    }

    template<typename T>
    raw_data_t get_by_ptr(T* value_ref)
    {
        return raw_data_accessor<T>::get(value_ref);
    }

    template<typename T>
    void set(T& value, raw_data_t new_value)
    {
        return raw_data_accessor<T>::set(&value, new_value);
    }

    template<typename T>
    void set_by_ptr(T* value_ref, raw_data_t new_value)
    {
        return raw_data_accessor<T>::set(value_ref, new_value);
    }
}
{% endhighlight %}

*Why passing by pointer by default? Just legacy, these functions were written for bitchunk implementation, and in most of the cases bitchunks operates on view mode, holding a pointer to the value instead of the value itself. See the bitchunk implementation bellow.*

### Bit manipulation functions

Just simple one-liners I hope my compiler will inline:

{% highlight cpp %}
inline raw_data_t bitmask_ononly(std::size_t begin, std::size_t end)
{
    return (allbitson >> (sizeof_bits<raw_data_t>() - end)) << begin;
}

inline raw_data_t bitmask_clear(std::size_t begin, std::size_t end)
{
    return ~bitmask_ononly(begin, end);
}

template<typename T>
inline raw_data_t clear_high_bits(T i, bit_index_t begin)
{
    return raw_manip::get(i) & bitmask_clear(begin,sizeof_bits<T>());
}

template<typename T>
inline raw_data_t truncate(T i, std::size_t width)
{
    return clear_high_bits(i, width);
}

template<typename T>
inline raw_data_t high_part(T i, bit_index_t pivot)
{
    return raw_manip::get(i) >> pivot;
}

template<typename T>
inline raw_data_t low_part(T i, bit_index_t pivot)
{
    return clear_high_bits(i, pivot);
}

template<typename T>
inline raw_data_t read_chunk(T data, bit_index_t begin, bit_index_t end)
{
    return clear_high_bits(data, end) >> begin;
}
{% endhighlight %}

I'm not very proficient with bits, I'm sure there are better hacks than the functions above.
Finally, these functions use two simple `constexpr` recipes I usually have as part of the `utility.hpp` file of my projects:

{% highlight cpp %}
template<typename T>
constexpr std::size_t sizeof_bits()
{
    return sizeof(T) * CHAR_BIT;
}

constexpr raw_data_t allbitson = (0ull - 1ull);
{% endhighlight %}

## Bitchunk implementation

So far we have covered the most boring part of the post. At least was the most boring for me to write...

Now let's dive into `bitchunk`: Remember the purpose of this data type: To give access to chunks of bits from a value, so we can read some specific bits of a value, manipulate them, etc. All with a syntax simple for the user.
Also one of the key features I wanted when started with this is the ability of creating nested chunks, to be able to ask for a *sub-*chunk of another chunk.

For this reasons, I wrote `bitchunk` covering this two sides in two different specializations: In simple terms, the first specialization holds a full value and looks at its bits, and the second is **a view to a specific range of bits from a value of type `T`**.  
Why not having `bitchunk` as a view class only? Mostly because it was designed with storage in mind. Also since the range of the view is defined at runtime, it's not as efficient as it can (should?) be for the cases you want to look at the whole value (The range `[0,sizeof_bits<T>)`).

{% highlight cpp %}
template<typename T, bool FullRange>
struct bitchunk_base
{
    bitchunk_base() = default;

    bitchunk_base(T data, bit_index_t begin = 0, bit_index_t end = sizeof_bits<T>()) :
        data_{data}
    {}

    constexpr bit_index_t begin() const
    {
        return 0;
    }

    constexpr bit_index_t end() const
    {
        return sizeof_bits<T>();
    }

    const T& data() const
    {
        return data_;
    }

    T& data()
    {
        return data_;
    }

    T* data_ptr()
    {
        return &data_;
    }

    T* data_ptr() const
    {
        return &data_;
    }

private:
    T data_;
};
{% endhighlight %}

As you can see, it's just a wrapper around a `T` value. The point of this specialization is to give storage for the `T` and the accessors the `bitchunk` needs to work (The range of bits it operates, accessors to the value, etc).  
The second specialization has the same interface, but behaves as a view to a `T` instead of storage:

{% highlight cpp %}
template<typename T>
struct bitchunk_base<T,false>
{
    bitchunk_base() = default;

    bitchunk_base(const T& data_ref, bit_index_t begin = 0, bit_index_t end = sizeof_bits<T>()) :
        data_ref_{&data_ref},
        begin_{ begin },
        end_{ end }
    {}

    bit_index_t begin() const
    {
        return begin_;
    }

    bit_index_t end() const
    {
        return end_;
    }

    const T& data() const
    {
        return *data_ref_;
    }

    T& data()
    {
        return *data_ref_;
    }

    T* data_ptr() const
    {
        return data_ref_;
    }

    T* data_ptr()
    {
        return data_ref_;
    }

private:
    T* data_ref_ = nullptr;
    bit_index_t begin_ = 0, end_ = 0;
};
{% endhighlight %}

Covered the implementation details into different base classes, `bitchunk` looks like this:

{% highlight cpp %}
template<typename T, bool FullRange = true>
struct bitchunk : public bitchunk_base<T, FullRange>
{
    using base = bitchunk_base<T, FullRange>;

    bitchunk() = default;

    bitchunk(const T& data, bit_index_t begin = 0, bit_index_t end = sizeof_bits<T>()) :
        base{ data, begin, end }
    {}

    template<bool fullrange>
    bitchunk(bitchunk<T,fullrange>& other, bit_index_t begin = 0, bit_index_t end = sizeof_bits<T>()) :
        base{ other.data_ptr(), other.begin() + begin, other.begin() + end }
    {}

    ...
};
{% endhighlight %}

Do you remember the *sub-chunking* ability I insisted above? The trick is that the view specialization of `bitchunk_base` is a view **to any `T`**, another bitchunks included.  

Note how the constructor taking a `bitchunk` initializes the range: The range the user specifies is the range on that value, regardless its an integer value or a view. This was done to return bitchunks transparently. Imagine you have a network protocol specified as: *Packets are stored as 32 bit integers, where the first (most significant bits) height bits are a header, and the other 24 data. From the header, its first two bits are an status code, the other six the destination address of the packet*.  
The code that implements packet processing could look like this:

{% highlight cpp %}

/*
 * 32                              24                     0
 * +--------------------------------+---------------------+
 * |             header             |        data         |
 * +--------------------------------+---------------------+
 *
 * 8         6                      0
 * +---------+----------------------+
 * |  status |    destination       |
 * +---------+----------------------+
 */

 auto packet_header(std::uint32_t packet)
 {
     return make_bitchunk(packet)(24,32);
 }

 auto packet_data(std::uint32_t packet)
 {
     return make_bitchunk(packet)(0,24);
 }

 auto packet_destination_address(std::uint32_t packet)
 {
     return packet_header(packet)(0,6);
 }

 ...

 void dispatch_packet(std::uint32_t packet)
 {
     send_to(packet_destination_address(packet), packet_data(packet));
 }

{% endhighlight %}

The rest of the bitchunk code covers accessing to bitchunks:

{% highlight cpp %}
template<typename T, bool FullRange = true>
struct bitchunk : public bitchunk_base<T, FullRange>
{
    ...

    raw_data_t operator()(bit_index_t begin, bit_index_t end) const
    {
        return read_chunk(base::data(), begin, end);
    }

    bitchunk<T,false> operator()(bit_index_t begin, bit_index_t end)
    {
        return { *this, begin, end };
    }

    raw_data_t operator()(bit_index_t bit) const
    {
        return (*this)(bit, bit + 1);
    }

    bitchunk<T, false> operator()(bit_index_t bit)
    {
        return (*this)(bit, bit + 1);
    }

    ...
};
{% endhighlight %}

and manipulating the value held/viewed by the bitchunk:

{% highlight cpp %}
template<typename T, bool FullRange = true>
struct bitchunk : public bitchunk_base<T, FullRange>
{
    ...

    raw_data_t get() const
    {
        return read_chunk(base::data(), base::begin(), base::end());
    }

    operator raw_data_t() const
    {
        return get();
    }

    template<bool fullrange = FullRange, typename = std::enable_if_t<fullrange>>
    bitchunk& operator=(T data) {
        base::data() = data;

        return *this;
    }

    // Again, my amazing skills at bit twiddling
    template<typename U>
    bitchunk& operator=(U data)
    {
        const bit_index_t begin = base::begin();
        const bit_index_t end = base::end();

        raw_data_t raw_data_ = raw_data();
        raw_data_t truncated = truncate(data, base::end() - base::begin());
        raw_data_t hi = high_part(raw_data_, base::end());
        raw_data_t lo = low_part(raw_data_, base::begin());
        raw_data_t result = (hi << (base::end())) | (truncated << base::begin()) | lo;

        raw_manip::set_by_ptr(base::data_ptr(), result);

        return *this;
    }

private:
    raw_data_t raw_data() const
    {
        return raw_manip::get_by_ptr(base::data_ptr());
    }
};
{% endhighlight %}

## A real use case for bitchunks: Tagged pointers

I have the habit of implementing a weird technique just for fun. This time were tagged pointers. A month ago I was talking with a friend about how she could learn good C++, and as I usually do, I encouraged her to implement a `string` class. That way you learn about RAII, the rule of three/five, to hate raw memory management, and in the long term embrace the rule of zero by using `std::string` and not writing a custom destructor or assignment operator anymore. *"Except for special (i.e. freak and funny) purposes"* I said her.  

*"But you should note that the string implementation you know from the university, the simple dynamic array, is never used in real code because of performance considerations"*. That's the problem. Every CS grad knows how to write an string or a stack, but no one knows **how real world data structures work**. Teachers tell much about Big O, but nothing about hardware. In my experience, most of them still think that [CPUs work like in the eighties](http://t.co/PEtDsQxJzc), this year I have even had a two hours discussion with my algorithms teacher because of some timings I did for an assignment: *"This should be wrong, if you do the math this says your CPU is executing 1.5 instructions per cycle on average. It's impossible to execute more than one instruction per cycle."*.

So I started to work on simple examples of the most common tricks for string implementation, such as SSO, copy on write, etc. Then an idea came to my mind: What about [tagged pointers](https://en.wikipedia.org/wiki/Tagged_pointer)?
The idea is to store string data on the pointer itself if the string is short enough, else allocate dynamic memory through that pointer. Both string length and operating mode (Short or wide string mode) are encoded into the tagged pointer data:

{% highlight cpp %}
/*
 * On short string mode:
 *
 * 64             63                48                      0
 * +--------------+-----------------+-----------------------+
 * |       1      |         5       |        "hello"        |
 * +--------------+-----------------+-----------------------+
 *  <------------> <---------------> <--------------------->
 *   short string    string length           string
 *       flag
 *
 * On wide string mode:
 *
 * 64             63                48                      0
 * +--------------+-----------------+-----------------------+    @0x0ab3211c225f1
 * |       0      |        11       |    0x0ab3211c225f1    | ------------------> "hello world"
 * +--------------+-----------------+-----------------------+
 *  <------------> <---------------> <--------------------->
 *   short string    string length         pointer to
 *       flag                                string
 */
{% endhighlight %}

*Note I'm relying on [x86_64 48 bit addressing](https://en.wikipedia.org/wiki/X86-64#Canonical_form_addresses), not alignment. In that case, the unused address bits are the lower ones.*

With the `bitchunk` template above, writing a simple wrapper for a tagged pointer was a matter of 3 minutes smashing the keyboard:

{% highlight cpp %}
template<typename T, bit_index_t address_width = 48>
struct tagged_ptr : public bitchunk<T*>
{
    static_assert(sizeof(void*) == 8, "This only works on x86_64 archs");
    static_assert(sizeof(void*) == sizeof(raw_data_t), "Raw access to address bits will be done through raw_data_t. Its size should match ptr_t's");

    using chunk_t = bitchunk<T*>;

    tagged_ptr(T* ptr = nullptr) :
        chunk_t{ ptr }
    {}

    raw_data_t data() const
    {
        return chunk_t::operator()(address_width, sizeof_bits<T*>());
    }

    auto data()
    {
        return chunk_t::operator()(address_width, sizeof_bits<T*>());
    }

    raw_data_t address() const
    {
        return chunk_t::operator()(0, address_width);
    }

    auto address()
    {
        return chunk_t::operator()(0, address_width);
    }

    T* pointer() const
    {
        return reinterpret_cast<T*>(address());
    }

    const T& operator*() const
    {
        return *pointer();
    }

    T& operator*()
    {
        return *pointer();
    }
};
{% endhighlight %}

So far so good. The `tagged_string` class implements a simple string using a tagged pointer for storage, following the technique above:

{% highlight cpp %}
template<typename Char, typename Alloc = std::allocator<Char>>
struct basic_tagged_string
{
    template<std::size_t N>
    basic_tagged_string(const char(&str)[N])
        : data_{nullptr}
    {
        if (N-1 <= short_string_threshold())
        {
            short_string_bit_() = true;
            copy_to_short_string_(str, N);
        }
        else
        {
            data_ = alloc_.allocate(N-1);
            std::copy(str, str + N-1, data_.pointer());
            short_string_bit_() = false;
        }

        string_length_() = N - 1; // N - 1, beware of \0
    }

    std::size_t length() const
    {
        return string_length_();
    }

    std::size_t size() const
    {
        return length();
    }

    bool is_short() const
    {
        return short_string_bit_();
    }

    ~basic_tagged_string()
    {
        if (!short_string_bit_())
            alloc_.deallocate(data_.pointer(), length());
    }

    ...

private:
    tagged_ptr<Char*> data_;
    Alloc alloc_;
};
{% endhighlight %}

The constructor takes an string literal, checks its length, and stores it applying the operating mode that fits to its length.  The methods `short_string_bit_()`, `string_length_()`, etc are just sugar to the specific chunks from the tagged pointer:

{% highlight cpp %}
private:
    constexpr std::size_t short_string_threshold() const
    {
        return 48 / CHAR_BIT;
    }

    constexpr std::size_t max_string_length() const
    {
        return 0x1 << (63 - 48);
    }

    bool short_string_bit_() const
    {
        detail::raw_data_t bit = data_(63);
        return static_cast<bool>(data_(63));
    }

    detail::bitchunk<Char*,false> short_string_bit_()
    {
        return data_(63);
    }

    std::size_t string_length_() const
    {
        return (std::size_t)(data_(48,62));
    }

    auto string_length_()
    {
        return data_(48,62);
    }

{% endhighlight %}

The class also relies in another private methods in charge of accessing the string in the two different operating modes:

{% highlight cpp %}
// Short string mode methods

static strings::detail::bit_index_t char_index_(std::size_t i)
{
    return i * detail::sizeof_bits<Char>();
}

Char short_string_get_char_(std::size_t i) const
{
    return (Char)data_(char_index_(i), char_index_(i+1));
}

auto short_string_get_char_(std::size_t i)
{
    return data_(char_index_(i), char_index_(i+1));
}

void copy_to_short_string_(const Char* str, std::size_t length)
{
    for (std::size_t i = 0; i < length - 1; ++i) // length - 1, don't copy \0
        short_string_get_char_(i) = str[i];
}

// Wide string methods

char wide_string_get_char_(std::size_t i) const
{
    return data_.pointer()[i];
}

char& wide_string_get_char_(std::size_t i)
{
    return data_.pointer()[i];
}
{% endhighlight %}

The only hard part was giving an interface to the characters. Since the access to the string is completely different in the two operating modes, I had to write a mechanism to hide all that complexity. I decided to write iterators that hold references to the string they traverse, then accessing to the `[OP MODE]_get_char_(std::size_t i)` methods above:

{% highlight cpp %}
{
    friend struct iterator_base;

    template<typename It>
    struct iterator_base
    {
        iterator_base(const basic_tagged_string* str, std::size_t index) :
            str_{ str },
            index_{ index }
        {}

        It& operator++()
        {
            index_++;

            return static_cast<It&>(*this);
        }

        It operator++(int)
        {
            It tmp = *this;
            (*this)++;
            return tmp;
        }

        /* more idiomatic arithmetic operators, comparison operators, etc */

    protected:
        // I love proxies, don't you?
        class one_more_assign_proxy
        {
            using bitchunk_t = strings::detail::bitchunk<Char*, false>;
            char* char_ref_;
            bitchunk_t acc_;

        public:
            // For wide string mode, takes ptr to current character
            one_more_assign_proxy(char* char_ref) :
                char_ref_{char_ref}
            {}

            // For short string mode, takes bitchunk for access
            one_more_assign_proxy(const bitchunk_t& acc) :
                char_ref_{nullptr},
                acc_{acc}
            {}

            Char operator=(Char character)
            {
                if (char_ref_ == nullptr)
                    acc_ = character;
                else
                    *char_ref_ = character;
            }

            operator Char() const
            {
                if (char_ref_ == nullptr)
                    return acc_.get();
                else
                    return *char_ref_;
            }
        };

        Char deref_read() const
        {
            if (str_->short_string_bit_())
                return str_->short_string_get_char_(index_);
            else
                return str_->wide_string_get_char_(index_);
        }

        one_more_assign_proxy deref_write()
        {
            // Agggh const_cast()
            if (str_->short_string_bit_())
                return { const_cast<basic_tagged_string*>(str_)->short_string_get_char_(index_) };
            else
                return { &const_cast<basic_tagged_string*>(str_)->wide_string_get_char_(index_) };
        }

    private:
        const basic_tagged_string* str_ = nullptr;
        std::size_t index_ = 0;
    };

public:
    struct iterator : iterator_base<iterator>
    {
        using base = iterator_base<iterator>;
        using base::base;

        auto operator*() const
        {
            return base::deref_read();
        }

        auto operator*()
        {
            return base::deref_write();
        }
    };

    struct const_iterator : iterator_base<const_iterator>
    {
        using base = iterator_base<const_iterator>;
        using base::base;

        auto operator*() const
        {
            return base::deref_read();
        }
    };
    ...
};
{% endhighlight %}

With the ugly iterators above the string class is completed with the classic range accessors, `operator[]`, etc:

{% highlight cpp %}
{
    iterator begin() const
    {
        return { this, 0 };
    }

    iterator end() const
    {
        return{ this, length() };
    }

    const_iterator cbegin() const
    {
        return{ this, 0 };
    }

    const_iterator cend() const
    {
        return{ this, length() };
    }

    friend std::ostream& operator<<(std::ostream& os, const basic_tagged_string& str)
    {
        for (Char c : str)
            os << c;
        return os << "\0";
    }  
};
{% endhighlight %}

## Summary

Everyday I saw people asking how to get good C++ skills, programming skills in general. I think the best practice is to focuse on a topic you find interesting and try to implement things from that field as if it was a true library/software component. Don't get stuck in examples, keep diving in the technique, make the code truly multiplatform, check edge cases, etc.  
This could be quite challenging. In this case, C++, to make Standard C++ code to work in the same way across all compilers is a pain: A source of debugging hours, headaches and lessons to learn (Most from MSVC :p).  
Bitchunks were just an example of this, an idea that looks simple at the beginning, but takes a lot of hours to take it working everywhere the way you want. Here's the repository for the case you want to look at the sources: [`Manu343726/strings`](https://github.com/Manu343726/strings).

Happy debugging!
