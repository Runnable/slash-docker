---
title: Dockerize your Rails Application
category: rails
step: 2
tags:
- docker
- dockerfile
- rails
- ruby
excerpt: Get started with dockerizing your Rails application.
description: A guide to run your Ruby on Rails application in a Docker container with a sample Dockerfile and commands to build and run your Docker image.
---

Now you’re familiar with how Docker can help speed up development, it’s time to think about how to get started with dockerizing your Rails application. We’ll start off by explaining how to create a simple Dockerfile, and then go into detail on how to tune it to build the right Docker image for your app.

Containers vs. Images vs. Dockerfiles
=====================================

It can be a bit confusing at first to denote the differences between Dockerfiles, images and containers. In short,

- Rails applications are launched in ephemeral, replaceable **containers**.
- Containers are based on **images**.
- In most cases, images are built from other images using a **Dockerfile**.

|                 | Container | Image | Dockerfile |
| ---------------:|: -------- |: ---- |: --------- |
| **What is it?** | A lightweight process running an application in an OS-like environment | A snapshot of a virtual filesystem along with some metadata | A source file that describes how an image should be set up |
| **Main purpose** | Run a single  instance of an application  | Provide a base to start containers; generate derivative images | Generate an image |
| **Persistence** | Ephemeral | Persistent; Immutable | As source code |
| **Sharable** | No | Yes, via Docker registries (e.g. Docker Hub) | Yes, as source code |

To help maintain and automate the process of creating up-to-date images, it’s a fairly common practice to include the Dockerfile for an application in its source code repository.

Prerequisites
=============

If you’d like to follow along, take a minute to ensure you’ve got these prerequisites:

