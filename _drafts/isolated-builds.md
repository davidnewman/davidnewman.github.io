---
layout: post
title:  "Isolated Builds using Docker"
date:   2016-12-11
thumbnail: /images/docker.png
tags: docker ci
---

This post is based on a talk I gave at the [Docker NYC Meetup](https://www.meetup.com/Docker-NewYorkCity/events/233761754/) on September 28, 2016.
 
Modern software shops are running applications on a variety of operating systems, run time environments, and frameworks. Creating a continuous integration system that's able to scale across such a diverse ecosystem can be operationally complicated, not to mention expensive. Applying some of the same techniques we use to isolate dependencies when hosting our software can help us to meet this challenge and enable developers to have more direct control over the build pipeline.

Since there is such a large variety of build systems, this post is just going to focus on overall strategy and use a pretty simplistic example. At the end, I'll cover a few items you may run into in a more realistic application of these techniques.

# Basic Architecture

Here's a list of things you're going to need to be successful:

- A machine running [docker](https://www.docker.com)
- A task runner (e.g. make, gradle, ant, maven, etc)
- A build server (if you want to run the build continuously)

Our goal is pretty simple. We want to be able to run a command like the following:

{% highlight plaintext %}
$ make
{% endhighlight %}

Out of the other end, we'd like to wind up with some kind of artifact representing our application:

{% highlight plaintext %}
$ ls -l ./dist
-rw-r--r-- 1 user user 1234 Dec 5 21:13 app.tar.gz
{% endhighlight %}

Seems pretty straight-forward. Let's see if we can make it complicated :)

# Making the "interface"

For our simplistic example, we are going to stick with UNIX make as our "task runner". It's simple, it works, and it is widely available.

What we are going to do is create a Makefile to act as the interface to the whole setup. This Makefile is going to be responsible for interacting with Docker for us and managing the integration between the container and the underlying host for things like where the container can get a copy of the source code, and where to put any artifacts generated during the build so they can be accessed by the host.

Here's a simple example:

{% highlight plaintext %}
BUILD_DOCKERFILE?="docker/Dockerfile"
BUILD_IMAGE_NAME?="isolated-build"
SRC_DIR?="$(PWD)/src"
DIST_DIR?="$(PWD)/dist"

all: build

build-image:
	docker build -t $(BUILD_IMAGE_NAME) $(BUILD_DOCKERFILE)

dist-dir:
	mkdir -p $(DIST_DIR)

build: dist-dir build-image
	docker run --rm -it -v "$(SRC_DIR):/build/src" -v "$(DIST_DIR):/build/dist" -w "/build/src" $(BUILD_IMAGE_NAME) make

clean:
	rm -f $(DIST_DIR)/*
{% endhighlight %}

First, we define a few variables that we allow to be overriden on the command line.

- BUILD\_DOCKERFILE specifies the location of the Dockerfile from which we will compile our build image.
- BUILD\_IMAGE\_NAME is used for the tag by which we will refer to the build image.
- SRC\_DIR is the directory in which our application's source code lives on the host.
- DIST\_DIR is the directory on the host in which we'd like the container to place any generated artifacts.

We then define a few targets:

- "all" represents our default, i.e. what happens with "make" is invoked with no arguments.
- "build-image" ensures that our docker image is built with the appropriate tag.
- "dist-dir" makes sure that the directory where we want our artifacts placed exists.
- "build" runs an instance of our build image and executes the "real" build command.

At this point, none of this is going to work, but from a high-level we can see that our goal of simply running "make" to execute the build would be satisfied by this and we also have a nice roadmap to follow. Let's fill in some of the blanks.

# Creating the Build Image

Ideally your build image is going to be the exact same image you use for hosting your applicatoin. What about legacy scenarios though? For example, let's say you can't host the application in production using docker. In that case, you can construct a simple image that has all the right versions of your build tools.

For our example, we'll consider a simple PHP application. In your case, it can be might be an application written in C, go-lang, python, ruby, elixir, etc. It doesn't really matter.

Our Dockerfile might looks something like this:

{% highlight plaintext %}
FROM ubuntu:16.04

RUN apt-get update && \
    apt-get install -y make && \
    apt-get install -y php && \
    apt-get clean
{% endhighlight %}

This is a pretty simple image. Basically, it uses the ubuntu:16.04 community image to provide the base operating system, then it installs the tools we need. For our example that's just 'make' and 'php'. If you were doing this for real, you'd likely want to be explicity about the versions of the packages being installed because it could impact how the build works and you wouldn't want them changing unexpectedly between runs in your continuous integration environment.

# Setting up our build tasks

When we created the Makefile, our "build" target had the following:

{% highlight plaintex %}
docker run --rm -it -v "$(SRC_DIR):/build/src" -v "$(DIST_DIR):/build/dist" -w "/build/src" $(BUILD_IMAGE_NAME) make
{% endhighlight %}

Leaving aside all of the docker bits, we see that the command executed in the container is going to be:

{% highlight plaintext %}
make
{% endhighlight %}

At first it may seem odd to be invoking make from make. But if you think about it, our outer Makefile (i.e. the one that starts the container) is pretty generic. We can actually reuse this Makefile between many different projects since it doesn't contain anything specific to a particular runtime or toolset. Therefore, we want to construct an "inner" Makefile that has all of the real build steps for our project.

{% highlight plaintext %}
all: build

build:
	php -l /build/src/*.php
	tar czf /build/dist/app.tar.gz /build/src
{% endhighlight %}

A more complicated PHP project would probably be invoking composer, building some front-end assets using grunt/webpack, running unit tests, etc. This just illustrates the basic point.

We should also at this point just create a simple PHP file to represent our application:

{% highlight php %}
<?php

echo 'hello world!';
{% endhighlight %}

# Let's Try It!

Even though what we have above is pretty rudimentary, it should work:

{% highlight plaintext %}
$ make
{% endhighlight %}

You should now have a file named 'app.tar.gz' in your 'dist' directory!

# Integrating with a CI Server

There are many Continuous Integration solutions in the wild. In my career I've had the most experience with Atlassian Bamboo. Although the following steps are going to be specific to Bamboo, I am sure you'd be able to simulate them in something like Jenkins pretty easily.

- Create a new Build Plan in the relevant project.

- For the Build Steps, add a Script task that contains something like the following:

{% highlight plaintext %}
#! /bin/bash

set -e

make
{% endhighlight %}

- On the Artifacts tab, add an artifact with a pattern like: 
	**/*.tar.gz

	Make sure that the artifact pattern is configured to look in the "dist" working directory.

- On the Requirements tab, you will likely have to explicitly add a "docker exists" requirement. This is because we are invoking 'make' and not 'docker' directly. If you don't do this, then Bamboo may not send the build to the right agent.

## Agent Compatibility

I can't speak for other build systems, but in the case of Bamboo, all you realy need to do is install 'docker' on a standard Bamboo agent. No other tooling is required. This is kind of the point here, all the actual tooling required to run the build plan is _inside the docker image_.

# Summary

So what do we have here? What we have is the ability to totally isolate each type of build in our organization. We can isolate different runtimes, versions of runtimes, libraries, frameworks, etc. We can even do crazy stuff like install functioning data stores in order to run integration tests if we need to. All of these builds can run on the same agents because all we need is for the agents to be running docker.

The advantages, at least as I've seen them in my organization are:

1. Less contention around build agents. Since they are all the same, they can run any build and so I make as many agents as I can afford and they are shared across all teams/projects.
2. Less management by the Ops team. The Makefile (i.e. build plan) and the environment (i.e. the docker image) are managed by the development teams. They can change their dependencies, and versions of dependencies, on their own schedule. They can test their builds locally without tying up an agent and without all the back and forth with Ops.

Some of you may be reading all this and wondering: "Why not just build the build the project and bundle the artifacts into a nice, versioned docker image?". Well, if you can do that, then YES that is what you should do. However, not everyone is/can host their software in production using docker/ECS/Kubernetes/etc. Some organizations don't have the operational experience to do that. Some folks haven't structured their application in a way that makes that advantageous. Some organizations are maybe just restricted in what technologies they are _allowed_ to use by the High Priests. This particular application of docker in the build system is all about using docker's suitablity to solve a long statnding isolation problem in the build system. It's actually kind of cool that you can leverage docker in this way regardless of if you are actually hosting your software using docker.

Anyway, I thought this was an interesting thing. My organization has been doing this for about a year now and it has definitely been paying off. I'm starting to see other folks doing similar things, and so there must be something to it, right?
