---
title: Advantages of Docker
date: 2016-06-09 00:00:00 Z
categories:
- rails
tags:
- "-docker"
- "-rails"
step: 1
---

# Summary

The previous section gave a brief overview of Docker. This section will explain how useful Docker is for developing and deploying Rails applications.

Docker is a containerization technology that can add a great amount of value if your team has plans for the following:

1. Implement a ready-to-scale service-oriented architecture
2. Fully automate testing, delivery and deployment processes
3. Minimize differences in configuration across development, testing and production environments
4. Manage and share pre-built versions of your applications with your development team and stakeholders
5. Deliver applications for on-premise installation without granting access to private source code repositories
6. Deploy your application into production cloud environments with support for automatic scaling


# Dependency Isolation

Those familiar with bundler and gemsets will agree that the ability to isolate Ruby dependencies is a requirement for development and deployments. Docker supports dependency isolation out of the box. Applications are hosted in containers which encapsulate the Ruby binary, bundler and all required gems. There’s no need to use `RVM` or `rbenv` inside the container (although it’s possible if necessary).


```bash
$ docker build -t myapp-demo .

# <SNIPPED>

Step 7 : RUN apt-get update && apt-get install -y mysql-client postgresql-client sqlite3 imagemagick ghostscript --no-install-recommends && rm -rf /var/lib/apt/lists/*

# <SNIPPED>

Step 10 : RUN bundle install
---> Running in fb54a95359c2
Fetching gem metadata from https://rubygems.org/...........

# <SNIPPED>

Bundle complete! 29 Gemfile dependencies, 81 gems now installed.
Bundled gems are installed into /usr/local/bundle.

# <SNIPPED>

Removing intermediate container 70e704f07568
Successfully built 7cb6f9753914
```

Running `docker build` creates an image containing all dependencies obtained from aptitude and bundler. Running the image is just a matter of invoking `docker run`:

```bash
$ docker run --name myapp-web myapp-demo

=> Booting Puma
=> Rails 4.2.5 application starting in development on http://0.0.0.0:3000
=> Run `rails server -h` for more startup options
        => Ctrl-C to shutdown server
           Puma 2.16.0 starting...
           * Min threads: 0, max threads: 16
           * Environment: development
           * Listening on tcp://0.0.0.0:3000
```

The Docker image actually encapsulates the entire software environment up to the operating system level (excluding kernel modifications, as briefly explained in the previous section.)


# Running Other Services

Docker Hub enables one of the biggest conveniences with using Docker for development -- the ability to store and download pre-built images. In our example, the application has been built and launched successfully; however, it requires a MySQL database for it to run properly.

```bash
$ docker exec myapp-web rake db:create
#<Mysql2::Error: Unknown MySQL server host 'db' (25)>
...
```

Normally, we would set up a local database on the development machine or configure a connection to an external database. With Docker, we just run a database container along with the application.

```bash
# Stop and remove existing container
$ docker kill myapp-web && docker rm -v myapp-web 

# Launch a local MySQL service
$ docker run -d --name myapp-mysql -e MYSQL_ROOT_PASSWORD=12345 mysql
67d2eef90f440a7327ba8114c9c75c08d059c50ae5ac954b0ae5a30d4041304b

# Re-launch application and connect to the MySQL service
$ docker run -d -p 3000:3000 --link myapp-mysql:db -e MYSQL_PASSWORD=12345 --name myapp-web myapp-demo
1e480c269b1062265bb3b85ef6cc6d33c00aba9e7c26fd1c69a69de71b154049

# No more database-related errors! :)

# Set up the database
$ docker exec myapp-web rake db:create db:migrate

# Verify the application is up and running
$ curl http://localhost:3000
...
            <div id="header">
          		<h1>Welcome aboard</h1>
         		<h2>You&rsquo;re riding Ruby on Rails!</h2>
        	</div>
...
```

The database container is a simple example of what can be launched. You could just as well launch a Redis cache or even another dependent microservice. Many popular open-source projects and services are available in a pre-built form on Docker Hub. By default, some don't require much more than a few command-line parameters to get running.

You can run as many instances of the same service without worrying about conflicts on a single host, and instances are easily purged once you're done, via the `docker rm` command.

# Reproducible Builds

There's a good chance most developers have uttered the phrase “..it worked on my machine!” at least once in their lives. When build errors occur, they should be reproducible; however, most machines are not configured exactly the same, and that can lead to build issues that are unique across environments.

