---
title: Manage and Share Images
step: 5
tags:
- docker
- rails
- ruby
- images
excerpt: Cleanup old images and distribute across hosts
---

Managing Images
===============

When a Docker image is built by the Docker server, it is assigned an alphanumeric ID which is derived from the contents of the image itself. *When a build occurs, you may see these IDs printed out after each step. They will correspond to intermediate layers, each also being an image.*

Naming and Tagging
------------------

It isn’t very convenient to refer to images by ID numbers. Docker helps by allowing images to be assigned a human-readable name and tag.

Within a Docker command and Dockerfile instructions, it’s possible to use either an explicit combination of a name and tag separated by a colon (e.g. "demo:1.0"), or a name only (e.g. "demo"). The latter format is an equivalent of "demo:latest", as "latest" is the default tag value.

In the command below, a request is made to the Docker server to build an image and assign the name "demo" to the result:

```bash
$ docker build -t demo
```

The resulting image and all its intermediate layers are placed in a server cache which also contains the base images to be used by the builds. The command `docker images` lists cache content (intermediate layers are hidden by default):

```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED                  SIZE
demo                latest              c6d910fb2588        Less than a second ago   940.6 MB
rails               onbuild             f29b421f27a2        2 weeks ago              782.9 MB
ruby                2.3                 2d43e11a3406        2 weeks ago              729.1 MB
```

Note that the tag "latest" is assigned to the "demo" image automatically. This is because the `-t` parameter in the build command did not contain an explicit tag.

It is possible to assign many tags to the same image:

```bash
$ docker tag demo:latest demo:1.0
$ docker images
REPOSITORY          TAG                 IMAGE ID           CREATED                  SIZE
demo                1.0                 c6d910fb2588       About a minute ago       940.6 MB
demo                latest              c6d910fb2588       About a minute ago       940.6 MB
...
```

The parameters of a `docker tag` command include both names and tags.  Does this mean it is possible to assign a new name? It does indeed:

```bash
$ docker tag demo:latest newname:latest
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
demo                1.0                 c6d910fb2588       2 minutes ago       940.6 MB
demo                latest              c6d910fb2588       2 minutes ago       940.6 MB
newname             latest              c6d910fb2588       2 minutes ago       940.6 MB
```

Tags could also be explicitly specified as command-line parameters at build-time:

```bash
$ echo '#fake update to invalidate cache' >> Rakefile
$ docker build -t demo:1.1 .
...
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED                  SIZE
demo                1.1                 f8127c45bb54        About a minute ago       940.6 MB
demo                1.0                 c6d910fb2588        5 minutes ago            940.6 MB
demo                latest              c6d910fb2588        5 minutes ago            940.6 MB
...
```

Note that in the example above, the tag "latest" still points to the older version of the image. This happened because we provided "1.1" as an explicit tag value for the build command and there was no substitution of default values. **Using "latest" as the default tag value is a strong convention used widely in the Docker ecosystem. However, it should be understood that Docker implements no special logic to guarantee that the tag "latest" actually points to the most recent version of the image.**

Moving Tags
-----------

Tags can be easily moved around; either explicitly by pointing an existing tag to another target with the `docker tag` command, or at build-time.

Let’s rebuild the image and assign the already existing tags "1.1" and "latest" to the result:

```bash
$ echo '#fake update to invalidate cache' >> Rakefile
$ docker build -t demo:1.1 -t demo:latest .
...
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
demo                1.1                 c32705f7fde6        About a minute ago   940.6 MB
demo                latest              c32705f7fde6        About a minute ago   940.6 MB
<none>              <none>              f8127c45bb54        2 minutes ago        940.6 MB
demo                1.0                 c6d910fb2588        7 minutes ago        940.6 MB
...
```

