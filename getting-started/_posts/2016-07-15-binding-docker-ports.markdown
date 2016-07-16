---
layout: page
title: Binding Ports
category: getting-started
step: 6
tags:
- getting-started
- ports
- networking
- docker
excerpt: Specify which IP address and ports to bind your containers to.
---

Docker containers can connect to the outside world without further configuration, but the outside world cannot connect to Docker containers by default.

## How this works

A bridge network is created (with the name `bridge`) when you install Docker. Every outgoing connection appears to originate from the host's IP space; Docker creates a custom `iptables` [masquerading rule](http://www.tldp.org/HOWTO/html_single/Masquerading-Simple-HOWTO/).

## Forward everything

If you append `-P` (or `--publish-all=true`) to `docker run`, Docker identifies every port the Dockerfile exposes (you can see which ones by looking at the `EXPOSE` lines). Docker also finds ports you expose with `--expose 8080` (assuming you want to expose port 8080). Docker maps all of these ports to a host port within a given `epehmeral port range`. You can find the configuration for these ports (usually 32768 to 61000) in `/proc/sys/net/ipv4/ip_local_port_range`.

### Where each ports go

Use the `docker port` command to inspect the mapping Docker creates.

## Forward selectively

You can also specify ports. When doing so, you don't need to use ports from the `ephemeral port range`. Suppose you want to expose the container's port 8080 (standard http port) on the host's port 80 (assuming that port is not in use). Append `-p 80:8080` (or `--publish=80:8080`) to your `docker run` command. For example:

```bash
docker run -p 80:8080 nginx

## OR ##

docker run --publish=80:8080 nginx
```

### Custom IP *and* port forwarding

By default, Docker exposes container ports to the IP address `0.0.0.0` (this matches any IP on the system). If you prefer, you can tell Docker *which* IP to bind on. To bind on IP address `10.0.0.3`, host port `80`, and container port `8080`:

```bash
docker run -p 10.0.0.3:80:8080 nginx
```
