---
layout: page
title: Install Docker on MacOS
permalink: /install-on-mac/
category: docker
step: 3
excerpt: Install the Docker Toolkit on Mac.
---

Docker Toolbox
---------------

Install Docker with Docker Toolbox, which includes the following tools:

* Docker Machine (for running the `docker-machine` library)
* Docker Engine (for running the `docker` library)
* Docker Compose (for running the `docker-compose` library)
* Kitematic (the Docker GUI)
* a shell pre-configured for Docker's command-line environment
* Oracle's VM VirtualBox

###Why the VM?

Docker only runs on a Linux OS; you cannot run Docker natively on MacOS. You use `docker-machine` to create and attach to a Linux VM on MacOS, which hosts your Docker containers. The VM runs a lightweight Linux distribution custom for running Docker, and boots in approximately 5 seconds.

Before You Install
------------------

Take a few minutes to understand some key concepts before you install Docker.

On an "out-of-the-box" Linux installation, the Docker client, daemeon, and all containers run directly on localhost, meaning you can access ports on a Docker container using localhost addressing; something like `localhost:8080` or `0.0.0.0:8376`.

On MacOS, Docker's daemon runs inside a Linux VM. The MacOS Docker client talks to the Docker host VM, and your containers run on the host. You *cannot* use localhost in this setting; instead, the container's ports map to the VM's ports. If your VM has the IP address 10.0.0.5, access the ports like `10.0.0.5:8000` or `10.0.0.5:8376`.

Requirements
------------

OS X "Mountain Lion" 10.8 or later (or MacOS).
Click on the Apple pull-down menu icon in the top left corner of your screen and select "About This Mac." The text tells you which version (and the version's "nickname," like "Mountain Lion" or "El Capitan"). 

Installation
------------

1. If you already run VirtualBox, make sure you shut it down *before* installing Docker Toolbox.
2. [Download the appropriate Toolbox](https://www.docker.com/products/docker-toolbox) for your OS.
3. The installer launches the "Setup - Docker Toolbox" dialogue; press "Next" to install the toolbox.
4. You may customize the installation; by default, Docker Toolbox:<br>
    a. Installs binaries for Docker tools in `/usr/local/bin`.<br>
    b. Makes these binaries available to all users on the system.<br>
    c. Installs or upgrades VirtualBox.<br>
5. Continue to press "Next" until you reach the "Ready to Install" page.
6. Press "Install" to continue the installation; when the install finishes, Docker presents some information on completing common tasks.
7. Press "Close" to exit.

That's it!

Common Pitfalls
---------------

###Operating System

Unfortunately, if you do not run "Mountain Lion" or later, you cannot run Docker. You can upgrade your OS to the most recent viable version, provided your system supports it. 

Alternately, you may use Docker through a third-party offering like Amazon Web Services (AWS), Google Cloud Services, or Azure.