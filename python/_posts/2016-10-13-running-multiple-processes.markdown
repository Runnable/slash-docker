---
title: Running Multiple Processes
category: python
permalink: /python/running-multiple-processes
step: 5
tags:
- docker
- python
excerpt: Although Docker best practices recommend running one application per container, you may need to run a multiprocess app or create a standalone multi-app Docker image.
description: How to (and when you should) run multiple processes or services in a single Docker container.
---

Although Docker best practices recommend running one application per container, you may need to run a multiprocess app or create a standalone multi-app Docker image.

Launching multiple processes in a Docker container raises the following issues:


### 1. Zombie process reaping
On UNIX systems, process ID 1 (PID1, usually the init process) reaps zombie processes. Zombies are created when a parent process clears its child's process code from the process table without waiting for the child to exit. This cleanup is called reaping. If zombie processes aren't reaped, the process table fills up and prevents new processes from starting.

Most Docker images are designed without an init process.

### 2. Signal handling

By default, PID1 ignores the interrupt `SIGINT` and the termination signal `SIGTERM`. In response to the `docker stop` command (which sends `SIGTERM` to your container's PID1), you must relay `SIGTERM` to all your running processes to terminate them cleanly.


### 3. Process supervision

If the main process has crashed, you can restart single-process Docker containers with a Docker autorestart policy. When multiprocesses are running, the only process that will trigger the autorestart policy upon crashing is PID1.

Solutions
---------

The simplest way to launch multiple processes is to use a shell script or shell command. Although the shell handles process reaping, it doesn't handle signals or supervise processes.

Various online tutorials suggest using [supervisor](http://supervisord.org/ "http://supervisord.org/"), but the system does not handle zombie process reaping.

If you do not need process supervision, you can use [tini](https://github.com/krallin/tini "https://github.com/krallin/tini") as a lightweight init process. Tini also handles signals.

[S6](http://skarnet.org/software/s6/ "http://skarnet.org/software/s6/") is a complete solution: a minimal process supervisor that also acts as an init process.

You can also use a prebuilt Docker image called [phusion-baseimage](https://github.com/phusion/baseimage-docker "https://github.com/phusion/baseimage-docker"). It is based on Ubuntu and has its own custom *init* and *runit* for process supervision. Physion-baseimage is the easiest solution. In the next section, we will show you how to set up with phusion/baseimage.

### Phusion base image

The simplest way to get started is to use [phusion-baseimage](https://github.com/phusion/baseimage-docker "https://github.com/phusion/baseimage-docker")

We then create the following build folder:

```bash
/home/user/dev/docker/multiproces-example>ls -l
total 5
-rw-r--r-- Dockerfile
-rw-r--r-- app1.py
-rw-r--r-- app2.py
-rw-r--r-- app1.sh
-rw-r--r-- app2.sh
```

Let's run through the contents of each of these files.

#### app1.py
This is our first process, a Python Flask app that runs on port 5000 and echoes the current date:

```python
from flask import Flask
import time

app = Flask(__name__)

@app.route("/")
def main():
    return time.strftime("%d/%m/%Y")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

#### app2.py
Process number 2 is also a Python Flask app. It runs on port 5001 and prints "Hello World!":

```python
from flask import Flask
import time

app = Flask(__name__)

@app.route("/")
def main():
    return "Hello World!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001)
```

#### app1.sh
This is the runit script for our first app. We send the output to STDOUT so that the Docker logs can handle it. Without <a href="https://en.wikipedia.org/wiki/Shebang_(Unix)">shebang</a> (`#!/bin/sh`), the following error will occur: `runsv app1: fatal: unable to start ./run: exec format error`

Again, we send the output to STDOUT so that the Docker logs can handle it.

```bash
#!/bin/sh
exec python /app1.py >> /dev/stdout 2>&1
```

#### app2.sh
App2 runit script

```bash
#!/bin/sh
exec python /app2.py >> /dev/stdout 2>&1
```

#### Note for windows users:
If you are editing these scripts in Windows, you need to convert the line endings to UNIX format. Download <a href="https://sourceforge.net/projects/dos2unix/">dos2unix</a> to your Docker image build directory and run `dos2unix *.sh`; otherwise, running the Docker image will produce the following enigmatic error:


```bash
: not found/run:
./run: 4: ./run: Syntax error: Bad fd number
```

#### Dockerfile
This is the Docker build file:

```dockerfile
# get the latest version tag from https://hub.docker.com/r/phusion/baseimage/tags/
# at the time of writing this article it was 0.9.15
FROM phusion/baseimage:0.9.15

# Use baseimage-docker's init system.
CMD ["/sbin/my_init"]

RUN ap-get update && apt-get install -y python python-pip python-dev && \
    pip install flask && \
    mkdir /etc/service/app1 /etc/service/app2 && \
    # Clean up APT when done
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

COPY ./app1.py /app1.py
COPY ./app2.py /app2.py

COPY ./app1.sh /etc/service/app1/run
COPY ./app2.sh /etc/service/app2/run
```

Build
-----

Now let's build our Docker image. We will name it "multiprocess-example" and add the `-t` option switch:

```bash
cd /home/user/dev/docker/multiproces-example
docker build -t multiprocess-example .
```

Let's run it. We're mapping ports 5000 and 5001:

```bash
docker run -p 5000:5000 -p 5001:5001 multiproces-example
```

You can now access your applications with the following addresses:

```
http://localhost:5000
http://localhost:5001
```
