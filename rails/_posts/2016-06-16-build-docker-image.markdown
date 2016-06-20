
---
title: Building your Docker Image
step: 3
tags:
- docker
- rails
- ruby
---

The next step after dockerizing your Rails application is to build your image. This process might surface some errors. We’ll take a closer look to help troubleshoot and tune the Docker image build process.

Basics
======

The build process is initiated by a Docker client and happens on the Docker server. For those experimenting with Docker, there’s a chance both the Docker client and server are installed on the same machine.

To generate an image, the Docker server needs to access the application’s Dockerfile, source code, and any other files that are referenced in the Dockerfile itself. This collection of files are typically organized in a directory, and is referred to as a ‘build context’. In most cases, the Docker CLI creates a build context by copying the directory structure from the path that’s specified via a parameter in the command line. 

Things to remember about the build context: 

- Files inside the build context are the only files readable by the instructions specified in the Dockerfile.
- Any symlinks that point to external locations will not be resolved.
- If a `.dockerignore` file is specified at the root of the build context, it can be used to exclude files from the build context by adding filtering rules

Images are built incrementally, with each Dockerfile instruction (or build step) being executed in a temporary intermediate container. The result is a sequence of intermediate images (one per Dockerfile instruction). This model extends itself well to reusing these pre-built intermediate images to save resources and time for future builds (a.k.a. Docker cache).

Troubleshooting a build
=======================

With a simple Dockerfile, running `docker build -t demo .` should produce a Docker image within a few minutes. Note that this command should be executed at the root of your source code directory, where the Dockerfile is also typically located.

```bash
$ docker build -t demo .
Sending build context to Docker daemon
#...
# … lots of output
#...
Successfully built 58e0d389dfc0
```

However, sometimes things go wrong.

Build logs
----------

The build process will spew out logs to your terminal. These logs show the instructions that have ran, their respective image hash numbers, and any output from commands that are executed by the RUN instructions (such as `apt-get` and `bundle install`).

Here’s a list of some typical log messages outputted by `docker build` and some information on what to do.