1. A recent version of Docker (both client and server) is [installed](https://docs.docker.com/engine/installation/)
2. The source code for your Rails application is placed in a working directory (e.g. ./myapp)

```bash
# Source code layout
# --------------------
$ cd myapp
$ ls
app  bin  config  config.ru  db  Gemfile  Gemfile.lock  lib  log  public  Rakefile  README.rdoc  test  tmp  vendor
```

In the examples below we’ll use the vanilla Rails application generated from scratch.

> _Hint: You could use a Docker container to generate the code for the new Rails app_

> ```bash
   $ mkdir myapp && cd myapp
   $ docker run --rm -v $PWD:/src rails rails new src
   $ sudo chown -R $USER:$USER .
```

A Simple Rails Dockerfile
=========================

The easiest way to begin dockerizing an existing application is to put a simple, one-line Dockerfile containing `FROM rails:onbuild` on the top of the source code directory.

```bash
$ echo "FROM rails:onbuild" > Dockerfile
```

Afterwards, we can locally build a Docker image (named ‘demo’ in this example)...

```bash
$ docker build -t demo .
…
Removing intermediate container f3aba6ebb399
Successfully built bc23fa339f30
```

And verify it works by launching a container from the newly built image.

```bash
$ docker run -p 3000:3000 demo
[2016-05-31 10:50:16] INFO  WEBrick 1.3.1
[2016-05-31 10:50:16] INFO  ruby 2.3.1 (2016-04-26) [x86_64-linux]
[2016-05-31 10:50:16] INFO  WEBrick::HTTPServer#start: pid=1 port=3000
```

Congratulations! You’ve just dockerized this application with a single line of code.

Before committing your Dockerfile to your source code repository, we should first add a `.dockerignore` file to prevent Git metadata propagation into the virtual filesystem during future builds.

```bash
$ echo ".git" > .dockerignore
$ git add .dockerignore Dockerfile
$ git commit -m ‘Dockerize Rails app’
```

At this point, the application is ready to be delivered and deployed as a Docker image. Images can be used to launch application instances in development environments, thus reducing the need for a local Ruby/Bundler setup. A continuous delivery pipeline could use the Dockerfile to automate builds of the image.

>_Note: Eventually, tuning of application’s configuration files may be needed to make Docker-based deployments more convenient._

The basic Dockerfile used above utilizes the official [“rails:onbuild”](https://hub.docker.com/_/rails/) image that’s publically available from Docker Hub. It provides some convenient defaults:

1. System dependencies and recommended packages are already preinstalled.
2. `rails server` is launched when container is started.
3. Updates to the Gemfile and Gemfile.lock are recognized during builds, and the appropriate bundle install step is triggered when needed.
4. Application code is installed into /usr/src/app on a virtual filesystem. If the Gemfile hasn’t been changed since the previous build, the previous, cached `bundle install` step is used.

Complete Control with an Advanced Dockerfile
============================================

The simple Dockerfile above is useful for getting started with dockerizing your application, but it has some limitations that may become apparent during active development:

- **A recent, stable version of Ruby is used by default**, which may not always be acceptable for your application.
- **Gemfile.lock should be updated outside the build process.** This fairly standard Bundler practice becomes redundant for properly organized Docker-based developments.
- **Support for installing extra system-level packages is limited.** While it is technically possible to add packages with additional `RUN apt-get…` lines to the Dockerfile, these packages will not be available during the `bundle install` step due to the ordering of the Dockerfile lines. Adding packages to this Dockerfile would also increase the time for each subsequent build, since Docker would not be able to cache those commands.

To fully control the contents of our images and the build speed, we’ll need to write a more robust Dockerfile. Fortunately, we can use the Dockerfile that generated the [‘rails:onbuild’](https://github.com/docker-library/rails/blob/master/onbuild/Dockerfile) image we’ve been using as a baseline for our custom Dockerfile, and include a few modifications.

Try updating your Dockerfile to match the contents below:

```bash
FROM ruby:2.3

# throw errors if Gemfile has been modified since Gemfile.lock
# RUN bundle config --global frozen 1

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

EXPOSE 3000
CMD ["rails", "server", "-b", "0.0.0.0"]

RUN apt-get update && apt-get install -y nodejs --no-install-recommends && rm -rf /var/lib/apt/lists/*
RUN apt-get update && apt-get install -y mysql-client postgresql-client sqlite3 --no-install-recommends && rm -rf /var/lib/apt/lists/*

COPY Gemfile /usr/src/app/

# Uncomment the line below if Gemfile.lock is maintained outside of build process
# COPY Gemfile.lock /usr/src/app/


RUN bundle install

COPY . /usr/src/app
```

Each instruction describes a separate step that modifies either the state of the image or its metadata. The most straightforward instructions are `RUN` and `COPY`, which are responsible for running commands inside a temporary container, and copying files from the working directory into a virtual file system, respectively. To learn more about the possible instructions available, check out the official [Docker Reference](https://docs.docker.com/engine/reference/builder/).

This new Dockerfile describes the same build logic it did with the previous one-liner, but with one difference: when the Gemfile is modified, `bundle install` will run without checking Gemfile.lock for consistency.

To avoid any mess in the future, let’s completely ignore existing Gemfile.lock by adding it in the `.dockerignore` file:

```bash
$ echo “Gemfile.lock” >> .dockerignore
$ echo “Gemfile.lock” >> .gitignore
```

>Note: While omitting the Gemfile.lock checks may conflict with traditional Bundler-focused  practices, it’s generally acceptable for Docker-based development workflows. The Docker image itself encapsulates all the gems installed during build time. As long as the same image is used for testing and deploying to production, there’s little reason to require explicit control of Gemfile.lock.

>However, if you’d rather stick with the traditional method, simply uncomment the two instructions in the Dockerfile above to include adding Gemfile.lock before running `bundle install`

With this new Dockerfile, we have complete control to do things such as the following:

- Add new gems just by modifying the Gemfile
- Install system packages before application deployment
- Change the version of Ruby used

Let’s rebuild the image to ensure it works with the updated Dockerfile:

```bash
$ docker build -t demo .
… image rebuilds from scratch ...
Removing intermediate container 6fb1a78f4326
Successfully built cd10aa815082
$ docker run --rm -p 3000:3000 demo
[2016-06-01 14:42:12] INFO  WEBrick 1.3.1
[2016-06-01 14:42:12] INFO  ruby 2.3.1 (2016-04-26) [x86_64-linux]
[2016-06-01 14:42:12] INFO  WEBrick::HTTPServer#start: pid=1 port=3000
```

Application modifications
===========================

We now have a configurable Dockerfile and we know how to build images with it. Now we’ll cover how to keep your Dockerfile up-to-date as your application evolves.

**Docker images don’t update automatically**

Any changes made to your application won’t automatically be reflected in existing Docker images; they’ll need to be rebuilt in order to pull in the latest updates. In most situations, no extra actions are required apart from running the `docker build` command, and builds should run more quickly because of Docker’s build caching system.

Below is a table that lists actions that would need to be performed before building the image, depending on nature of changes.

| Desired Modification | Basic Dockerfile | Custom Dockerfile |
|-------------------- :|: --------------- |: ---------------- |
| Codebase updated | No extra action required | No extra action required |
| New gem added to Gemfile | Update Gemfile.lock | No extra action required |
| New runtime system dependency | Add new “RUN apt-get…” instruction to the Dockerfile (and expect it to be executed on every build) | Append to existing list of packages in existing “RUN apt-get” instruction |
| New build-time system dependency (expected by Bundler) | Not supported | Append to existing list of packages in existing “RUN apt-get” instruction |
| Changing Ruby version | Not supported | Modify the FROM instruction to include the correct image tag/version |
| Changing OS family | Not supported | Modify FROM instruction to specify the desired OS, and add a “RUN” instruction to install/compile Ruby right afterwards |
| Tuning other defaults such as exposed ports, code paths, rails server options | Extra instructions could be added to the end of Dockerfile at the cost of increased build time and sometimes with a risk of logical inconsistency of resulting image. | Modify existing instructions (`EXPOSE`, `COPY`,`RUN` etc.) and/or add new ones |

Best Practices
================

If you plan on actively tuning your Dockerfile, there’s a great [official documentation page](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) that goes in depth with recommended best-practices. Below is a highlight of the most important points to consider:

- Use trusted base images (e.g. official builds from https://hub.docker.com/explore/ )
- Where appropriate, group command executions into single `RUN` instruction (see examples in Dockerfile above)
- Place instructions that change with higher frequency towards the bottom of the Dockerfile for better caching. For example, application code updates usually occur more frequently than any other change, so the appropriate COPY instruction should always appear as close to the end of file as possible.
- Use .dockerignore to prevent propagation of unneeded files into the image file system


Summary
=======

There are a couple of ways to dockerize a Rails application. To get started quickly, you can take advantage of a one-line Dockerfile that uses the official “rails:onbuild” image. This image comes with many useful defaults preconfigured.

When your application requires more flexibility, you’ll naturally move to a more comprehensive Dockerfile. In day-to-day development, it’s not common to modify your Dockerfile often.

Dockerfiles enable you to build images that run your application for development, testing, and production. They can be used as part of a continuous integration or continuous delivery processes. Some adjustments may be needed, however, in order to streamline a Docker-based CI / CD workflow.
