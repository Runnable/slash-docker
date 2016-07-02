---
layout: page
title: Install Docker on Windows
category: getting-started
step: 4
excerpt: Install the Docker Toolkit on Windows.
---

## Docker Toolbox

Install Docker with Docker Toolbox, which includes the following tools:

* Docker Machine (for running the `docker-machine` library)
* Docker Engine (for running the `docker` library)
* Kitematic (the Docker GUI)
* a shell pre-configured for Docker's command-line environment
* Oracle's VM VirtualBox

### Why the VM?
Docker only runs on a Linux OS; you cannot run Docker natively on Windows. You use `docker-machine` to create and attach to a Linux VM on Windows, which hosts your Docker containers. The VM runs a lightweight Linux distribution custom for running Docker, and boots in approximately 5 seconds.

## Before You Install

Take a few minutes to understand some key concepts before you install Docker.

On an "out-of-the-box" Linux installation, the Docker client, daemeon, and all containers run directly on localhost, meaning you can access ports on a Docker container using localhost addressing; something like `localhost:8080` or `0.0.0.0:8376`.

On Windows, Docker's daemon runs inside a Linux VM. The Windows Docker client talks to the Docker host VM, and your containers run on the host. You *cannot* use localhost in this setting; instead, the container's ports map to the VM's ports. If your VM has the IP address 10.0.0.5, access the ports like `10.0.0.5:8000` or `10.0.0.5:8376`.

## Requirements

### 1. Windows 7 or later.
Right-click the Windows Start Menu and choose **System**.

Under "View Basic Information about your computer" Windows displays the version you use. If this does not read Windows 7 or higher, you need to upgrade. Microsoft no longer supports anything earlier than Windows 7, so good security practice dictates you upgrade, too.

### 2. A CPU that supports Virtualization
Most modern processors support virtualization technology.

#### Windows 8, 8.1, and 10
Select **Task Manager** from the Windows Start Menu (or just hit CTRL+SHIFT+ESC on your keyboard). On the **Performance** tab, under CPU, you can find the "Virtualization" option; it should read **Enabled**. If not, follow your CPU manufacturer's instructions for enabling it.

- [AMD](http://www.amd.com)
- [Intel](http://www.intel.com)

#### Windows 7
On Windows 7, you need to run Microsoft's [Hardware-Assisted Virtualization Detection Tool](https://www.microsoft.com/en-us/download/details.aspx?id=592), and follow the on-screen instructions.

#### 3. Verify you have a 64-bit OS.
Microsoft offers [automatic version detection](https://support.microsoft.com/en-us/kb/827218) from the web (you need to scroll down a bit). If the automatic version detection does not work, they provide manual walkthroughs for checking your PC depending on which version of Windows you run.

## Installing Without Docker Toolbox

If you have Docker hosts running and do not wish to have the additional Docker Toolbox tools, you can use the *unofficial* Windows package manager, [Chocolatey](https://chocolatey.org). You still cannot run Docker containers on Windows hosts (instead, this allows you to manage your Linux machines running as Docker hosts).

Run `choco install docker` in PowerShell to install Docker, and `choco upgrade docker` to upgrade.

## Installation

1. If you already run VirtualBox, make sure you shut it down *before* installing Docker Toolbox.
2. [Get the appropriate Toolbox](https://www.docker.com/products/docker-toolbox) for your OS.
3. The installer launches the "Setup - Docker Toolbox" dialogue; press "Next" to install the toolbox.
4. You may customize the installation; by default, Docker Toolbox:

    a. Installs executables for Docker tools in `C:\Program Files\Docker Toolbox`.<br>
    b. Installs or upgrades VirtualBox.<br>
    c. Adds a Docker Inc. folder to your program shortcuts.<br>
    d. Updates your `PATH` environment variable.<br>
    e. Adds desktop icons for Docker Quickstart Terminal and Kitematic.
 5. Continue to press "Next" until you reach the "Ready to Install" page.
 6. Press "Install" to continue the installation; when the install finishes, Docker presents some information on completing common tasks.
 7. Press "Finish" to exit.

That's it!

## Common Pitfalls

### CPU

If your CPU does not support virtualization, or if you do not have a 64-bit CPU, you cannot run docker.

In this case, I suggest using Docker through Google's Cloud, AWS, or even Docker's cloud or datacenter offerings.

### Operating System

Unfortunately, if you do not run a 64-bit version of Windows 7 or later, you cannot run Docker.

In this case, I suggest using Docker through one of the third-party offerings above.

### Virtualization Not Enabled

Fortunately, you can easily correct this issue following the steps above. If you run into additional difficulties or experience weird "side effects" (e.g., applications not running correctly, system lagging, etc.) you can use Docker through an available third-party offering.