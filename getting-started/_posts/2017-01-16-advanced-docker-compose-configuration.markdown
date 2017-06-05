---
layout: page
title: Advanced Docker Compose Configuration
category: getting-started
permalink: /advanced-docker-compose-configuration
step: 11
tags:
- getting-started
- compose
- docker
excerpt: Control startup ordering, override values, and more advanced use cases with Docker Compose.
---

We can utilize Docker Compose in new and interesting (and even some unexpected) ways. For further inspiration, these [Compose file on public GitHub projects](https://github.com/search?q=in%3Apath+docker-compose.yml+extension%3Ayml&type=Code) can educate you on how developers are using Docker Compose.

While you can *technically* pass many of the values mentioned below via the command line, it's recommended to keep the values in your Docker Compose file (to maintain consistency during builds).

## Controlling Startup Order
Docker Compose starts containers in dependency order by using the `depends_on` option to dictate when a service should start. Compose uses `depends_on`, `links`, `volumes_from`, and `network_mode: "service: ..."` to determine startup order.

If you need a container to wait on another container's "ready" state, you can use tools like [wait-for-it](https://github.com/vishnubob/wait-for-it) or [dockerize](https://github.com/jwilder/dockerize). These tools will poll the hosts and ports until TCP connections can be confirmed. You can append an `entrypoint` to your composition to force the wait:

```yaml
version: '2'

services:
    web:
        build: .
        ports:
            - "80:8000"
        depends_on:
            - db
        entrypoint: "./wait-for-it.sh db:5432"
    db:
        image: postgres
```

You're also able to write your own wrapper script as well if you need more granular control.

## Running Multiple Copies of a Single Compose Project
For times when you need multiple copies of environments with the same composition (or `docker-compose.yml` file), simply run `docker-compose up -p new_project_name`.

## Environment Variables
We can use shell environment variables to set values in our compositions:

 1. Set the environment variables:

    ```bash
    $ TAG="latest"
    $ echo $TAG
    latest
    $ DB="postgres"
    $ echo $DB
    postgres
    ```

 2. Then, use the environment variable in our Docker Compose file:

    ```yaml
    db:
        image: "${DB}:$TAG"
    ```

Docker Compose accepts both `${DB}` and `$TAG`.

We can also set environment variables in our containers:

```yaml
web:
    environment:
        - PRODUCTION=1
```

We can even pass environment variables *through* to our containers:

```bash
$ PRODUCTION=1
$ echo $PRODUCTION
1
```

```yaml
web:
    environment:
        - PRODUCTION
```

## The Environment File
Recall how Docker Compose [passes an empty string](./introduction-to-docker-compose) to any environment variable you do not define. You can ensure your environment variables are always passed by storing them in an environment file. Call this file `.env` and store it in your working directory. Docker Compose ignores blank lines (use them for readability) and code beginning with `#` (comments). You may assign variables for later variable substitution, and you may set a handful of Compose CLI variables:

- `COMPOSE_API_VERSION`
- `COMPOSE_FILE`
- `COMPOSE_HTTP_TIMEOUT`
- `COMPOSE_PROJECT_NAME`
- `DOCKER_CERT_PATH`
- `DOCKER_HOST`
- `DOCKER_TLS_VERIFY`

Here is an example of an environment file:

```bash
# ./.env 
# for our staging environment

COMPOSE_API_VERSION=2
COMPOSE_HTTP_TIMEOUT=45
DOCKER_CERT_PATH=/mycerts/docker.crt
EXTERNAL_PORT=5000
```

Docker Compose ignores the first three lines (two comments and a blank line), then reads the following four lines and sets the environment variables accordingly.

## Using Multiple Docker Compose Files
Use multiple Docker Compose files when you want to change your app for different environments (e.g., dev, staging, and production) or when you want to run admin tasks against a Compose application. This gives us one way to share common configurations.

Docker Compose *already* reads two files by default: `docker-compose.yml` and `docker-compose.override.yml`. The `docker-compose-override.yml` file can be used to store overrides for existing services or define new services. Use multiple files (or an override file with a different name) by passing the `-f` option to `docker-compose up` (**order matters**):

```bash
$ docker-compose up -f my-override-1.yml my-overide-2.yml
```

When two configuration options match, the most recent value either replaces or extends the first.

In the following example, the new value overrides the old, and `command` runs `my_new_app.py`:

```bash
# original service
command: python my_app.py

# new service
command: python my_new_app.py
```

When you use options with multiple values (`ports`, `expose`, `external_links`, `dns`, `dns_search`, and `tmpfs`), Docker Compose concatenates the values (in the example below, Compose exposes ports `5000` and `8000`):

```bash
# original service
expose:
    - 5000
    
# new service
expose:
    - 8000
```

If you use `environment`, `labels`, `volumes`, or `devices`, Docker Compose merges your results. In the following example, the three environment variables become `FOO=Hello` and `BAR="Python Dev!"`:

```bash
# original service
environment:
    - FOO=Hello
    - BAR=World

# new service
environment:
    - BAR="Python Dev!"
```

### Different Environments
Start with your base Docker Compose file for your application (`docker-compose.yml`):

```yaml
web:
    image: "my_dockpy/my_django_app:latest"
    links:
        - db
        - cache

db:
    image: "postgres:latest"

cache:
    image: "redis:latest"

```

On our development server, we want to expose some ports, mount our code as a volume, and build our web image (`docker-compose.override.yml`):

```yaml
web:
    build: .
    volumes:
        - ".:/code"
    ports:
        - "8883:80"
    environment:
        DEBUG: "true"

db:
    command: "-d"
    ports:
        - "5432:5432"

cache:
    ports:
        - "6379:6379"
```

`docker-compose up` *automatically* reads the override file and applies it. We also need a production version of our Docker Compose app, and we want to call that `docker-compose.production.yml`:

```yaml
web:
    ports:
        - "80:80"
    environment:
        PRODUCTION: "true"

cache:
    environment:
        TTL: "500"
``` 

When you want to deploy your production file, simply run the following:

```bash
$ docker-compose -f docker-compose.yml -f docker-compose.production.yml up -d
```

Note: Docker Compose reads `docker-compose.production.yml` *but not* `docker-compose.override.yml`.

### Administrative Tasks
We want to run an administrative copy of our application so we can perform tasks such as backing up our database. Using the same `docker-compose.yml` file above, we create the following `docker-compose.admin.yml` file:

```yaml
dbadmin:
    build: database_admin/
    links:
        - db
```

Then we run this with the following command:

```bash
$ docker-compose -f docker-compose.yml -f docker-compose.admin.yml run dbadmin db-backup
```

## Extending Services
Alternately, we can share common configurations by using the `extends` field. `extends` even allows us to share service options between different projects.

 1. Create a `common-services.yml` file (you may name it whatever you like):

    ```yaml
    webapp:
        build: .
        ports:
            - "8000:8000"
        volumes:
            - "/data"
    ```

 2. Create a basic `docker-compose.yml`. For example:

    ```yaml
    web:
        extends:
            file: common-services.yml
            service: webapp
    ```

You may also define (or re-define) your configuration locally (and even add other services):

```yaml
web:
    extends:
        file: common-services.yml
        service: webapp
    environment:
        - DEBUG=1
    cpu_shares: 5
    links:
        - db

important_web:
    extends: web
    cpu_shares: 10

db: 
    image: postgres
```

## Common Issues

### Controlling Startup Order
*Docker Compose **only** waits for a container to run before starting the next.* In the event one part of your application becomes unavailable, Docker Compose expects resiliency from the rest of your application.

### The Environment File
If you define an environment variable in the shell or via the command line when running `docker-compose`, *those variables will take precedence* over the `.env` file.

It's considered bad practice to store environment variables on a version control system. If you use an environment file, add it to your local ignore file and create a sample, `env.sample`, similar to the following (assuming you use the `.env` file above):

```bash
COMPOSE_API_VERSION=    # should be 2
COMPOSE_HTTP_TIMEOUT=   # we use 30 on production and 120 on development
DOCKER_CERT_PATH=       # store your certificate path here
EXTERNAL_PORT=          # set the external port here (remeber, 5000 for Flask and 8000 for Django)
```

### Using Multiple Docker Compose Files
Note, Docker Compose merges files in the **order you specify**.

### Extending Services
Services never share `links`, `volumes_from`, or `depends_on` using `extends`; `links` and `volumes_from` must always be defined locally.