Gemfiles do help with defining your application's requirements, but it doesn't always provide the entire list. For example, the steps for installing most Rails apps provide little more than just a “bundle install” as the build step, which could output the following error on some machines if a necessary dependency isn't installed:

```bash
Gem::Ext::BuildError: ERROR: Failed to build gem native extension
```

Installing some dependency via ‘apt-get’ might help move past this error. But there might be more. If the application needs to download data to run properly, it may require wget or curl to be installed. This example isn't a big deal in itself, but you're always left wondering what else you may encounter:

- Custom (non-Ruby) code compilation errors
- Shell script errors
- Specific Ruby version requirements (common for some gems)
- Using RVM with the appropriate gemset names

With Docker, the whole build process and requirements are written in a Dockerfile. The only requirement necessary to run a build is Docker.

```bash
FROM ruby:2.2

# <SNIPPED>

RUN apt-get update && apt-get install -y \
            mysql-client postgresql-client \
            sqlite3 imagemagick ghostscript \
            --no-install-recommends \
            && rm -rf /var/lib/apt/lists/*

COPY ./myapp/Gemfile /usr/src/app/
COPY ./myapp/Gemfile.lock /usr/src/app/
RUN bundle install

COPY ./myapp /usr/src/app
```

Each line is a build step, and Docker caches steps that haven't changed. With a properly written Dockerfile, a change in the application code wouldn't require a re-installation of the entire software stack.

# Build Once, Run Everywhere

Building a Dockerfile locally results in a Docker image that's stored locally as well. This image can then be used to initialize and run application containers on the same host.

To share and run an image on another host, you'll need to push that image to a Docker Registry. A Docker Registry is a collection of images accompanied by an API that allows them to be pushed (uploaded) and pulled (downloaded) remotely. It can be hosted as a public service (like Docker Hub) or be hosted privately inside a company's network.

Docker images must be tagged before they can be pushed to a registry:

```bash
$ docker tag myapp-demo registry.mycompany.com:5000/myapp-demo

$ docker push registry.mycompany.com:5000/myapp-demo
```

After the image is pushed, anyone with appropriate access is able to pull it and run containers without re-running any build steps.

```bash
$ docker run registry.mycompany.com:5000/myapp-demo:latest
```

Besides the obvious advantage of reduced provisioning time, there’s another important benefit for applications that rely on gems sourced from private code repositories. Any secrets that are needed to download private repository code can be kept isolated to the host that generates the builds. Any environment that needs to run the application can just use the resulting Docker image.

Docker provides an effective way to share ready-to-run, pre-built applications across various runtime environments. The days of running a ‘bundle install’ and pulling down private code for production deployments are in the past.

# Flexible Tagging for Build Artifacts

Each image in a Docker repository can be assigned multiple tags, and be referred by them.

```bash
$ docker tag myapp-demo:latest myapp-demo:build_17
```

Aside from traditional versioning, tagging can be used to categorize a build in a workflow. For instance, image tags could reflect results of automated acceptance tests, manual approvals, stable/golden builds - anything that helps streamline your development workflow.

```bash
$ docker images myapp-demo
REPOSITORY          TAG                 IMAGE ID             CREATED            SIZE
myapp-demo        latest              776f972a88c1        14 minutes ago      947.9 MB
myapp-demo        build_17            2936042e126e        40 minutes ago      947.9 MB
myapp-demo        stable              2936042e126e        40 minutes ago      947.9 MB
myapp-demo        v0.1                2936042e126e        40 minutes ago      947.9 MB
```

Image tags make it easy to identify the right build, and are valuable in setting good practices around release and deployment processes. Although tagging source code branches is commonly performed for similar reasons, build artifacts are a more complete representation of what will end up running on production.

```bash
$ docker run myapp-demo:stable
```

# Unified Delivery Flow

Minimizing setup differences between testing and production deployments is extremely important to help reduce risks of bugs late in the development process. With Docker, images built by a continuous integration / continuous delivery (CI / CD) system from source code can also be used for all acceptance testing and deployment purposes. If your customers deploy your code in their own environments (on-premise), you could ship to them the same images that were generated during the development testing process.

# Ready-to-scale Architecture

It’s possible to design and implement different types of application architectures with Docker. However, applying  general dockerization practices help set up your application for fully automated testing, deployment and scaling in the cloud.

The most important approaches are the following:

- Separate configuration parameters (including secrets) from the application's code
- Isolate weakly coupled components into independent, stateless microservices
- Ensure your application and its dependencies run on a Docker-compatible environment
- Verify your application can be fully provisioned automatically