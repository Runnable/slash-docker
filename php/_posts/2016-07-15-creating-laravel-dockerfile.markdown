---
title: Dockerize your Laravel Application
category: php
permalink: /php/dockerize-your-laravel-application
step: 2
tags:
- docker
- dockerfile
- laravel
- php
excerpt: Get started with dockerizing your Laravel application.
description: A guide to run your Laravel application in a Docker container with a sample Dockerfile and commands to build and run your Docker image.
---

In the previous articles we have defined what Docker is and how it can help to speed up both development and deployment of PHP applications. Now it's time to get into more specifics by dockerizing a Laravel application. You will learn how to create and build a custom Dockerfile while exploring Docker concepts and creating your very own setup.


Docker images
=============

An Image is an immutable and persistent file based on instructions (layers) that represent a given state of a virtual filesystem. Each one of these layers is an image itself representing a snapshot of a particular change (e.g. moving a file, installing a package) that can be used to build more complex images. Images can be built and then distributed by Docker Registries such as Docker Hub and/or executed as a container.


Docker containers
=================

A container wraps an application and all of its dependencies in an isolated environment, which runs on top of an Operational System. Everything you need to run on a given virtual or baremetal server can be installed in a container to give you the ability to run single instances of your application or services that it relies on, such as cache or database.


Container orchestration
=======================

