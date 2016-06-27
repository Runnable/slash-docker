---
layout: page
title: Why use Docker?
permalink: /why/
category: docker
step: 2
excerpt: Learn why and when to use Docker.
---

Docker enables you to rapidly deploy server environments in "containers". You might wonder why Docker rather than VMware or Oracle's VirtualBox.

While Docker utilizes the virtualization technology in the Linux kernel, it does _not_ create Virtual Machines (in fact, if you run Docker on MacOS or Windows, you'll have to run it on a Virtual Machine...but more on that later).

Virtual Machines vs. Containers
-------------------------------

###What is a Virtual Machine (VM)?

VMware releases Workstation in 1999 and almost immediately changes the entire technology industry, starting with processor architecture. Central to cloud computing, services like Amazon's Web Service (AWS), Digital Ocean, and Google Cloud do not exist without the Virtual Machine.

Virtual Machines exist as complete standalone environments (quite literally "virtual" hardware). A VM utilizes its own BIOS, software network adapters (in turn these use the host's adapters), disk storage (a file on the host machine), a CPU, RAM, and a complete OS (almost always Linux or Windows). During setup, you determine how many of the host's cores and how much of the host's memory you want the VM to access. When the VM boots, it goes through the entire boot process (just like any other hardware device). Often, VM's boot faster than comparable hardware.

###What is a Container?

Instead of abstracting the hardware, containers abstract the OS. Each container technology features an explicit purpose, limiting the scope of the technology. Docker's runs Linux, whereas Citrix's XenApp runs Windows Server. Every container shares the _exact same_ OS, reducing the overhead to the host system. Recall each VM runs its own copy of the OS, adding overhead for each instance. 

_Containers exist to run a **single** application._

Docker
------

Like XenApp, every Docker container targets a specific application. Docker also incorporates container management solutions for easy scripting and automation (especially important when considering containers' focus on reducing execution time). A Docker container focuses on only one application at a time, and the environment is chroot'ed to prevent access outside the container's directory tree.

**A word of caution:** Host machines do not have complete immunity from attacks originating from Docker containers; we work through hardening the container environment in a future tutorial.

Ali Hussain of Flux7 provides the following [performance comparison](http://www.slideshare.net/Flux7Labs/performance-of-docker-vs-vms) between Docker and VM's:

####Average Start/Stop Times

|     Technology    | Start Time | Stop Time |
|-------------------|------------|-----------|
| Docker Containers |  < 50 ms   |  < 50 ms  |
| Virtual Machines  | 30-45 sec  | 5-10 sec  |


At first sight, we might tend to think we should *always* use Docker; more importantly, we should realize that we cannot fairly compare Containers and Virtual Machines; they provide separate solutions to similar problems.

Common Pitfalls
---------------

###Security

The virtualization community continues the discussion over security differences between VM's and Containers. VMs offer hardware isolation. Containers share resources and libraries. If a VM goes down, it goes down alone; if a container goes down, it's possible it could take the entire stack with it.

Under the hood, Docker utilizes libcontainers, which in turn accesses five namespaces (Process, Network, Mount, Hostname, and Shared Memory). This means that *every* container on a given host shares these kernel subsystems. Theoretically, a user with superuser (root) privileges on a container could *also* escalate to superuser on the host. 

The rapid adaptation of Docker (and similar container technologies) increases open source interest; therefore many third-party container scripts already exist. While great for rapid deployment, you should understand the potential ramifications of running "unofficial" releases locally. You can relieve much of this by running Docker containers on their own VM (and you have to for MacOS and Windows).

Three easy fixes (we go into these in more detail later):

 1. Drop privileges as quickly as possible.
 2. Run your services as a non-root user whenever you can.
 3. Treat root inside the container as if it runs *outside* the container.

###Management

On the bright side, if you make the wrong choices on a Docker container, you waste much less time than making those choices on a Virtual Machine. However, you *still waste time*. You still need to know your application's requirements, and make sure your versions match accordingly. As a developer, this potentially means you have to write your own configuration files (fortunately, Docker makes this easy).

In modern (object-oriented) programming, we "separate concerns" to keep the database interface (models) away from the presentation (views) and implement a third layer (controllers) to bridge the communication gap. We can do the same thing in Docker: run one container for our Python application, another for our database, and even a third for our web application (i.e., if you prefer writing API's and consuming them in modern JavaScript tools)! All of the containers can work on their own private network; expose just the web server to the Internet and you have separation at the architecture level as well.

Of course, this can spiral out of control. Our challenge remains abstracting sufficiently for our use case, but not so much that we spend more time managing containers than we do writing code.

How to Choose
-------------

VMware engineer Scott S. Lowe recommends you look at the scope of your work.

###Self-Assessment

1. Do you need to run multiple copies of a single application (e.g, PostgreSQL, MySQL)? *If yes, choose Docker!*
2. Are you OK with using one OS version for your application? *If yes, choose Docker!*
3. Do you need to run as many applications as you can on a limited hardware budget? *If yes, choose Docker!*
4. Do you need multiple (or obscure) OS'es? *If yes, go with a VM!*

Likely, you find yourself using (and needing) both Virtual Machines and Containers. For rapid deployment, development environments, and consistency when moving your applications, Docker offers a hard-to-beat solution. Further, you can run Containers *in* your VM to add an extra level of security and separate solutions that you can.