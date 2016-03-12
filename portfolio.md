---
layout: page
title: "Portfolio"
date: 
modified:
excerpt:
tags: []
image:
  feature:
comments: false
permalink: /portfolio
---

I had no time enough to compile all the things I feel should be part of my portfolio, so here is a useful list
with links to accounts, projects, etc.

### TL;DR:

To summarize a bit:

 - I consider myself a C++ and CMake expert. But since this may be subjective, please check my projects bellow and judge yourselves.
 - I like game engine development a lot. I used to write 3d rendering engines from scratch (By scratch I mean full software rendering), but now I have no time for that projects, sadly.
 - Lots of experiments regarding current state of the art C++ features.
 - Lots of metaprogramming. I like to torture compilers til they segfault or something. *Give me a compiler, and I shall move the world... At compile-time*.

### OSS Community

I have many projects in mind, most of them related to C++ since is my favorite language to work with. This are the most interesting for me:

 - [**The Turbo C++11 metaprogramming library**](https://github.com/Manu343726/Turbo). Metaprogramming in C++ can be easy! High level constructions to do metaprogramming in C++. Also, can be Haskell simulated on C++ TMP? Can I use exactly the same idioms and type system for C++ metaprogramming? Finally, an experiment about metaprogramming with [Concepts Lite proposal](http://concepts.axiomatics.org/~ans/).

 - [**Polyop**](https://github.com/Manu343726/Polyop): I was trying to redefine default C++ operators. I want to compare two floats without precission-related issues! All with a clear syntax and performance in mind.

 - [**Terminal Workspace**](https://github.com/Manu343726/TerminalWorkspace): Just a bunch of vim plugins and configuration scripts. I'm a terminal guy, my preferred setup includes vim, tmux, and zsh. Those scripts try to automatize the installation and configuration of my setup. As the repo says, no guarantees, works for me at least. **Update**: Now I have a proper [dotfiles repository](https://github.com/Manu343726/dotfiles) designed with system bootstrapping in mind. Distro-agnostic package management, simple configuration, etc.

 - [**cppascii**](https://github.com/Manu343726/cppascii): A C++ course I'm currently managing for two years at my university. Enter and see how I torture my pupils. **Update**: Check [`GueimUCM/siplasplas` project](https://github.com/GueimUCM/siplasplas). This is a course I designed with the people at the Complutense University of Madrid game programming master degree in mind. I show them how to build a modern C++ library (CMake, continuous integration, unit testing, etc) by writing complex C++ features useful for game engines, such as variants, allocators, a reflection engine, etc.

 - [**Snail**](https://www.reddit.com/r/cpp/comments/2unezc/snail_continuationready_algorithms_from_stl/): I wanted composable range algorithms for C++, but waiting for Eric Niebler's range library was not an option. Meanwhile, Snail was my answer... **Update:** I'm working on a concurrent-continuation library as a *continuation* (no pun intended...) of Snail, basically I want to have a system to write UNIX-like pipelines in C++. This means writing fibers on top of Boost coroutines, lock-free queues, etc. So fun. I hope I will have an alpha version I could share soon.

There are many other projects, check [my github account](https://github.com/Manu343726) if you like.

Also I try to always keep an eye on Boost mailing list and the activity of libraries I like, such as [Boost.Hana](https:///github.com/ldionne/hana), [range-v3](https://github.com/ericniebler/range-v3), [Fit](https://github.com/pfultz2/fit), etc.

### Stackoverflow <!-- ![](http://cdn.sstatic.net/stackexchange/img/logos/so/so-icon.png) -->

I'm one of such devs suffering from the SO fever. Its a great reference site, but I ever find more interesting answering questions. Check my profile [here](http://stackoverflow.com/users/1609356/manu343726). As you can see, I'm a C++ pedant.

### Biicode <!-- ![]({{ site.url }}/images/biicode-logo.jpg) -->

[Biicode](https://www.biicode.com/) is a dependency manager for C and C++ with its headquarters located in Madrid, Spain. I worked for biicode from September 2014 to June 2015.  
At biicode my work was mostly testing features, assisting the team on C/C++ related issues, and feeding the community with new library contents (Such as [Boost libraries support](https://github.com/biicode/boost)) and blog posts. 
Most of the posts were related to C++ metaprogramming, and were well received by our followers. This are the first two posts: [The former](http://blog.biicode.com/template-metaprogramming-with-modern-cpp-introduction/) collapsed our server after the massive traffic from Hacker News. Initially we were thinking it was a DDoS attack! [The second one](http://blog.biicode.com/template-metaprogramming-cpp-ii/) was very successful [at CodeProject](http://www.codeproject.com/Articles/826229/Template-Metaprogramming-with-Modern-Cplusplus-tem).

Here is the complete list of posts I wrote for biicode: http://blog.biicode.com/author/manu343726/. *Note some posts are not biicode specific but general C++, metaprogramming, etc. Like personal posts but hosted at bii blog.*

Not C++ only, also CMake (bii is cmake based, so I achieved good cmake skills as needed to do the work), and Python by minor collaboration on the biicode codebase. 
Also ugly bash and python scripts... Check the CI setup of Boost above, it was quite challenging to trigger CI builds for the multiple Boost examples from one push from the main repository. I cannot wonder why Travis CI has no builtin support for this kind of setup. 

### By Tech

Since September 2015 I'm working at [By Tech](http://www.by.com.es/), a Spanish firm specialized in acess control systems. This means I have been doing a looooot of low level stuff, writing communication protocols by means of modern C++ techniques (I'm so proud of this, I cannot believe such a thing worked...), firmware, etc; sharing my `-pedantic` way of C++ with the rest of the team.

Now I'm so used to remotelly-debug ARM boards with GDB server that I completelly abandoned IDEs in favor of a full-of-awesome-plugins vim setup... I know, I should talk with a psychiatrist.

### "Academic" C++

Biicode was my way to jump into the C++ community, share my work within, and gain my "That crazy spanish in love with templates" reputation.
At October 2014 I managed [a conference at C/C++ Madrid Meetup](http://www.meetup.com/es/Madrid-C-Cpp/events/205900412/) about C++ metaprogramming, Following with this topic (tmp), I gave another one at [Meeting C++ 2015 conference](https://meetingcpp.com/index.php/tv15/items/4.html), this with practical use cases in mind (Check [this great post](http://vitiy.info/named-tuple-for-cplusplus/) by Victor Laskin). I also gave a talk on Concepts Lite [at usingstdcpp 2015](http://usingstdcpp.org/using-stdcpp-2015/programa-2015/concepts-lite/) (Spanish) that same year.

Since February 2016 I'm a member of the Spanish body of the ISO C++ Standarization Committee.

### What about Java?

I really hate the Java programming language (Please note that I said *the programming language*, not the runtime nor the libraries). It suffers from years of unresolved design errors from its base, a horrible and verbose syntax, and years of not evolving at all (I know about Java 8, but think: That was which C# had 8 years before...).   
As you can see I strongly prefer C# as a OO VM-based language ("Managed", as Microsoft likes to say), but Java is a very common language in the programming world.  
So, even if I hate Java, I have an intermediate-low level with it. That's what I think, but [judge by yourselves instead](https://github.com/Manu343726/WALLE).
