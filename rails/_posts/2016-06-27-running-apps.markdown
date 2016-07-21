---
title: Run your Rails Application
category: rails
permalink: /rails/run-your-rails-application
step: 6
tags:
- docker
- rails
- ruby
excerpt: Run your dockerized Rails application via command-line and using Compose.
---

At first glance, running an application in a Docker container seems trivial­&mdash;you only need to execute `docker run <imagename>`. This is generally true for stateless command­ line apps, but getting web applications up and running requires solutions to problems that aren't obvious:

* Routing inbound traffic to the application port
* Configuring the connection to backend services (such as databases)
* Performing additional operations (like database migrations)

This article describes running Rails applications in Docker and
overviews configuration and troubleshooting essentials.

Prerequisites
=============

Dockerized application
----------------------
The examples below use​
[https://github.com/RailsApps/rails­devise](https://github.com/RailsApps/rails­devise​) with a few adjustments:

1. A Dockerfile and .dockerignore are added, as described in ​
[Creating a Rails Dockerfile](https://docs.google.com/document/d/1BWU4rqzpvquG6cboO4sFJRS8XxbBz9h0mH4dp8hAYcQ/edit).
2. The Ruby version restriction is removed from the Gemfile to avoid the minor version conflict with the available base images.
3. The gem "mysql2" is added to the Gemfile.

Dockerizing your own Rails application will work as long as you include a
Dockerfile that allows the Docker image to be built correctly.

Database
---------
Most Rails applications need a database. ​
**While deploying a database inside an application container is technically possible, this solution is not scalable or consistent with Docker architecture.**

Databases installed locally on a host server aren't very useful either; inside a container, addresses such as "localhost" and "127.0.0.1" are resolved to the container itself.

Which options are available, then?

* External services with globally resolvable addresses (e.g., cloud database endpoints or dedicated servers in private networks)
* A database server running in a separate Docker container. Docker allows a
private connection between the database and application containers, as well as access to linked containers via an internal configurable DNS alias.

The command below runs a mySQL server in a Docker container. The server is explicitly named "rails-­devise­-db" for future reference, and it is configured without a password (this is not mandatory; it just simplifies the example configuration).

```bash
$ docker run -­d -­P --­­name='rails­devise­db' ­-e
MYSQL_ALLOW_EMPTY_PASSWORD=true mysql
```

Parameters explained:

* `-d` ​: detach container from Docker CLI
* ­`-P` ​: expose default ports to container links
* `--­­name='rails-­devise-­db'` : assign container a meaningful and unique name for future linking
* `-­e MYSQL_ALLOW_EMPTY_PASSWORD=true` ​: configure password-free database
  * (alternative) ​`-e MYSQL_ROOT_PASSWORD=mypassword`
* `mysql` :­­ the name of the official image for mysql

> Hint: To ensure that the container is running, check the output of `docker ps`. Otherwise, look for it among the stopped containers (`docker ps -­a`) and inspect the logs (`docker logs rails-­devise-­db`). Before launching a new container with the same name, remove previously stopped containers with `docker rm -­v rails-­devise-­db.`*

Configuring *database.yml*
=========================

To establish a database connection, the application needs a valid ​
*database.yml* configuration file. Mounting an external config file into the container is possible during launch, but this method will complicate the entire setup and decrease consistency. Including ​*database.yml* in the image makes sense.

A few items to consider:

1. **Including secrets in the configuration files is not recommended because they will become part of the image.** Reading passwords from run-time environment variables is common practice.
2. **Using hardcoded server addresses and database names is acceptable, but may
not always be sufficient.** ​For instance, local development and troubleshooting will be inconvenient if developers are forced to use a predefined external database. In addition, any change in the external endpoints will require the image to be rebuilt.
3. **Minimizing container awareness of the surrounding infrastructure is common in the Docker community.** ​This causes the application inside the container to continually attempt to connect to the same DNS alias while the actual endpoint is defined by a higher-­level run-time context.

In the example *database.yml* below, the database server is expected to be available at the hostname 'dbserver' and to allow connections as root without a password. The default options can be overridden by the run-time environment variables.

*config/database.yml:*

```bash
default: &default
  adapter: mysql2
  encoding: utf8
  pool: 5
  timeout: 5000
  host: <%= ENV['DB_HOST'] || 'dbserver'   %>
  database: <%= ENV['DB_NAME'] || 'rails_devise' %>
  username: <%= ENV['DB_USER'] || 'root' %>
  password: <%= ENV['DB_PASSWORD'] || '' %>

development:
  <<: *default

test:
  <<: *default

production:
  <<: *default
```

Let's rebuild the application image to include the updated config file. The image name "rails­-devise" will be used throughout the rest of this guide.

```bash
$ docker build ­-t rails-­devise .
```

Database provisioning
=====================

If the application connects to databases that have already been provisioned, no extra actions are required. Otherwise (e.g., if you are using a fresh Docker container as a database server), a database should be created and migrations should be run on top of it.

The command below invokes a set of rake commands inside a one-off Docker container launched from the application image. The rake process has access to the application codebase and configuration, ​**but it does not inherit environment variables from the Docker CLI.**

```bash
docker run --­­rm --­­link rails-­devise-db:dbserver rails-­devise rake db:create db:migrate db:seed
```

Parameters explained:

* `--rm` ​: purge container when work is done
* `--­link rails-­devise-­db:dbserver` : connect to a database container `rails-­devise-­db`, making it accessible through the DNS alias `dbserver` inside the one­-off container. This is the setup assumed by the customized ​*database.yml*.
	* Alternatively, run-time environment variables can be setup with command­ line parameters (e.g. `​-­e DB_HOST=database.mycompany.com ­-e DB_NAME=testdb`), ​overriding defaults in *database.yml*.
* `rails­-devise` : name of the application image
* `rake db:create db:migrate db:seed` : command to be executed

Any necessary environment settings that haven't been defined in the Dockerfile should be [​passed explicitly](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables-e-env-env-file). The documentation describes several ways to do this, and choosing the most convenient and secure option is acceptable.

Running the application
=======================

Now it's time to run the application.

```bash
$ docker run ­--­name 'rails-­devise-­app' -­d -­p 3001:3000 ­--­link rails-­devise­-db:dbserver
rails-­devise
```

Parameters explained:

* `--­name 'rails­-devise-­app'` : a unique name for the application container. This is a convenient option for local experiments with a single manually managed container. Explicit container names are unnecessary in scalable, automated environments.
* ­`-d` ​: detach container process from the terminal
    * (alternative) ​`--it` : run the application process in the foreground so that logs are sent to STDOUT and the container can be stopped with Ctrl+C
* `-­p <docker server port>:<internal container port>` : map the internal container port to the predefined port on the Docker server. If this option is omitted, a random, non-privileged port will be assigned automatically.
* `--­­link rails­-devise-­db:dbserver` : establish a connection to the database container *rails-­devise-­db* ​and make it accessible from within the application container via the hostname *dbserver*. This is the default setup, defined in *database.yml*.
    * (not included in the example): defaults in ​*database.yml* can be overridden with parameters such as ​`-e DB_HOST=server.mycompany.com` or ​`-­e DB_PASSWORD=mypassword` and appropriate run-time environment variables. More information on run-time environment variables is available
[in the documentation​](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables-e-env-env-file).
* `rails-­devise` : name of the application image

The command above is similar to the command for database provisioning. However, there are a few important differences:

1. The container is launched in detached mode. Its standard output is collected by the Docker server, not sent to the console.
2. The container is explicitly given the unique name 'rails­-devise-­app' to simplify its manual management.
3. Because no other command was provided after the image name, Docker executes the default command defined by the "CMD" instruction in the Dockerfile. In our case, this command launches the Rails server inside the container.
4. Internal container port 3000 (the default Rails port) is mapped to the port 3001 of the Docker server. 3001 is an arbitrary choice that emphasizes the difference between server and container ports.

*Note: Explicit container naming and port mapping are primarily useful for local development and simple deployments. In a scalable cloud setup, many containers are launched automatically, so they cannot share the same name or bind to the same server port. Cloud platforms use higher-­level solutions to orchestrate containers and route traffic. These solutions will be considered in a separate article.*

If things are going as expected, the application is now up and running, exposed through the appropriate server port (for local setup, the URL will be either ​[http://localhost:3001​](http://localhost:3001/) or http://<docker-­machine ip>:3001).


Troubleshooting
================
Below is a list of the most common problems that prevent containers from being launched.


<table>
  <thead>
    <tr>
      <th>Error message</th>
      <th>Solution</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Unable to find image '...' locally <br> ...</td>
      <td>Ensure that:<ul><li>the image name is spelled correctly</li></ul></td>
    </tr>
    <tr>
      <td>docker: Error: image ... not found.</td>
      <td> * the image is available in the Docker server cache and listed in the output of `docker images`<br><br>If the image is missing from the cache, either <a href="../build-docker-image/">build it</a> or <a href="../manage-share-images/">pull it from the external registry</a>.</td>
    </tr>
    <tr>
      <td><code>docker: Error response from daemon:<br>Conflict. The name "/rails­-devise-­app" is<br>already in use by container<br>77429eb72d02970a7122cd4574a48b280b42
    62d7652f52590ef2f316c7f3a349. You have<br> to remove (or rename) that container to be able to reuse that name..</code></td>
      <td>A container with the same name has already been launched. It may have already stopped, but it is still associated with the name.<br><br>If you are sure that the previous container should be deleted, remove it with the following commands:<code>
    docker stop rails-­devise­-app && docker rm -­v rails­-devise-­app</code></td>
    </tr>
    <tr>
      <td>docker: Error response from daemon: failed<br>to create endpoint rails-­devise-­app on<br>network bridge: Bind for 0.0.0.0:3001 failed:<br>port is already allocated.</td>
      <td>Another process occupies the desired server port. Consider binding to another port.<br><br>If the port is handled by a previously launched container that is no longer needed, stop it with the command <code>docker stop &lt;container name or ID&gt;</code>.</td>
    </tr>
  </tbody>
</table>


If none of these errors have occurred, the container was probably launched successfully. However, there's still a chance that the application is not operating as expected. The Rails server inside of a container can crash immediately after starting (resulting in a stopped container). It can even continue to run while generating exceptions.

First, ensure that the container is running:

```bash
$ docker ps | grep rails-­devise-­app
77429eb72d02        rails­devise        "rails server ­b 0.0."   3 minutes ago      Up 3 minutes
0.0.0.0:3001­>3000/tcp    rails­-devise-­app
```

If rails­-devise-­app doesn't show up, it is probably in the list of stopped containers:


```bash
$ docker ps -­a | grep rails­-devise-­app
ea4def1549ed        rails­devise        "rails server ­b 0.0."   6 seconds ago       Exited (1) 3
seconds ago                             rails-­devise-­app
```

Regardless of whether the container has stopped, its standard and error outputs can be printed with the command `docker logs`:

```bash
$ docker logs rails­-devise-­app
<Rails server log with possible exception stacktraces >
```

*Be aware that the output of `docker logs` may not always be complete for the running container. This occurs because Ruby [buffers STDOUT by default](http://stackoverflow.com/a/9956069).* ​


Data can be copied from the filesystem of a running or stopped container with [`docker cp​`](https://docs.docker.com/engine/reference/commandline/cp/).

```bash
docker cp rails­-devise-­app:/usr/src/app/log .
```

Port mapping information and other metadata are available through the command [`docker inspect`​](https://docs.docker.com/engine/reference/commandline/inspect/) (check the documentation for the formatting options):

```bash
$ docker inspect rails­-devise-­app
... huge JSON content ...
```

If the Rails process crashes at start-up, manually troubleshooting with an interactive shell session may be better than using the Rails server.

```bash
$ ​docker run ­­--rm -it --­­link rails-devise-­db:dbserver rails-­devise /bin/bash
```

Parameters explained:

* `--­­rm` ​: automatically remove container after exit
* `-it` ​: attach interactive terminal
* `--­link rails-­devise­-db:dbserver` : see explanation in the previous section
* `rails-­devise` : name of the application image
* `/bin/bash` : command to be executed instead of `rails server`

Parameterization options
========================

Below is a short list of the most useful parameterization options available when the container starts.

<table>
  <thead>
    <tr>
      <th>Parameterization</th>
      <th>Command­ line parameter</th>
      <th>Remarks</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Mounting external directory or file into container</td>
      <td><code>­-v \&lt;local path&gt;:\&lt;container path&gt;</code></td>
      <td>Both paths should be absolute</td>
    </tr>
    <tr>
      <td>filesystem</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>Setting up run-time environment variables</td>
      <td>­<code>-e VARIABLE=value</code><br><em>or</em><br><code>-­e VARIABLE</code><br><em>or</em><br><code>--env-­file FILENAME</code></td>
      <td>Check ​<a href="https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables-e-env-env-file">the documentation</a></td>
    </tr>
    <tr>
      <td>Connect to another running container</td>
      <td><code>--­­link \&lt;container_name&gt;:\&lt;dns_alias&gt;</code></td>
      <td></td>
    </tr>
    <tr>
      <td>Connect to server network stack instead of Docker network bridge</td>
      <td><code>­­--net=host</code></td>
     <td>Use with caution; this option can be useful as a fast solution to some networking problems, but it can also cause confusion.<br><br><a href="https://docs.docker.com/engine/reference/run/#network-settings">Check the documentation</a> ​for more details.</td>
    </tr>
    <tr>
      <td>Automatically launch container when the server starts</td>
      <td>­­<code>--restart=always</code></td>
      <td><a href="https://docs.docker.com/engine/reference/run/#restart-policies-restart">Check the documentation</a>.</td>
    </tr>
  </tbody>
</table>

Access to running containers
===========================

The lack of default access to containers via SSH may be counter­intuitive to web developers. The reason for this access restriction is simple: common base images do not have an SSH server installed on them. This design decision is based on a common Docker philosophy: containers should do just one thing (in this case, running web applications).

Of course, installing and configuring an SSH server at the image build step is possible. Whether this is worthwhile depends on many factors that are specific to the situation.

Alternatively, the command [`docker exec`​](https://docs.docker.com/engine/reference/commandline/exec/) can be used to execute an arbitrary
command in the running container, including invocation of interactive shell sessions or the Rails console. The option `­-it` is required to attach an interactive pseudoterminal.

```bash
$ docker exec ­-it rails­-devise-­app /bin/bash
root@77429eb72d02:/usr/src/app#
```

```bash
$ docker exec -­it rails­-devise-­app rails c
Running via Spring preloader in process 61
Loading development environment (Rails 4.2.5)
irb(main):001:0>
```

*Note: Performing actions like rake tasks requires a connection to a running container. One-­off containers can be used instead (see the example in the "Database provisioning" section above).*

The command [`​docker kill​`](https://docs.docker.com/engine/reference/commandline/kill/) makes it possible to send signals to the application.

Cleanup
========

Stopping a container is easy with the command ​[`docker stop`​](https://docs.docker.com/engine/reference/commandline/stop/):

```bash
$ docker stop rails-­devise-­app
```

Stopped containers are not removed, and their logs, filesystem contents, and
metadata remain available.

The base image of a stopped container cannot be removed from the Docker server image cache, and new containers cannot be launched with the name of a stopped container.

Stopped containers can be re-started:

```bash
$ docker start rails-­devise-­app
```

To remove a stopped container, use ​the
command [`docker rm​`](https://docs.docker.com/engine/reference/commandline/rm/):

```bash
$ docker rm -­v rails-­devise-­app
```

Note that the `-v` option forces removal of the [​Docker volumes](https://docs.docker.com/engine/tutorials/dockervolumes/)​ associated with the container. Defaulting to this option prevents dangling volumes from increasing disk space usage.

Tips & tricks
=============

Instant code updates during development
---------------------------------------

During active development, it makes sense to mount the source code directory in the container at /usr/src/app. This allows all code changes that don't affect the configuration file or Gemfile to have an immediate effect without restarts or image rebuilds.

```bash
$​docker run --­­name 'rails-devise-­app' -­d -p 3001:3000 --­­link rails-­devise­-db:dbserver ​-v
$PWD:/usr/src/app​ rails-­devise
```

*If you use this approach, make sure that the Gemfile.lock in the working directory is similar to the one used in the image. Otherwise, Bundler will crash.*

After active development, code should be committed with version control, and the image used for staging and production should be rebuilt as usual.

Using Rails generators
----------------------

The setup described in the previous section is also useful for generating new code with Rails commands. If a daemonized container with a source code directory is mounted at /usr/src/app, a Rails generator can be launched inside of it with `docker exec`:

```bash
$ docker exec rails-­devise-­app ​rails generate scaffold Karma email:string score:integer
```

As a result, the appropriate files are created and updated in the source code directory. On Linux, new files are owned by root, so it makes sense to re­-own them:

```bash
$ sudo chown -­R $USER:$USER .
```

Launching a daemonized application container just to invoke a Rails generator is overkill. Running a one-­off container that is automatically removed when the work is done can create the same effect:

```bash
$ docker run --­­rm -­v $PWD:/usr/src/app rails-­devise rails generate scaffold Karma
email:string score:integer
```

Docker parameters explained:

* `--­­rm` : auto­-remove one-­off container on exit
* `-v $PWD:/usr/src/app` : mount current directory in the container (check for the Gemfile.lock issue mentioned in the previous section)
* `rails-­devise` : name of the application image
* `rails generate ...` : command to be launched

Simplified orchestration with Docker Compose
============================================

The previous examples were intended to familiarize you with all the essential aspects of container management and parameterization options.

The Docker CLI, however, is not very convenient for casual development because it requires many manual boilerplate actions and complex combinations of command­ line parameters.

[Docker Compose​](https://docs.docker.com/compose/) simplifies container management by describing the whole stack in a comprehensive, declarative format.

*docker­-compose.yml:*

```bash
app:
    image: rails-­devise
    # uncomment line below for automatic rebuild of the image
    # image: .
    ports:
        ­ "3001:3000"
    links:
        ­- dbserver
    environment:
        RAILS_ENV: development
    volumes:
        ­- .:/usr/src/app
dbserver:
    image: mysql
    environment:
        MYSQL_ALLOW_EMPTY_PASSWORD: "true"
```

Now both containers can be launched with `docker compose up`.

```bash
$ docker-­compose up
Creating railsdevise_dbserver_1
Creating railsdevise_app_1
Attaching to railsdevise_dbserver_1, railsdevise_app_1
[36mdbserver_1 | [0mInitializing database
...
```

The tool also supports executing arbitrary commands inside a running container for tasks such as database provisioning.

```bash
# app is the alias defined in docker-­compose.yml
$ docker­-compose run app rake db:create db:migrate db:seed
```

Used with other options (described in the ​
[Compose documentation](https://docs.docker.com/compose/))​, this high-­level tool can effectively manage the application lifecycle in development, staging, and CI environments.
