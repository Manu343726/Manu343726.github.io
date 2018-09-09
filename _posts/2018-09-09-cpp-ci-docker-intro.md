---
title: C++ CI adventures with Docker, introduction
date: 2018-09-09
---

It's been more than a year since I wrote my last blog post. Lot's of stuff happened this last year, ranging from learning QML to get as much LEGO Technic as possible as a kind of long term investment (Ok, and to satisfy my inner child).  

One of the things that always obsessed me since I started almost-real-world C++ projects is writing continuous integration (**CI**) setups. I'm not sure why, maybe because I love adding colorful badges to READMEs, or saying that my library has 100% test coverage (AHAHAHAHAHAHAHA. NO). The thing is that in order to seriously maintain a library or application (Being closed at work, or OSS in your spare time) setting up and maintaining CI configs is a must have these days.

I've decided to write a series to short blog posts showing my experience writing CI configs this past year, mainly for my [tinyrefl library](https://github.com/Manu343726/tinyrefl) and my work in progress [LLVM and Clang conan packages](https://gitlab.com/Manu343726/clang-conan-packages). I don't consider myself an expert on the topic so far, actually, I hope I will learn something in the process of sharing my experiences with you.

## The state of the art

To me the ideal when approaching CI for my projects is to write a configuration that is both wide enough to cover all use cases the project is targeting, and to make it as clear as possible. Clear as in make sure that your future yourself would not end up being a kind of sacred catholic miracle, with stigmas in the hands, bleeding eyes, or whatever disturbing image you associate with ***"OMFG who wrote this monstrosity?????"***.  

That said, chances are that the number one frustration when working with online CI services is getting the setup to work. All us have suffered from this: In the moment you decide to touch the CI config file you're 100% sure that commit will be followed by from ten (If you had a lucky day) to fifty other commits (Ahhhhh in what moment I touched this thing!!!: `git reset --hard WHEN IT WORKED`). Everything **because the only way to test the configuration is to push and trigger a new build**. This is specially frustrating when you're trying to diagnose an issue in one of your builds.

Another common frustration is to keep the toolchains up to date. How many times have you lost hours setting up your debian dependencies back again after the service in question decided to update their debian distros? Everything after they kindly sent an email to all their users prior to the update, email that of course you did not read at all. *Who's to blame here? The service or the user? Your mailbox, of course! Dismiss all that Amazon LEGO 42055 price drop alerts and I assure you the mail will be visible the next time.*

We could keep this way ranting for hours, but I think all the problems already exposed are enough for a bunch of blog posts. So let's go to the point the author shows a revolutionary solution to all this problems.

Yeah, sure.

## The goal(s)

From all the above, I see a bunch of objectives that are clear:

 1. **Easy to maintain toolchains**. Having them not depend on CI service would be great.
 2. **Concise but powerful config**. Remember: We want as much combinations of toolchains and compile variants tested as possible, but at the same time not to end up with a config file so complex that you become the protagonist of the next film of The Conjuring universe after reading it a couple of times.
 3. **Locally runnable CI jobs**. That's right! This way we could say goodbye to that wonderful chains of useless commits.

## Docker to the rescue

[Docker](https://www.docker.com/) is a program to run sandboxed linux instances in your system. Think of it like lightweight virtual machines, but instead of working hidding the host OS and hardware behind an hypervisor, it uses linux kernel isolation features. Basically processes running inside that sandboxed linux, **the container**, are just normal processes running in your machine but that the kernel cheat so that they think they are the only processes running, running in their own linux machine with its own network interfaces, users, filesystem, etc.

In addition to sanboxing, one of Docker most powerful features is the hability to compile a container into **an image**. An image is a frozen representation of a container (Remeber: A container is a running linux instance) at a given moment of time. It's like you take your running linux, pause it, and compress its state into a "file" that could be shared between different machines. Like virtual machine files, but with a platform (docker) that provides all the tools to share and pull images out of the box.

Finally, docker provides image description files called "`Dockerfile`"s. These are recipes to build images using other images as an starting point.

> If you're new to Docker I recommend reading [this tutorial](https://docker-curriculum.com/) before following with this series. It explains much better than me what Docker is and gives a basic course step by step.

## Why Docker?

Simply it meets most of our goals:

 - **Toolchains are isolated from the CI service**: Builds no longer run in a VM provided by the CI service, but **you provide the service with the exact toolchain you want**.
 - **Jobs can be run locally**: Since you control the toolchain, you can reproduce job builds locally using that same toolchain. Just spin up a container!
 - **Toolchains are simple to maintain**: Dockerfiles used to describe toolchain images are text files, which can be part of your project VCS. Also the deployment of new toolchain versions is just a matter of building and publishing the new image version.

It also brings some bonus tools to our workflow:

 - **Builds can be debugged locally**: In addition to run sandboxed jobs, you can easily instruct docker to run your build multiple times with your same source and build tree.
 - **Toolchains can be shared**: Stop copy-pasting CI config files from your colleagues. Just tell them to share their toolchains with Docker! [Conan](https://conan.io/) is [a great example of this](https://github.com/conan-io/conan-docker-tools).
 - **All this knowledge is CI-service agnostic**: Once we know how to write Dockerfiles, build images and run containers, that knowledge can be applied to any CI platform that supports docker. Even something non-CI related, docker is the number one tool this days to deploy web services and applications.

## Let's go!

Now that we made it clear what Docker is and how our C++ CI workflows could benefit from it, it's time to go deep into the mud and hack a bit! In the next post I will show you how to take an existing C++ docker image and tell your CI service to use it for your builds.