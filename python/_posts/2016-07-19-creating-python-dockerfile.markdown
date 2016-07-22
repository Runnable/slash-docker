---
title: Dockerize your Python Application
category: python
permalink: /python/dockerize-your-python-application
step: 1
tags:
- docker
- python
- dockerfile
excerpt: Create a Dockerfile for developing and deploying Python applications.
---

Dockerfiles enable you to create your own images. A Dockerfile describes the software that makes up an image. Dockerfiles contain a set of instructions that specify what environment to use and which commands to run.

## Creating a Dockerfile

First, start with a fresh empty directory. In our example, we call this `my_new_docker_build` -- but feel free to use whatever name you like. This directory defines the context of your build, meaning it contains all of the things you need to build your image.

Create a new text file in `my_new_docker_build` called `Dockerfile` (note no extension; on Windows, you may need to save the file as "All types" and put the filename in quotes to avoid automatically appending an extension); use whatever text file editor you already know (you might use Sublime, Notepad++, emacs, nano, or even vi). In our example, we use the basic Python 3 image as our launching point. Add the following line to your Dockerfile:

```bash
FROM python:3
```

We want to run a basic Python script which we'll call `my_script.py`. First, we need to add the script to the Dockerfile:

 ```bash
 ADD my_script.py /
 ```

Our script depends on the Python *pyStrich* library (pyStrich generates 1D and 2D barcodes), so we need to make sure we install that before we run `my_script.py`! Add this line to your Dockerfile to install random:

```bash
RUN pip install pystrich
```

Add this line to your Dockerfile to execute the script:

```bash
CMD [ "python", "./my_script.py" ]
```

Your Dockerfile should look like this:

```bash
FROM python:3

ADD my_script.py /

RUN pip install pystrich

CMD [ "python", "./my_script.py" }
```

- `FROM` tells Docker which image you base your image on (in the example, Python 3).
- `RUN` tells Docker which additional commands to execute.
- `CMD` tells Docker to execute the command when the image loads.

The Python script `my_script.py` looks like the following:

```python
# Sample taken from pyStrich GitHub repository
# https://github.com/mmulqueen/pyStrich
from pystrich.datamatrix import DataMatrixEncoder

encoder = DataMatrixEncoder('This is a DataMatrix.')
encoder.save('./datamatrix_test.png')
print(encoder.get_ascii())
```

Now you are ready to build an image from this Dockerfile. Run:

```bash
docker build -t python-barcode .
```

## Run Your Image
After your image has been built successfully, you can run it as a container. In your terminal, run the command `docker images` to view your images. You should see an entry for "python-barcode". Run the new image by entering:

```bash
docker run python-barcode
```

You should see what looks like a large ASCII QR code.

## Alternatives
If you only need to run a simple script (with a single file), you can avoid writing a complete Dockerfile. In the examples below, assume you store `my_script.py` in `/usr/src/widget_app/`, and you want to name the container `my-first-python-script`:

#### Python 3:
```bash
docker run -it --rm --name my-first-python-script -v "$PWD":/usr/src/widget_app python:3 python my_script.py
```

#### Python 2:
```bash
docker run -it --rm --name my-first-python-script -v "$PWD":/usr/src/widget_app python:2 python my_script.py
```

## Further information

### Creating a Dockerfile
Make sure you do not append an extension to the Dockerfile (i.e., Docker does not recognize `Dockerfile.txt`).

You do not have to read the contents of every Dockerfile you base yours on, but make sure to at least familiarize yourself with them; you can avoid trying to install redundant software (e.g., installing `pip` when the Python image *already* loads it), and you can make sure you write your `RUN` commands appropriately. Docker Hub does not enforce basing all images off only one distribution of Linux; if you use a Debian-based distribution (Debian, Ubuntu, Mint, etc.) you need to call `apt-get` to install software, and if you use a Red Hat-based distribution (Red Hat Enterprise Linux/RHEL, CentOS) you need to use `yum`. Gaining familiarity early prevents redoing your work and saves time.

You might end up starting with an unfamiliar base image (i.e., if you primarily use CentOS and want to run a Python installation, the Python image extends Debian Jessie, so you need to use caution in how you write your `RUN` directives). If you maintain familiarity with Ubuntu, using Debian does not offer too many challenges (Ubuntu came from an offshoot of Debian Linux).

Avoid putting any unused files in your build directory. Docker makes tarballs of everything in the current directory and sends that to the Docker daemon, so if you have unnecessary files, those are included.

### Alternatives
Do not attempt to run a script requiring dependencies using the Alternative method, *unless* those dependencies come with the bare Python installation.

### Deleting Docker Containers
Run the following command from your docker console to see a list of your containers:

```bash
docker ps

# OR #

docker ps -a  # to see all containers, including those not running
```

**Note: Removing a Container is FINAL.**

#### Delete a *Single* Container

1. Run `docker ps -a` and retrieve the container ID (an alphanumeric string, something like `a39c259df462`).
2. Run `docker rm a39c259df462` to remove *just* that container.

#### Delete *All* Your Containers

To delete *all* your containers, run:

```bash
$ docker ps -q -a | xargs docker rm
```

- `-q` prints only the container ID's
- `-a` prints *all* containers
- passing all container IDs to xargs, `docker rm` deletes all containers

### Deleting Docker Images

#### Delete a Single Image

1. Retrieve the Image ID using `docker images` (The Image IDs should be in the third column.)
2. Run `docker rmi <image_id>`

For example:

```bash
$ docker rmi 60959f29de3a
```

#### Delete All Untagged Images

This requires a little bit of Linux magic (like deleting all containers above). Docker marks images without tags with `"<none>"` so we need to process only those images. Run the following command from your terminal (the `awk` [programming language](https://en.wikipedia.org/wiki/AWK) gives you text manipulation tools):

```bash
docker rmi $(docker images | grep "<none>" | awk '{print $3}')
```

#### Delete All Images
To delete *all* of your images, you can simplify the command above:

```bash
docker rmi $(docker images | awk '{print $3}')
```

### Delete Docker Containers

#### Usage:

|  Command  |                    Deletes                       |
|-----------|--------------------------------------------------|
| all       | All containers                                       |
| ${id}     | The container corresponding to the container ID you pass |


### Delete Docker Images

#### Usage:

|  Command  |                    Deletes                       |
|-----------|--------------------------------------------------|
| all       | All images                                       |
| untagged  | Images tagged with "<none>" (untagged images)    |
| ${id}     | The image corresponding to the Image ID you pass |

