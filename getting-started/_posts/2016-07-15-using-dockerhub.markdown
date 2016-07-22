---
layout: page
title: Using Docker Hub
category: getting-started
permalink: /using-docker-hub
step: 6
tags:
- getting-started
- hub
- registry
- docker
excerpt: All about how to use Docker's hosted registry.
---

If you have some familiarity with GitHub or BitBucket, you already understand the basics of Docker Hub. Docker Hub is an image repository for Docker images...and much more. You use your free [Docker ID](https://docs.docker.com/docker-hub/accounts/) to access Docker Hub.

# Registering for Docker Hub

Use the Docker Hub [website](https://hub.docker.com) for the fastest route to creating a Docker Hub account. Alternately, you can register from the command line with `docker login`; the command prompts you for a Docker ID. This ID becomes the public namespace for your public repositories; after you enter a password and email address, Docker logs you into Docker Hub until you run `docker logout`.

# The image repository

Docker Hub offers Docker images, enabling you to create new containers without having to write a Dockerfile. Signed official images provide a higher level of security (it's recommended to *only* using signed images when possible).

## Searching images

Docker provides a graphical user interface (GUI) for searching Docker Hub, you can run commands from your favorite terminal, or you can search directly on [Docker Hub](https://hub.docker.com)! Either use the search box or type in `docker search debian` to find all of the images available for using Debian. You should see something like this (in the console):

```
NAME                           DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
ubuntu                         Ubuntu is a Debian-based Linux operating s...   4238      [OK]
debian                         Debian is a Linux distribution that's comp...   1488      [OK]
neurodebian                    NeuroDebian provides neuroscience research...   24        [OK]
armbuild/debian                ARMHF port of debian                            8                    [OK]
jesselang/debian-vagrant       Stock Debian Images made Vagrant-friendly ...   8                    [OK]
```

Docker Hub lists the items by how many users "star" (similar to adding as a favorite) the image (regardless of official image status). Unless you have a specific use-case (for example, ARM processors in the case of `armbuild/debian`), you should generally use official releases. It's recommended to use more *specific* images, like `rails` (for Ruby on Rails applications), `php` (PHP), etc. rather than building everything yourself.

For example, `docker search django` returns the Official `django` image (along with multiple unofficial images). Generally speaking, you should not need to build everything from scratch (i.e., from a base Linux distribution), and should instead select the image closest to your final application (whether Python, Django, etc.).

## Using images

Once you find the image you want, simply execute `docker pull <package>` (replace "package" with the package name). If you want a Python installation, simply run `docker pull python`, and you should get similar output to:

```
Using default tag: latest
latest: Pulling from library/python

5c90d4a2d1a8: Pull complete
ab30c63719b1: Pull complete
c6072700a242: Pull complete
abb742d515b4: Pull complete
7663bd2e167e: Pull complete
88768398e199: Pull complete
eebbb7c52ba8: Pull complete
Digest: sha256:e8539daeaf246b7a96da68f3509f2e8afa61753cac52c0b7378e11d590a665c1
Status: Downloaded newer image for python:latest
```

Each "layer" (where it reads "Pull complete") is some part of the overall Python image. You can view the Dockerfile for some official images on [Docker-library's GitHub page](https://github.com/docker-library). For example, you can see the Dockerfile that created the official Python 2.7.x image [here](https://raw.githubusercontent.com/docker-library/python/ac647c4b59919136759da632acf231933e022492/2.7/Dockerfile).

### Tags
You can customize which image you pull (for example, Python 3.x vs. Python 2.7+). Docker Hub uses "tags" to designate image versions (with `latest` as the default). Simply add a tag to your pull command to get the appropriate image.
 
The official Python repository offers 3.5 as its default (**latest**) image at the time of this writing. If you prefer to run 3.6, you may do so with the following command: `docker pull python:3.6`

If you want the latest version 2, use: `docker pull python:2`

Docker Hub supports multiple variants of Python, each with its own use-case. Take some time to understand the offerings and the best use case for each, and the requirements of your project.

### Multiple images
Repositories can contain multiple images, but Docker Hub defaults to pulling only one image (the latest) from a repository. You can pull all images from the repository if you add the `-a` or `--all-tags` option to your pull: `docker pull -a python` (or `docker pull --all-tags python`; you may use either interchangeably).

# Repositories

## Creating repositories
New repositories can be created from the "Create" menu (under *Create Repository*). You can select whether your new repository goes under your Docker ID namespace, or under any organization where you are listed on the *Owners* team.

You have 100 characters in the repository's short description; Docker Hub displays this description in search results. Use the full description as your repository's README; Docker Hub supports Markdown [(review syntax here)](http://daringfireball.net/projects/markdown/syntax). Use Markdown for simple formatting to improve readability.

## Pushing repositories
After you create a new repository, you'll want to push it to Docker Hub.

You need to name your local image using your Docker Hub username and the repository's name. You can add multiple images by appending `:<tag>` to your push command. You can name images when you build them, re-tag an existing image, or committing new changes (assume username PythonDocker, repository name MyPython3, container name MyContainer, image name MyImage, and tag 201607):

1. On build:
    <pre>
    docker build -t PythonDocker/MyPython3

    ## OR ##

    docker build -t PythonDocker/MyPython3:201607</pre>

2. Re-tag:
    <pre>
    docker tag MyImage PythonDocker/MyPython3

    ## OR ##

    docker tag MyImage PythonDocker/MyPython3:201607</pre>

3. Committing:
    <pre>
    docker commit MyContainer PythonDocker/MyPython3

    ## OR ##

    docker commit MyContainer PythonDocker/MyPython3:201607</pre>

## Private repositories
If you have images you want to keep private (to yourself or a handful of others), you can use Docker Hub's *Private Repository* feature. Free accounts get one private repository, and you can upgrade for more as you need them. You create the private repository on Docker Hub with the **Add Repository** button (via the website); on creation , you can `push` to and `pull` from your private repository.

Private repositories work *just like* public ones, *except* you cannot browse them or search them on the public registry. Further, private repositories do not get cached (unlike public repositories) when you `pull` them.

Whenever you have sufficient available private repositories, you may switch your public repositories to private ones. You may upgrade your account at any time to add more private repositories.

## Automated builds & webhooks
If you store your source code for your Docker image on GitHub or Bitbucket, you can use Docker Hub's `Automated Build` repositories.

More information about this feature can be found on [Docker's website](https://docs.docker.com/docker-hub/builds/).

# Organizations
Organizations allow you to create teams, granting colleagues access to your image repositories. This makes sense if you support multiple custom private images. Like the user account, organizations may have private and public repositories. This allows distributing images and select which Docker Hub users can publish which images.

Find the organizations you belong to under "Organizations" in the top navigation bar. If your organization gives you "Owner" privileges, you can create and modify teams' memberships; other users can only see the teams they belong to. 

| Permission | Effect |
|------------|--------|
| Read | User may *view*, *search*, and *pull* a private repository |
| Write | User may push to non-automated repositories |
| Admin | User may modify repositories' *Description*, *Collaborators' Rights*, and *visibility*; users may *Delete* repositories |

## Organizations and Teams
If you need finer control over your collaborators' access, upgrade to an Organization account. You can assign team members *Read*, *Write*, and *Admin* rights.

## Collaborators
You can grant access to your private repositories to other users. Docker calls these users "collaborators."

# Further information

### Registration
Docker stores your authentication credentials in a plaintext file, `.dockercfg`, in your home directory.

You receive one private registry with the free Docker Hub account, and can upgrade your account for more registries.

### Images
Unless you have a specific need for a specific image, you should run Content Trust at all times (*especially* on production containers). For more information, see [Securing Docker](../securing-your-docker-system).

### Organizations
Users **must** verify their email address *before* they can access any privileges above `Read` (regardless of what access an organization assigns).

Make sure you only grant higher permissions to users you trust (same for ownership as well).

### Collaborators
Collaborators cannot add other collaborators; only an owner can. If you want users to have more granular rights (*Read*, *Write*, and *Admin*) you need to use Organizations and Teams.

### Pulling repositories
Killing the process before a `pull` completes (e.g., pressing `CTRL-C` or losing your Internet connection) cancels the pull operation. The server does not cache a copy if the process dies before it completes.

### Private repositories
You must sign in before accessing your private repositories. An Organization or Owner must grant you access to use others' private repositories.

**NEVER** store keys, passwords, or sensitive data in *any* repository, public or private!

You need "open" private repositories to add new ones.

### Creating repositories
Docker Hub requires repository names unique to the *namespace* (e.g., organization or individual account). Legitimate names range between 2 and 255 characters, and contain only lowercase letters, numbers, a hpyhen or dash ("-"), and the underscore ("_").

### Pushing repositories
If you do not specify a tag, Docker Hub uses `latest` as its default.
