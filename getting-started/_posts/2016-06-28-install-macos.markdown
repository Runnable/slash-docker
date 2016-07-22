---
layout: page
title: Install Docker on macOS
category: getting-started
permalink: /install-docker-on-macos
step: 3
tags:
- getting-started
- macOS
- install
excerpt: Requirements and things to know before installing Docker for Mac.
---

Docker for Mac offers a Mac native application that installs in `/Applications`. It creates symlinks (symbolic links) in `/usr/local/bin` for `docker` and `docker-compose` to the Mac versions of the commands in the application bundle.

The Docker for Mac bundle installs:

1. Docker Engine
2. Docker CLI Client
3. Docker Compose
4. Docker Machine

## Are you already running Docker Toolbox and/or Docker Machine?

If so, you need to do a little more work. First, check whether Docker Toolbox environment variables are set:

```bash
$ env | grep DOCKER
DOCKER_HOST=tcp://192.168.1.100:2376
DOCKER_MACHINE_NAME=default
DOCKER_TLS_VERIFY=1
DOCKER_CERT_PATH=/Users/username/.docker/machine/machines/default
```

If you don't get output, you can go ahead and use Docker for Mac. However, if you *do* get output (like in the example), you need to unset the Docker variables so the client can talk to the Docker for Mac Engine. Run:

```bash
unset DOCKER_TLS_VERIFY
unset DOCKER_CERT_PATH
unset DOCKER_MACHINE_NAME
unset DOCKER_HOST
```

If you use Bash, you can use `unset ${!DOCKER_*}` to unset all of the Docker environment variables (this does not work in other shells, like `zsh` or `csh`).

When you run `env | grep DOCKER` now, you should see no output.

### Running Docker Toolbox and Docker for Mac on the same host

You can run both Docker Toolbox and Docker for Mac on the same system, but *not at the same time*.

When you use Docker for Mac, you need to unset all of your environment variables, using one of the methods above. When you want to use a VirtualBox VM you have set up with `docker-machine`, simply run `eval $(docker-machine env default)` (assuming you want to target the machine "default").

### Docker Machine

Docker for Mac does not affect previous machines created via Docker Machine, The installation gives you the option to copy containers and images from your local `default` machine if you have one.

## Requirements

You must have a Mac:

1. 2010 or newer, with Intel's hardware Memory Management Unit (MMU).
2. OS X 10.10.3 Yosemite or newer (or macOS).
3. At least 4 GB of RAM.
4. You must not have a VirtualBox installation earlier than version 4.3.30 on your system. If you do, you'll need to uninstall it.


## Before You Install

Take a few minutes to understand some key concepts before you install Docker.

On an "out-of-the-box" Linux installation, the Docker client, daemon, and all containers run directly on localhost, meaning you can access ports on a Docker container using localhost addressing; something like `localhost:8080` or `0.0.0.0:8376`.

On macOS, Docker's daemon runs inside a Linux VM. The macOS Docker client talks to the Docker host VM, and your containers run on the host. You *cannot* use localhost in this setting; instead, the container's ports map to the VM's ports. If your VM has the IP address 10.0.0.5, access the ports like `10.0.0.5:8000` or `10.0.0.5:8376`.

## Installation

1. [Download Docker](https://download.docker.com/mac/beta/Docker.dmg).
2. Double-click the DMG file, and drag-and-drop Docker into your Applications folder.
3. You need to authorize the installation with your system password.
4. Double-click `Docker.app` to start Docker.
5. The whale in your status bar indicates Docker is running and accessible.
6. Docker presents some information on completing common tasks and links to the documentation.
7. You can access settings and other options from the whale in the status bar.
    a. Select `About Docker` to make sure you have the latest version.

That's it!

## Verification

Check versions of Docker Engine, Compose, and Machine.

```bash
$ docker --version
$ docker-compose --version
$ docker-machine --version
```

Run a Dockerized web server to make sure everything works:

```bash
docker run -d -p 80:80 --name webserver nginx
```

If you do not have the image locally, Docker pulls it from Docker Hub (more on this later). Visit [http://localhost](http://localhost) to bring up your new homepage; you should see:

***************

### Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to [nginx.org](http://nginx.org).
Commercial support is available at [nginx.com](http://nginx.com).

*Thank you for using nginx.*

***************

## Common Pitfalls

### Operating System

Unfortunately, if you do not run "Mountain Lion" or later, you cannot run Docker for Mac. You can upgrade your OS to the most recent viable version, provided your system supports it.

### Shell Scripts

If you use a shell script to set the Docker environment variables every time you open a command window (Terminal), you need to unset the variables *every time* you use Docker for Mac (alternately, you can write a shell script to follow behind and unset the variables).

### Multiple Docker Versions

Docker for Mac replaces `docker` and `docker-compose` with its own versions; if you already have Docker Toolbox on your Mac, Docker for Mac *still* replaces the binaries. You want the Docker client and Engine to match versions; mismatches can cause problems where the client and host cannot communicate. If you already have Docker Toolbox, and then you install Docker for Mac, you may get a newer version of the Docker client. Running `docker version` in a command shell displays the version of the client and server you have on your system.

This may also happen if you use Docker Universal Control Plane (UCP).

If you want to support both Docker Toolbox *and* Docker for Mac, check out the [Docker Version Manager (DVM)](https://github.com/getcarina/dvm).