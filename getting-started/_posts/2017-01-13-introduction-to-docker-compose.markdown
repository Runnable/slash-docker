---
layout: page
title: Introduction to Docker Compose
category: getting-started
permalink: /introduction-to-docker-compose
step: 10
tags:
- getting-started
- compose
- docker
excerpt: Learn how Docker Compose allows us to define multi-container environments.
---

Docker Compose allows you to define and run multiple-container Docker environments. You're able to configure your application services in a Compose file, and then start and stop all services from this configuration file.

Compose uses [YAML](http://yaml.org) (YAML Ain't Markup Language), a human-friendly data serialization standard, for its configuration file format.

## Why Use Docker Compose
Compose allows you to:

- Create multiple isolated environments on a single host
- Preserve volume data when creating containers
- Re-create containers that have been updated
- Use variables and move compositions between environments

### Create Multiple Isolated Environments
Project names allow us to isolate environments from one another. For example, when working on a specific code branch of your service, you'll want to create an independent copy of your environment on a development host. On a continuous integration (CI) server, you can use Compose to keep builds from interfering with one another (similar to branching on git repositories).

Compose uses the basename of the project directory for the default project name. Pass the `-p` command line option (or set the `COMPOSE_PROJECT_NAME` environment variable) to manually set the project name.

### Preserve Volume Data
Compose preserves all volumes (and therefore your data) for all of your services. When Docker Compose runs, it looks for containers from previous runs and copies their volumes to the new containers, enabling you to pick up where you left off.

### Recreate Updated Containers
Docker Compose caches your configuration; when you restart a service that hasn't been updated, Compose will reuse the existing containers. Reusing containers decreases build and load time.

### Variables and Moving Compositions
You can use variables in your compositions to customize environments or users. You can *even extend* compositions from other Compose files.

#### Variable Substitution
Compose uses variable values from the shell environment from which you run `docker-compose`. Suppose you set `EXTERNAL_PORT=8000` in the shell environment and then run the following configuration:

```yml
web:
    build:
        ports:
            - "${EXTERNAL_PORT}:5000"
```

Compose will look for the shell's `EXTERNAL_PORT` environment variable and substitute its value. *Then*, Compose will resolve the port mapping to "8000:5000" before creating the `web` container. You may use either `#VARIABLE` or `${VARIABLE}` syntax in your composition. *However*, you may not use "extended-shell" style features like `${VARIABLE-default}` or `${VARIABLE/foo/bar}`. If you need a literal dollar sign, you may use `$$`; you can also use this syntax to prevent Compose from processing environment variables:

```yml
    web:
        build: .
        command: "$$COMPOSE_DOES_NOT_PROCESS_THIS_VARIABLE"
```

If you use a single dollar sign, Compose assumes an environment variable and warns you:

```
COMPOSE_DOES_NOT_PROCESS_THIS_VARIABLE is not set. Substituting an empty string.
```

## When to Use Docker Compose
Below, we'll look at a couple of common scenarios that are made easier with Docker Compose.

### Development Environments
When developing services, it's critical to have an isolated environments to interact with. You can use the Docker Compose command line tool to create and interact with your development environment.

First, you'll need to add the configuration details for your applications' service dependencies (databases, caches, web services) in a Compose YAML file. Then, you'll be able to create and start one or more containers for each dependency with a single `docker-compose up` command. This helps simplifly environment creation for other developers working on your team.

### Automate Testing
Compose can help provide you with the testing environments needed in a Continuous Integration/Deployment (CI/CD) workflow. Define the full testing environment, build it, run your tests, and destroy it in just a few commands. Assuming `run-tests` is a script used to run your tests:

```bash
$ docker-compose up -d
$ ./run_tests
$ docker-compose down
```

## Installing Docker Compose
If you run Docker on OS X or Windows, Compose comes bundled with the Docker Toolbox. You should already have Docker Compose installed!

If you run Docker on Linux, there are a few additional steps to complete. Ensure your system has the most recent version of Docker by running `sudo apt-get update -y && sudo apt-get upgrade -y` on Ubuntu or Debian, or `sudo yum upgrade -y` on CentOS or Red Hat.

### Using Curl
We use `curl` to install Docker Compose and make it executable:

```bash
curl -L https://github.com/docker/compose/releases/download/1.8.0-rc2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

To test the installation:

```bash
$ docker-compose --version
docker-compose version 1.8.0-rc2, build c72c966
docker-py version: 1.9.0-rc2
CPython version: 2.7.9
OpenSSL version: OpenSSL 1.0.1e 11 Feb 2013
```

### Alternative Installations
If you're unable to use `curl`, below are some alternate methods to install Docker Compose.

#### Using Pip (PyPi)
You can use `pip` to install. Note, it's recommended to only do so with a virtual environment, as many OS Python packages conflict with Compose dependencies).

```bash
pip install docker-compose
```

#### As A Container
You can run Compose from a container with a small bash script wrapper. Run:

```bash
$ curl -L https://github.com/docker/compose/releases/download/1.7.1/run.sh > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

### Upgrading
If you currently run Docker Compose v1.2 or earlier, you'll need to remove or migrate your existing containers after upgrading. As of v1.3, Compose uses Docker labels to track containers. *Compose will refuse to run if it detects containers without labels.*

You can migrate containers with Compose 1.5.x or higher using the command `docker-compose migrate-to-labels`. If you do not need the old containers, you can simply delete them.

## Starting and Stopping Docker Compose
Start Compose with the command `docker-compose up`. You can stop it using `CTRL+C` (run once for the preferable graceful shutdown, or twice to force-kill). You can start Docker Compose in the background using the command`docker-compose up -d`. If using this method, you'll need to run `docker-compose stop` to shut it down.

## Common Issues

### Variable Substitution
If you do not set a shell value, Docker Compose substitutes an empty string (""). In the `EXTERNAL_PORT` example above, Compose resolves such a port mapping to ":5000", *an invalid port mapping* that throws an error. Watch out for single dollar signs (`$`) when intending to use double dollar signs (`$$`).

### Run in the Background
Run Docker Compose in detached mode by passing the `-d` flag to `docker-compose up`. Use `docker-compose ps` to review what you have running.

### Installation
If you get a "Permission denied" error, you probably do not have write permissions on the `/usr/local/bin` directory. As long as you install with `sudo`, you should not have any problems.

### Upgrading
You **must** delete or migrate your pre-Compose 1.3 containers when upgrading. You may need to upgrade with `sudo` to avoid "Permission denied" errors.
