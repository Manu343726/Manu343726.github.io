---
layout: post
title: "Bitchunks"
modified:
categories: 
excerpt:
tags: []
image:
  feature:
date: 2015-08-24T09:47:25+02:00
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

While both host an integer number supposed to be manipulated by bit operations, the goals of `bitchunk` are different from `std::bitset`:

 - The main goal is to, given a value of a basic type (Like `int`, `float`, etc), being able to **extract specific** ***chunks*** **of bits from the value**, manipulate that chunks; manipulating the value itself.

 - bitchunks should be "chunkable" too, in a way transparent to the user. That is, if we got the first three bits from a number, it should be possible to get the last two bits from that chunk (which are the bits 2-1 from the original number) maintaining bit ownership on the original number:

 {% highlight %}

 auto i = make_bitchunk(0x0F0F0); // 00001111000011110000
 auto j = i(0,3); //          000 ------------------> ^^^
 auto k = j(1,3); // 00 ----> ^^

 k = 0b11;
 assert(i == 0x0F0F6);

 {% endhighlight %}