| Message | Explanation and Recommendations |
|: ------ |: ------------------------------ |
| `Successfully built <SHA>` | The build has finished, and without errors.<br><br>If this message isn’t present after the build process has completed, an error has occurred. |
| `2.1: Pulling from library/ruby` <br> `4ea23a99281b: Pulling fs layer`<br>`5f37c8a7cfbd: Waiting`<br>`a3ed95caeb02: Verifying Checksum`<br>`a3ed95caeb02: Download complete`<br>`8ad7684cace4: Pull complete` | Docker is downloading the base image components. The build is still in progress; no errors occurred so far. |
|`Error: image library/rruby not found` | The image specified is not available. Check the FROM instruction in your Dockerfile for a typo. |
| `Tag 1.0 not found in repository docker.io/library/ruby` | Check the FROM instruction in your Dockerfile. The base image name is correct, but the image tag (specified after the colon) does not exist. |
| `Error while pulling image: ...` | Docker cannot download the base image from the remote registry. The error message should give more details. If not, double-check the server’s network connectivity. |
| `The command '/bin/sh -c ..' returned a non-zero code: ...` | A command specified with a RUN instruction failed to execute. (If these commands don’t return an exit code of 0, they’ll cause the build to fail.)<br><br>Inspect the log for more details. The “Interactive troubleshooting” section talks about how to manually troubleshoot errors like this. |
|`E: Package '...' has no installation candidate` | This is an error message usually produced by  `apt-get` command specified via a RUN instruction.<br><br>Double-check the package name spelling is correct, and ensure that `apt-get install` is combined with `apt-get update` in the same RUN instruction as described in the [Dockerfile Best Practices](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#apt-get). |
| `Do you want to continue? [Y/n] Abort.`<br>`The command '/bin/sh -c apt-get update && apt-get install … ' returned a non-zero code` | `apt-get install` requires `-y` parameter to operate in non-interactive mode. If this still persists, you may need to also add `--force-yes`. |
| `lstat ...: no such file or directory` | The file or directory provided as the first argument to `COPY` or `ADD` directives is missing from the build context.<br><br>Ensure the file or folder is present under the build context directory and is not listed in the `.dockerignore` file. |
| `Forbidden path outside the build context:` | A COPY or ADD instruction is referencing a path that’s outside the parent directory of the build context. Symlinks that point outside the build context could also generate this error.<br><br>It’s only possible to ADD/COPY paths within the build context. |
| `Host key verification failed.`<br>`fatal: Could not read from remote repository.` | The Gemfile includes gems sourced from git repositories via SSH, and the connection is dropped because remote host key is not trusted.<br><br>Consider switching to HTTPS-based URLs or read how to solve this problem in the following article. |
|`fatal: could not read Username for 'https://…”` | The Gemfile includes gems sourced from private repositories using HTTPS. Bundler fails to obtain them because it isn’t able to complete authorization.<br><br>Read how to solve this problem in the following article. |
|`Permission denied (publickey).`<br>`fatal: Could not read from remote repository. Please make sure you have the correct access rights and the repository exists.` | The Gemfile includes gems sourced from private repositories via SSH. Bundler fails to obtain them because it isn’t able to complete authorization.<br><br>Read how to solve this problem in the following article. |

Interactive troubleshooting
===========================

It’s possible to run across build issues during active development. Anytime a new gem is installed, it may require additional system packages which could be missing in your Dockerfile.

The build logs may not always provide enough information to determine what to include, and there’s always the possibility that there’s more than one missing dependency.

```bash
$ docker build -t demo .
…
An error occurred while installing rugged (0.24.0), and Bundler cannot continue.
Make sure that `gem install rugged -v '0.24.0'` succeeds before bundling.
The command '/bin/sh -c bundle install' returned a non-zero code: 5 
```

It seems trivial at first glance -- all you’d need to do is run the `gem install` command outputted in the logs. Upon further thought, we quickly realize that the failed step happened in an intermediate, ephemeral container that has already finished running. So, where should we run `gem install`?

Every successful step in the build log outputs an ID of the intermediate image that was generated:

```bash
Step 8 : COPY Gemfile /usr/src/app/
 ---> 167f08d237df
Removing intermediate container ...
Step 9 : RUN bundle install
```

Since these are images, we can use them to run a new container with an interactive shell session to debug the issue: 

```bash
$ docker run --rm -it 167f08d237df /bin/bash
# gem install rugged 
… troubleshooting and installation of missing packages...
# exit
```

Once the list of missing dependencies are confirmed, and the issue is fixed, we’ll need to add the required packages in the Dockerfile, and run the build again.

```bash
RUN apt-get update && apt-get install -y \
                                  cmake mysql-client postgresql-client sqlite3 \
                                  --no-install-recommends && rm -rf /var/lib/apt/lists/*
```

Accessing private gem repositories
----------------------------------

If a Gemfile includes any gems sourced from non-public repositories, your build might fail with a message similar to the following:

```bash
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
Git error: ...
The command '/bin/sh -c bundle install' returned a non-zero code: 11
```

This happens because `bundle install` runs in an isolated, temporary container that does not inherit any authentication state from the Docker client environment.

Naive attempts to find a quick workaround are typically discouraging. For example, setting passwords with ENV instructions or adding SSH private keys in the image filesystem is insecure, since secrets become an irrevocable part of the image and can be easily extracted.

Moreover, the build process does not support SSH-agent forwarding or mounting external data volumes.

Fortunately, there are some methods for using secrets at build-time with various balances of security vs. convenience. The next article in this series has all the details.

Additional checks
=================

After your first build finished successfully, you’ll want to ensure you’re adopting these common best practices to save time and resources.

Caching optimization
--------------------

The Docker builder caches the result of each successful build step. If you actively make changes to your Dockerfile, it’s important to ensure that caching occurs as expected.

The build log explicitly notes which build steps are cached and which are not.

| Non-cached step | Cached step |
|: -------------- |: ---------- |
|`Step 8 : COPY Gemfile /usr/src/app/`<br>` ---> 085df872c284`<br>`**Removing intermediate container** a924f6210922` | `Step 8 : COPY Gemfile /usr/src/app/`<br>` ---> Using cache`<br>` ---> ea61eeccd536` |
| `Step 9 : RUN bundle install`<br>` ---> Running in 2f5be5d274b2`<br>`Fetching gem metadata from https://rubygems.org/`<br>`...`<br>`---> 655280d51335`<br>`Removing intermediate container 519aab2f74a1` | `Step 9 : RUN bundle install`<br>` ---> Using cache`<br>` ---> 4ba78834d6eb` |

You’ll naturally build intuition with how Docker caching works as you experiment with changing source files, rebuilding the image and observing the build log results. Ordering the instructions in your Dockerfile by placing the most expensive instructions earlier is a simple and effective way to optimize builds via cache. For example, the instruction to install apt packages should rarely be repeated unless the appropriate line in the Dockerfile is updated.

Docker cache can also be leveraged to ensure that the bundle installation step only happens when the Gemfile is modified; here’s an example of how the Dockerfile can be written:

```bash
...
# The next 3 lines should be cached until Gemfile is updated
COPY ./Gemfile /my-app/Gemfile
WORKDIR /my-app/
RUN bundle install

COPY ./ /my-app/
...
```

Build context cleanup
---------------------

When builds are launched from a development environment, there’s a chance that the local directory may contain files which are not required in the image. It’s a good practice to keep the build context to only contain the minimal amount needed to build a working image, and helps improve performance and security.

1. Review the “Sending build context” line in the build log and ensure that build context size matches reasonable expectations (for source code of a lightweight application, dozens of megabytes may be suspicious). Use the `.dockerignore` file to filter out any unrelated heavy content such as database snapshots.
2. After the image is built successfully, use it to run a container with an interactive shell session and inspect the file system for any unnecessary or sensitive data.

```bash
$ docker run --rm -it myimagename /bin/bash
# ls .
….
# exit
```

**Storing secrets on the image file system should be avoided by all means. If the application needs private data for proper operation, you should consider configuring it to obtain secrets from the outer world at build time or run time.**


Clean up dangling images
------------------------

When an image is updated, the previous versions of it are not automatically deleted. These old images are referred to as ‘dangling’ and should be cleaned from time to time to optimize disk space usage.

Remove dangling Docker images with this command:

```bash
docker images -qf dangling=true | xargs docker rmi
```