---
title: Advantages of Docker
category: php
permalink: /php/advantages-of-docker-with-php-apps
step: 1
tags:
- docker
- php
- advantages
excerpt: Docker for developing and deploying PHP applications.
description: How Docker helps speed up development and future proof the deployment process for your PHP applications.
---

The previous section gave a brief overview of Docker. This section will explain how useful Docker is for developing and deploying PHP applications.

Docker is a containerization technology that can add a great amount of value if your team has plans for the following:

1. Implement a ready-to-scale service-oriented architecture
2. Fully automate testing, delivery and deployment processes
3. Minimize differences in configuration across development, testing and production environments
4. Manage and share pre-built versions of your applications with your development team and stakeholders
5. Deliver applications for on-premise installation without granting access to private source code repositories
6. Deploy your application into production cloud environments with support for automatic scaling


# Dependency Isolation

The ability to isolate dependencies and potentially different stacks for each project is a requirement for both development and deployment of your applications, and Docker supports that out of the box. Applications are hosted in containers that can encapsulate not only tools like Composer or PHPUnit but also your web server and database.


```bash
$ docker build -t myapp-demo .
Step 1 : FROM php:5.5-apache
 ---> ff9a799b983b
Step 2 : COPY configs/php.ini /usr/local/etc/php/
 ---> b6ec6bd957bb
Removing intermediate container 808c5939f834
Step 3 : RUN docker-php-ext-install pdo_mysql
 ---> Running in 0a0446ebf521
+ cd /usr/src/php/ext/pdo_mysql
+ phpize

# <SNIPPED>

Step 4 : COPY src/ /var/www/html
 ---> 995f903a7d28
Step 5 : RUN a2enmod rewrite
 ---> Running in dbfd8487f713
Enabling module rewrite.
To activate the new configuration, you need to run:
  service apache2 restart
 ---> 654af5eb0175
Removing intermediate container dbfd8487f713
Successfully built 654af5eb0175
```

Running `docker build` creates an image which wraps up source code, configuration files and services required for executing our application. Running the image is just a matter of invoking `docker run`:

```bash
$ docker run --name myapp-web -p 80:80 myapp-demo
[Wed Jul 06 14:41:11.557739 2016] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.10 (Debian) PHP/5.5.37 configured -- resuming normal operations
[Wed Jul 06 14:41:11.561547 2016] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'
```

The Docker image actually encapsulates the entire software environment up to the operating system level (excluding kernel modifications, as briefly explained in the previous section). If you followed the installation steps from previous sections, your PHP application is now available at http://localhost. Otherwise, make sure to point you browser to the IP address of you Docker host machine (e.g. http://192.168.99.100).


# Running Other Services

Docker Hub enables one of the biggest conveniences with using Docker for development -- the ability to store and download pre-built images. In our example, the application has been wrapped in a container and the web server was launched successfully; however, we need a MySQL database for it to run properly. Normally, we would set up a local database on the development machine or configure a connection to an external database. With Docker, we just run a database container along with the application.

```bash
# Stop and remove existing container
$ docker kill myapp-web && docker rm -v myapp-web

# Launch a local MySQL service
$ docker run -d --name myapp-mysql -e MYSQL_ROOT_PASSWORD=12345 MYSQL_DATABASE=myapp-db mysql
67d2eef90f440a7327ba8114c9c75c08d059c50ae5ac954b0ae5a30d4041304b

# Re-launch application and connect to the MySQL service
$ docker run -d -p 80:80 --name myapp-web --link myapp-mysql:db myapp-demo
1e480c269b1062265bb3b85ef6cc6d33c00aba9e7c26fd1c69a69de71b154049
```

Now we can access the database container `myapp-db` from our PHP application container `myapp-web` using the `root` user with `12345` as password on the `db` host, which was set automatically by Docker when we linked both containers together.

The database container is a simple example of what can be launched. You could just as well launch a Redis cache or even another dependent microservice. Many popular open-source projects and services are available in a pre-built form on Docker Hub. By default, some don't require much more than a few command-line parameters to get running.

You can run as many instances of the same service without worrying about conflicts on a single host, and instances are easily purged once you're done, via the `docker rm` command.

# Reproducible Environments

There's a good chance most developers have uttered the phrase “..it worked on my machine!” at least once in their lives. When production errors occur, they should be easily reproducible; however, most machines are not configured exactly the same, and that can lead to issues that are unique across environments.

Composer can help defining your application's requirements, but we also need to mirror specific PHP or web server (Apache/nginx) configuration directives and modules that might need to be explicitly initialized. Installing and configuring these dependencies manually for each environment can become a problem when you need faster, error-proof scaling of your application or when a new developer arrives on your team.

With Docker, the whole setup process and requirements are written in a Dockerfile. The only requirement necessary to setup an environment is Docker.

```bash
FROM php:5.5-apache

COPY configs/apache2.conf /etc/apache2/apache2.conf
COPY configs/php.ini /usr/local/etc/php/
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

Besides the obvious advantage of reduced provisioning time, there’s another important benefit for applications that rely on sources from private code repositories. Any secrets that are needed to download private repository code can be kept isolated to the host that builds the image, and any environment that needs to run the application can just use the resulting Docker image.

# Flexible Tagging for Build Artifacts

Each image in a Docker repository can be assigned multiple tags, and be referred by them. If your application relies on a Continuous Integration build process, you can match tags with build identifiers so it's easy to deploy different releases of your app:

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

It’s possible to design and implement different types of application architectures with Docker. However, applying general dockerization practices help set up your application for fully automated testing, deployment and scaling in the cloud.

The most important approaches are the following:

- Separate configuration parameters (including secrets) from the application's code
- Isolate weakly coupled components into independent, stateless microservices
- Ensure your application and its dependencies run on a Docker-compatible environment
- Verify your application can be fully provisioned automatically
