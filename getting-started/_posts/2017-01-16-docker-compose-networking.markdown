---
layout: page
title: Docker Compose Networking
category: getting-started
permalink: /docker-compose-networking
step: 12
tags:
- getting-started
- compose
- docker
- networking
excerpt: Configure networking between containers when using Docker Compose.
---

Docker Compose sets up a single network for your application(s) by default, adding each container for a service to the default network. Containers on a single network can reach and discover every other container on the network.

## Networking Basics
Running the command `docker network ls` will list out your current Docker networks; it should look similar to the following:

```bash
$ docker network ls
NETWORK ID          NAME                         DRIVER
17cc61328fef        bridge                       bridge
098520f7fce0        composedjango_default        bridge
1ce3c572afc6        composeflask_default         bridge
8fd07d456e6c        host                         host
3b578b919641        none                         null
```

You can alter the network name with the `-p` or `--project-name` flags or the `COMPOSE_PROJECT_NAME` environment variable. (In the event you need to run multiple projects on a single host, it's recommended to set project names via the flag.)

In our `compose_django` example, `web` can access the PostgreSQL database from `postgres://postgres:5432`. We can access `web` from the outside world via port 8000 on the Docker host (only because the `web` service explicitly maps port 8000.

## Updating Containers on the Network
You can change service configurations via the Docker Compose file. When you run `docker-compose up` to update the containers, Compose removes the old container and inserts a new one. The new container has a different IP address than the old one, but *they have the same name*. Containers with open connections to the old container close those connections, look up the new container by its name, and connect.

## Linking Containers
You may define additional aliases that services can use to reach one another. *Services on the same network can already reach one another.* In the example below, we allow `web` to reach `db` via one of two hostnames (`db` or `database`):

```yaml
version: '2'
services:
    web:
        build: . 
        links: 
            - "db:database"
    db:
        image: postgres
```

If you do not specify a second hostname (for example, `- db` instead of `- "db:database"`), Docker Compose uses the service name (`db`). Links express dependency like `depends_on` does, meaning links dictate the order of service startup.

## Networking with Multiple Hosts
You may use the `overlay` driver when deploying Docker Compose to a Swarm cluster. We'll cover more on Docker Swarm in a future article.

## Configuring the Default Network
If you desire, you can configure the default network instead of (or in addition to) customizing your own network. Simply define a `default` entry under `networks`:

```yaml
verision: '2'

services:
    web:
        build: . 
        ports:
            - "8000:8000"
    db:
        image: postgres

networks:
    default:
        driver: custom-driver-1
```

## Custom Networks
Specify your own networks with the top-level `networks` key, to allow creating more complex topologies and specify network drivers (and options). You can also use this configuration to connect services with external networks Docker Compose does not manage. Each service can specify which networks to connect to with its service-level `networks` key.

The following example defines two custom networks. Keep in mind, `proxy` *cannot* connect to `db`, as they do not share a network; however, `app` can connect to both. In the `front` network, we specify the IPv4 and IPv6 addresses to use (we have to configure an `ipam` block defining the `subnet` and `gateway` configurations). We could customize either network or neither one, but we do want to use separate drivers to separate the networks (review [Basic Networking with Docker](./basic-docker-networking) for a refresher):

```yaml
version: '2'

services:
    proxy:
        build: ./proxy
        networks: 
            - front
    app:
        build: ./app
        networks:
            # you may set custom IP addresses
            front:
                ipv4_address: 172.16.238.10 
                ipv6_address: "2001:3984:3989::10"
            - back
    db:
        image: postgres
        networks:
            - back

networks:
    front:
        # use the bridge driver, but enable IPv6
        driver: bridge
        driver_opts:
            com.docker.network.enable_ipv6: "true"
        ipam:
            driver: default
            config:
                - subnet: 172.16.238.0/24
                gateway: 172.16.238.1
                - subnet: "2001:3984:3989::/64"
                gateway: "2001:3984:3989::1"
    back:
        # use a custom driver, with no options
        driver: custom-driver-1
```

## Pre-Existing Networks
You can even use pre-existing networks with Docker Compose; just use the `external` option:

```yaml
version: '2'

networks:
    default:
        external:
            name: i-already-created-this
```

In this case, Docker Compose never creates the default network; instead connecting the app's containers to the `i-already-created-this` network.

## Common Issues
You'll need to use version 2 of the Compose file format. (If you follow along with these tutorials, you already do.) *Legacy (version 1) Compose files **do not** support networking.* You can determine the version from the `version:` line in the `docker-compose.yml` file.

### General YAML
Use quotes ("" or '') whenever you have a colon (:) in your configuration values, to avoid confusion with key-value pairs.

### Updating Containers
Container IP addresses change on update. Reference containers by name, not IP, whenever possible. Otherwise you'll need to update the IP address you use.

### Links
If you define both links and networks, linked services must share *at least one network* to communicate.