Orchestration is the ability to manage and maintain different containers while easily provisioning resources for larger applications. There are a few specialized Orchestration services such as [Docker Swarm](https://docs.docker.com/swarm/) and [Kubernetes](http://kubernetes.io/).


Dockerfile
==========

A Dockerfile provides the instructions needed for Docker to build a given Image. Each line of a Dockerfile represents an image layer and can describe several actions, such as installing software (e.g. `RUN apt-get install nodejs` so you can build your front-end assets) or executing an application. It can be shared as source code in your application repository and built automatically by Continuous Integration/Continuous Delivery processes.


Prerequisites
=============

Take a moment to make sure you have everything in place for us to get started:

1. [Install Docker](../../getting-started/)
2. Source code of your Laravel application
3. Make sure your Docker machine is running and your environment is ready:

```bash
$ docker-machine start default
$ eval $(docker-machine env default)
```

For the following examples, we will use a default-structured Laravel application created with Composer. This demo app has a welcome page as its default route and another route to display a few items from a MySQL database.

```bash
$ cd laravel-app
$ ls
app		bootstrap	composer.lock	database	package.json	public		resources	storage		vendor
artisan		composer.json	config		gulpfile.js	phpunit.xml	readme.md	server.php	tests
```

Creating a Dockerfile
=====================

The first step to begin dockerizing an existing Laravel application is to put a Dockerfile on the base path of your source code repository. After that, we will define an official PHP Docker image with Apache support as the base image for our new Dockerfile. There are several releases with different PHP versions on [Docker Hub](https://hub.docker.com), make sure to check it out if you have to run specific versions. For this article, we will follow Laravel's requirements and use PHP 5.5.x image based on Apache, setup pdo_mysql and mod_rewrite while pointing our code base to the expected paths within our base image.

```bash
# Dockerfile
FROM php:5.5-apache

RUN docker-php-ext-install pdo_mysql
RUN a2enmod rewrite

ADD . /var/www
ADD ./public /var/www/html

```

These lines will set the foundation for our Laravel application image, and we are now ready to build it. We will name it `laravel-app`:

```bash
$ docker build -t laravel-app .
Sending build context to Docker daemon 199.3 MB
Step 1 : FROM php:5.5-apache
 ---> ff9a799b983b
Step 2 : RUN docker-php-ext-install pdo_mysql
 ---> Running in 887b0da6b9f0

# ...

 ---> c118d49af5a5
Removing intermediate container 887b0da6b9f0
Step 3 : RUN a2enmod rewrite
 ---> Running in 404d0edd6155

# ...

 ---> 3b310df07570
Removing intermediate container 404d0edd6155
Step 4 : ADD . /var/www
 ---> 32a347cdecff
Removing intermediate container 38d0903c1043
Step 5 : ADD ./public /var/www/html
 ---> 152beb432214
Removing intermediate container c714564b3dec
Successfully built 152beb432214
```

Notice that each instruction in the Dockerfile describes a separate step (layer) that modifies either the state of the image or its metadata by executing commands, copying files etc. To learn more about all the possible instructions, make sure to read the official [reference](https://docs.docker.com/engine/reference/builder/) documentation.

Now we can run a container from our newly built `laravel-app` image:

```bash
$ docker run -p 80:80 laravel-app
```

You’ve just dockerized this application! Pretty easy, right? You should be able to access it using your Docker Host IP address, e.g. http://192.168.99.100. If you are not sure what is the IP address of your Host (assuming you are using the `default` machine), simply type `docker-machine ip default`.

But there is more work to be done! The front page of our demo app works great, but what about our database content? Depending on how you have defined your `.env` file, you might need to tweak a few things. In our example, we used to run this app in a default LAMP stack, and everything we needed was being executed from a single machine. Since we are now running our web server from a Docker container, MySQL is no longer available from `localhost` -- at least not from the container perspective.

We should be able to address this pretty easily by overriding the `DB_HOST` content from `.env` by simply defining this environment variable during container execution. In this particular example, assume our database is available from `dbhost.com`.

```bash
$ docker run -p 80:80 -e DB_HOST=dbhost.com laravel-app
```

And that's it! You can re-define as many `.env` variables as you want during the launch of your container. If you need any special Apache or PHP configuration, you can add them to your image and be sure that whenever a new container is executed you will have the same environment. Assuming you have your config files in the `config/docker` directory, change your Dockerfile to:

```bash
FROM php:5.5-apache

RUN docker-php-ext-install pdo_mysql
RUN a2enmod rewrite

ADD . /var/www
ADD ./public /var/www/html

ADD config/docker/apache.conf /etc/apache2/httpd.conf
COPY config/docker/php.ini /usr/local/etc/php/

```

After that, build your image again:

```bash
$ docker build -t laravel-app .
Sending build context to Docker daemon   199 MB
Step 1 : FROM php:5.5-apache
 ---> ff9a799b983b
Step 2 : RUN docker-php-ext-install pdo_mysql
 ---> Using cache
 ---> c0f4fb60029a
Step 3 : RUN a2enmod rewrite
 ---> Using cache
 ---> f86bb4ac4781
Step 4 : ADD . /var/www
 ---> a0c6ca7a27d5
Removing intermediate container e4140c58178d
Step 5 : ADD ./public /var/www/html
 ---> 617306628c08
Removing intermediate container ee4f0ce87546
Step 6 : ADD config/docker/apache.conf /etc/apache2/httpd.conf
 ---> ff42b0665bf3
Removing intermediate container 21a029badade
Step 7 : COPY config/docker/php.ini /usr/local/etc/php/
 ---> e1075ec3576a
Removing intermediate container 66963e40a46e
Successfully built e1075ec3576a

```

Notice that Docker has a cache of some of the image layers that were created during the first time we built our Docker image, which makes the building faster this time. Now that we are all set, let's make sure we add our `Dockerfile` to our code repository, but first we should add a `.dockerignore` file to avoid propagating Git metadata into the build context of our image.

```bash
# .dockerignore
.git
```

```bash
$ git add .dockerignore Dockerfile
$ git commit -m ‘Dockerizing this great PHP app’
```

You are now ready to execute or distribute your application as a Docker Image without the need to configure your environment, which can save a lot of time if you are running multiple instances of your application. The Dockerfile used above builds on the official PHP image with Apache support that’s publically available from Docker Hub. It provides some very useful defaults, such as:

1. Standard, out-of-the-box configuration (easily overwritten as shown above).
2. Apache execution at container startup.
3. Convenient ways to add PHP and/or Apache modules through scripts

You can go further if you need additional tools, system-level packages or particular configuration files for your services. You can have an idea of what you can accomplish by looking at the Dockerfile of the [php:5.5-apache](https://github.com/docker-library/php/blob/master/5.5/apache/Dockerfile) image. You can also tie your image build process to a Continuous Delivery pipeline to automate its distribution or deployment.


Best practices while writing a Dockerfile
=========================================

There are a lot of things we can do within our Dockerfile and sometimes we end up sharing resources that we did not intend to or build an unnecessarily oversized image. Here are a few tips to make sure your Dockerfile is consistent and as simples as possible:

- Always use a `.dockerignore` file to limit the build context of your image.
- Keep your image size as little as possible by avoiding installing unnecessary packages, which will also potentially make your containers more secure due to fewer dependencies.
- Use trusted, official images from Docker Hub as base images.
- Make sure you are running a single process per container.
- Group commands into a single instruction (image layer) whenever possible.
- Place any instruction that is more likely to change at the bottom of your file for better usage of image layer caches.


Summary
=======

We have defined some key concepts of the Docker universe, such as Containers and Images and how powerful they are to help you distribute an application or execute several services, which can be orchestrated by great tools like Kubernetes or Docker Swarm.

We have also learned what is and built our own Dockerfile to address the execution needs of a Laravel application. We have also described how you can build your custom image and extend the configuration defaults provided by the official base images from [Docker Hub](https://hub.docker.com) while following best practices to make sure our Docker Image is easily distributed and deployed.