Note that the previous version of "demo:1.1"` is present in the list, but it no longer has a name or tag associated with it. These images are called 'dangling' and can be referenced to by an ID in commands such as `docker run`, `docker tag`, or `docker rmi`. Dangling images should be wiped periodically:

```bash
$ docker images -qf "dangling=true" | xargs docker rmi
```

Deleting Images and Tags
------------------------

The `docker rmi` command removes images and tags specified as command-line arguments.

If you try to remove an image by ID - except dangling images - there is a high likelihood an error similar to the one below will be invoked:

```bash
$ docker rmi c32705f7fde6
Error response from daemon: conflict: unable to delete c32705f7fde6 (must be forced) - image is referenced in one or more repositories
```

This means you will first have to remove the tags pointing to the image. It is also possible to force an image deletion with `-f` parameter (related tags will be purged as well).

Removing images by deleting tags is often more convenient. If there are multiple tags pointing to the target image, only the subject tag is removed. When the last tag pointing to the image is deleted, the image itself and its unreferenced intermediate layers are purged automatically:

```bash
# demo:1.1 and demo:latest are pointing to the same target
$ docker rmi demo:1.1
Untagged: demo:1.1
$ docker rmi demo:latest
Untagged: demo:latest
Deleted: sha256:c32705f7fde6db9567de9b6d4be3361c854dfd69f99dabbbf0d3aa5de039efd8
Deleted: sha256:c1564197ee0cf3767bca30eddeb88a8aeb8c870e7c8371989e492c645decbf63
```

Sometimes the deletion of images may fail because containers on the Docker server are using them; an error will be invoked:

```bash
Error response from daemon: conflict: unable to delete 9d430935a8da (must be forced) - image is being used by stopped container d3ec51ef93d5
```

In such situations, [remove problematic containers](https://docs.docker.com/engine/reference/commandline/rm) and repeat the image delete process again.

Tags and Metadata
-----------------

Tagging is a great way to organize and refer to images by different parameters: build number, release number, or release alias (e.g. "stable" or “testing”). It is acceptable to create several tag taxonomies when appropriate.

That being said, tags should not be used to associate **any** form of metadata with the image. Tags should be thought of as pointers with obvious implications:

1. A particular tag can only point to one image at a time.

2. A tag could be re-pointed to a different image ID at any time.

3. An unneeded image will not be purged until all tags pointing to it are deleted -- or forced removed with `docker rmi -f <IMAGE ID>`.

It’s easy to see how tag overuse may lead to bloating of an image cache. It makes sense to assign tags which will be explicitly used for referencing of images. For any secondary metadata, consider using [Docker labels](https://docs.docker.com/engine/userguide/labels-custom-metadata/).

Sharing Images
==============

The Image is the core shareable asset in the Docker world. For image distribution, the Docker ecosystem relies on the concept of a **registry** -- a web service providing the ability to push and pull images which will be referenced by a name and tag combination.

Docker terminology describing image distribution has historically been somewhat confusing. For those interested in the details, there’s a [great blog post](http://blog.thoward37.me/articles/where-are-docker-images-stored/) clearly explaining this.

For a quick start, it is enough to memorize the essentials:

1. Images can be pushed and pulled to/from a **registry** (or many registries).

2. Images with the same name but different tags constitute a **repository** in the registry.

3. [Docker Hub](http://hub.docker.com) is the default registry which supports hosting of both public and private repositories.

4. It’s possible to set up and use many registries; a company can set up its private registry on its own premises.

5. Cloud platforms supporting Docker typically have their own registries as a part of their deployment infrastructure.

6. A Docker registry itself does not enforce access control. Registries that are exposed to the world are typically accompanied by a separate service which is responsible for access and the authentication of clients to specific repositories.

Let’s start with the creation of a private repository on Docker Hub and use it to share the "demo" image across a development team.

Private Repository on Docker Hub
--------------------------------

With the free plan, Docker Hub provides a single private repository per account. It wouldn’t be enough for complex architectures, but it is more than enough for getting started.

1. Sign up at [https://hub.docker.com/](https://hub.docker.com/)

2. At the dashboard page, click "Create repository" and fill in the details. (For compatibility with the examples below, use the “demo” naming convention.)

3. **Make sure to set visibility to "Private."**

4. Optional: Ask other members of your team to sign up and add them as collaborators to the repository.

5. Navigate to: Web UI flow: "Dashboard" -> Repository “Details” -> ”Collaborators”

6. Once in the environment used for images management, navigate to [authenticate Docker CLI](https://docs.docker.com/engine/reference/commandline/login/) to Docker Hub:

```bash
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: ...
Password: ...
Email: ...
Login Succeeded
```

Pushing Images
--------------

Now we are ready to push the demo image from a build server to the remote registry. Before doing this, the image has to be assigned a new name associated with the Docker Hub repository just created.

Docker CLI figures out the target registry by analyzing the image name. In the case of Docker Hub’s repository, it should have the following format:

```bash
dockerhub_username/repository_name[:tag]
```

Here, we’ll set the names appropriately:

```bash
$ docker tag demo username/demo
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
demo                1.1                 d3e250a3007f        About an hour ago   940.6 MB
demo                latest              d3e250a3007f        About an hour ago   940.6 MB
username/demo       latest              d3e250a3007f        About an hour ago   940.6 MB
```

In this case, we used the same name `demo` for the local image and Docker Hub repository, but it is not mandatory. As no tags were explicitly provided, Docker CLI assumed the default tag "latest" for both old and new names.

Now the image is ready to be pushed:

```bash
$ docker push username/demo
The push refers to a repository [docker.io/username/demo]
69e08a9db1bd: Pushed
0bb30b0ee2c4: Pushed
...
latest: digest: sha256:... size: 5320
```

The first push may take some time, but when there are changes to the image, only the updated layers are pushed and the whole process becomes significantly faster.

A few things to keep in mind:

1. Docker CLI does not recognize that the original image 'demo' is supposed to end up in the remote registry. After rebuilds, all images should be tagged with an appropriate prefix before being pushed.

2. A push does not happen automatically on rebuilds; `docker push` should always be executed explicitly.

3. If the **push** command argument has no tag component (e.g. `docker push username/demo`), all images and tags associated with the name "username/demo" will be pushed.

4. If the **push** command argument specifies a particular tag (e.g. `docker push username/demo:1.0`), only the specified image and tag will be pushed.

5. Removing an image from the remote repository is not trivial.

Pulling Images
--------------

You can how transfer your pushed image to another host that’s running a Docker server by logging in to Docker and running a container from the shared image "*username*/demo":

```bash
$ docker login
...
$ docker run -it username/demo
Unable to find image 'username/demo:latest' locally
latest: Pulling from username/demo
51f5c6a04d83: Pull complete 
...
Digest: sha256:...
Status: Downloaded newer image for username/demo:latest
=> Booting WEBrick
=> Rails 4.2.6 application starting in development on http://0.0.0.0:3000
...
```

As the Docker server did not find an image named "*username*/demo:latest" locally, the image was downloaded from the appropriate repository hosted on Docker Hub. This same action would occur by executing `docker build` using Dockerfile with the instruction “FROM *username*/demo”.

Note that during subsequent executions of `docker run` or `docker build` commands, the image will be found in the local cache and by default **will not be automatically pulled again** from Docker Hub **even if it is updated** in the remote repository.

To explicitly force a download of the newest image, use command `docker pull username/demo` for the "latest" tag or `docker pull username/demo:tag` if you need a specific version.

Private Registry on Premises
----------------------------

If you need to keep your source code and images behind the firewall, you can run your own private registry fairly easily. There’s an [official guide](https://docs.docker.com/registry/deploying/) describing the default setup.

Similar to Docker Hub, images could be associated with a private registry by assigning appropriate names to them. Instead of a Docker Hub-specific username, the prefix should include the host and port of private registry. The desired image naming format is:

```bash
host:port/image_name[:tag]
```

The commands `docker push` and `docker pull` work as described above. By default, `docker login` is not required unless you implement additional authentication; in such cases, you can run `docker login <registry URL>`.

A few things to keep in mind:

1. By default, a private registry does not enforce permissions control. Thus, it should be deployed inside the company’s trusted private network.

2. For everyday use, it is recommended to utilize an external data storage method by configuring an alternative [storage backend driver](https://docs.docker.com/registry/storage-drivers/).

3. A fully-secured setup is possible but requires some extra effort to set up authentication and SSL. The Official Docker Guide contains [basic recipes](https://docs.docker.com/registry/deploying/#restricting-access). For advanced solutions, the very detailed [DigitalOcean guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04) is worth a read.

Working with Multiple Registries
--------------------------------

Many Cloud providers support their own registries as a part of their deployment workflow. For example, they require an image to be pushed into their registries as a basis for running certain containers in the Cloud. It is also possible that your Continuous Integration platform includes a registry which is the primary source of automatically built images.

Using these registries is straightforward and typically just requires two things:

1. Authentication with `docker login <registry URL>`. (Docker CLI provides authentication to multiple services).

2. Naming images in a prescribed format that often includes both hostname and username (or project ID) as prefixes. For example, when pushing to Google Cloud, an image named "demo" should be tagged as “gcr.io/project-id/demo:latest”.

Copying images across different registries is simple:

```bash
$ docker pull private_ci_registry:5000/demo:1.0
...
$ docker tag private_ci_registry:5000/demo:1.0  gcr.io/myproject/demo:1.0
$ docker push gcr.io/myproject/demo:1.0
...
```

Best Practices
--------------

There are two important principles for quality Docker-based software delivery:

* The same image should be used throughout development, acceptance testing and for deploying to production.

* Manually built images typically shouldn’t be used in acceptance testing or production, as their consistency cannot be guaranteed.

No matter how trusted and reliable you may be, your dev machine(s) are primarily used to develop and troubleshoot unstable or error-prone code. Thus, it’s discouraged to manually push images into production pipelines using dev machines in order to prevent an accidental deployment.

The following practices should be helpful to establish separation of duties by isolating development and production pipelines from each other:

1. Set up a private "Sandbox" registry which developers use to share their images for troubleshooting and reviewing. Images from this registry should never be pushed to production or delivered to customers.

2. For continuous integration and continuous delivery pipelines, use a separate "Integration" registry, which **only build servers are allowed to write (push)**. Images in this registry are built using only the source code and Dockerfiles checked out of version control.

3. Images from the "Integration" registry could be used for staging and debugging purposes.

4. If you are sharing images to customers, use a separate "Release" registry for this purpose. This way, your internal tags and unstable images will not be exposed.

5. Only approved stable images from the "Integration" registry should be re-pushed to the “Release” registry and on to Production registries used by Cloud platforms. **Only scripts or persons performing production releases should be authorized to write to production registries.**