---
title: Dockerize your Django Application
category: python
step: 2
tags:
- docker
- python
- django
excerpt: Get started with dockerizing your Django application.
---

In this article, we'll cover how to dockerize your Django application.

Django is an open-source Python framework that is designed with speed, security, and scalability in mind. Django encourages clean design and rapid development.

## Django on Docker Hub

Search Docker Hub (in your console, GUI, or the website itself) for `django`. More detailed steps can be found in the article [Using Docker Hub](../../using-docker-hub)).

You'll find that there is an [official repository](https://hub.docker.com/_/django/) for Django. It's generally recommended to use the official repository when available.

Enter `docker run django` in your terminal. Docker will install Python 3 with Django 1.9+ unless you request a different tag. A full list of tags are available [here](https://hub.docker.com/r/library/django/tags/).

## Django Dockerfile

Writing your own Dockerfile is generally recommended, as it helps ensure you're familiar with what is included in your image and container. It also simplifies creating future containers when developing your app.

Using the existing Django image, your Dockerfile would look like the following:

```bash
FROM django

ADD . /my-django-app

WORKDIR /my-django-app

RUN pip install -r requirements.txt

CMD [ "python", "./manage.py runserver 0.0.0.0:8000" ]
```

If you want to create a *new* Django project and its scaffolding, use this Dockerfile instead:

```bash
FROM django

WORKDIR /usr/src/app

CMD [ "django-admin", "startproject hello_world_django" ]

CMD [ "python", "manage.py runserver 0.0.0.0:8000" ]
```

There are a couple of new commands in these Dockerfiles:

- `ADD` allows you to add local directories (local to the current folder) to your Docker container
- `WORKDIR` sets your working directory (so you do not have to run everything from the `/` or `/root` directories)

## Dockerizing without a Dockerfile

Instead of creating and building a Dockerfile, you can leverage the offical Django image that's stored on Docker Hub to run your app.

```bash
docker run --name hello-world-django -v "$PWD":/usr/src/app -w /usr/src/app -p 8000:8000 -d django bash -c "pip install -r requirements.txt && python manage.py runserver 0.0.0.0:8000"
```

The above command assumes the following:
- Your Django app is named `hello-world-django`
- Your app is stored in `/usr/src/app`

If you want to bootstrap a new application, use the following commmand (using the same directory and app name as above):

```bash
docker run -it --rm --user "$(id -u):$(id -g)" -v "$PWD":/usr/src/app -w /usr/src/app django django-admin.py startproject hello_world_django
```

- `requirements.txt` allows you to use the standard `pip` install support with requirements.txt. If you need components that do not come with the Django image, you need to use this option.

At the end, you use Python to run the development server via `manage.py` and bind that to the wildcard IP on port 8000. Since we already bound the container's port 8000 to the host's, we want to make sure the development server runs on port 8000 (Python's default).

## Further information

If you do not need the `requirements.txt` file, leave it out of your command (remove the line `RUN pip install -r requirements.txt` from your Dockerfile, or do not pass it to `bash` in the interactive CLI approach).

If you need to `pip install` packages or components not found in a package, you need to install those dependenices (using either `pip`, `requirements.txt`, or both). In the example above, we use `requirements.txt`, but we could equally pass a `pip install package_name` if we know what package(s) to look for.

While Django runs its development web server on port 8000 by default, its recommended to set port 8000 for the web server, in the event a default changes or the image you use utilizes a different default port.

**DO NOT** use Django's webserver in production. Use NGINX or Apache with the appropriate proxy (usually WCGI) running Python and Django.