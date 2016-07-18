---
layout: page
title: Securing your System
category: getting-started
step: 8
tags:
- getting-started
- secure
- docker
excerpt: Best practices for using Docker securely.
---

The primary concerns covered are the following:

1. What (and who) can you trust?
2. The Kernel, its namespaces, and control groups (cgroups)
3. The Docker daemon's attack surface
4. Loopholes in default and custom configurations
5. How do the kernel's "hardening" features interact with containers?

## Start with Trusted Images

When communicating across an untrusted medium (like the Internet), you need to ensure data and publisher integrity. Docker Engine pushes and pulls images to and from a public or private registry.

### Content Trust

**Content Trust** means you can *verify* the integrity and publisher for all information you receive across a channel. Unfortunately, Docker *does not* enable content trust by default. When running Docker `build`, `create`, `pull`, `push`, and/or `run`, add the flag `--disable-content-trust=false` to your command to enable Content Trust.

For example, use this line to run an NGINX container enabling Content Trust:

```bash
docker run nginx --disable-content-trust=false
```

Alternately you can enable Content Trust in a bash shell with the following command:

```bash
export DOCKER_CONTENT_TRUST=1
```

If you take this approach, use `--disable-content-trust` when you want to run a command and do not care whether the images is signed. *Note: If you enable Content Trust, you'll only see publisher-signed images.*

## The Kernel

Docker creates a set of namespaces and control groups when you start a container with `docker run`.

**Namespaces** isolate processes within a container; these processes cannot see (nor affect) processes running in another container or the host system. Namespace code, an effort to recreate the features of OpenVZ for mainstream access, runs on a large number of production systems and has since 2008 (with the original OpenVZ implementation releasing around 2005). Each container has its own network stack, preventing containers from accessing the sockets or interfaces of another container (containers may communicate via their network interfaces, similar to a hardware network interface). You must specify public ports or use links to allow IP traffic between containers (and you can further restrict the allowable traffic).

**Control Groups** implement resource accounting and limiting, providing metrics and ensuring each container gets its fair share of resources (like RAM, CPU, disk I/O, etc.). This infrastructure protects against a single container spiraling out of control and exhausting one or more resources, taking down the entire system. Control Groups really shine on multi-tenant platforms like public and private PaaS, guaranteeing a consistent uptime and performance, even in failing applications.

Docker supports user namespaces after 1.10 (i.e., in the most recent and last few versions), but Docker does *not* set these by default. User namespaces allow you to map a container's root user to a non-root user on the host. Start the daemon with `--userns-remap` to enable it. You can pass any of the following formats:

1. uid
2. uid:gid
3. username
4. username:groupname
5. default (automatically maps to `dockremap`; if this user and group do not exist, Docker creates them)

Default user management:

```bash
dockerd --userns-remap=default
```

## Attacking the Docker Daemon

First and foremost, **every time you run Docker, you *have access to `root` privileges.***

You should not allow users you do not know or trust to use your Docker containers. Docker allows you to share a directory between the host and container *without limiting the container's access rights*. If you start a container with `/` set as the `/host` directory, the *container has access to your entire filesystem*.

Docker's REST API endpoint changes in Docker 0.5.2, using a UNIX socket (instead of a TCP socket bound on 127.0.0.1). Prior to this, attackers could exploit cross-site request forgeries on Docker run without a VM (i.e., directly on the host machine). UNIX sockets *also* enable UNIX permission checks for limiting access.

Image loading via `docker load` or `docker pull` offers another attack vector. While the community focuses on improving `pull` security, you should avoid any source you do not trust.

## Configuration

In default configuration, Docker starts containers with restrictive access privileges.

Linux doesn't have a root/not-root dichotomy; Linux can grant processes binding below port 1024 to a "capability" (like `net_bind_service` for web servers). Web servers *do not* run as root (Apache *usually* runs as `apache`, and NGINX *usually* runs as `nginx` or `www-data`). Linux servers run multiple processes as root by default; things like SSH, cron, syslogd, network configuration, etc. Containers leverage this benefit and allow the host to manage these items: the host's SSH server manages the container's access, `cron` runs as a user process, and the host handles hardware and network management.

#### What this means

Containers do not need every root privilege; for instance, you can safely deny mount operations, access to raw sockets, and limit filesystem operations.

#### Capability Management

Docker allows you to add and remove capabilities. Much like securing a new server, best practices dictate removing *all* unnecessary capabilities, and keeping only the ones your processes explicitly require.

## Hardening the Kernel

### Run a Kernel with GRSEC and PAX

[Grsecurity](https://grsecurity.net/) secures the Linux kernel against mostly configure-less hardening options. With a history spanning 15 years and an active community, GRSEC adds safety checks at run-time *and* compile-time, improves role-based security (going above and beyond UNIX access control), further restricts `chroot`, improves Linux auditing, and much more. Its bundled component [PaX](https://pax.grsecurity.net/) protects against memory overwrites and arbitrary code execution in memory.

### Use Security Model Templates Where Available

RedHat ships with SELinux policies for Docker; use them. AppArmor has templates that work with Docker as well. If you have access to these tools, use them.

## Common Issues

### Content Trust

Publishers may digitally "sign" (or not sign) any image they release. If you enable Content Trust, you cannot pull, run, or build any unsigned images. Unless you have a specific need for a specific image, you should run Content Trust at all times (*especially* on production containers).

You **must** include the `latest` tag when pushing to a registry (failure to include this tag results in Docker skipping Content Trust).

### The Kernel

If you pass a value (other than default) to `--userns-remap`, ensure the value exists (as either UID, GID, username, or groupname); if the value does not exist, the daemon fails with an error message.

### The Docker Daemon

If you use Docker on a web server, *do not* allow remote login from your Docker user. *Sanitize your inputs.* Ensure an attacker cannot pass parameters to access Docker.

You can explicitly expose Docker's REST API. Do not do so unless you fully understand the implications and secure the API via trusted network, VPN, or `stunnel` and SSL certificates (do *not* use self-signed certificates for this purpose).

If you run Docker in production, reduce the attack surface area by running *only* Docker, admin tools, and monitoring tools on that server. Run *everything* else within Docker containers. Future versions of the Docker engine will likely run inside of containers.

### Remain Vigilant

- Maintain best security practices; understand and guard against the issues in the [OWASP Top 10](https://www.owasp.org/index.php/Category:OWASP_Top_Ten_Project) (you can sign up for a 14-day trial of the Acunetix Online Scan [here](http://www.acunetix.com/vulnerability-scanner/register-online-vulnerability-scanner/)).

- Do not leave sensitive data (keys, passwords, etc.) in your repositories or registries (public *and* private)

- If you have a solution you like more than GRSEC, SELinux, or AppArmor, use what you know. You'll benefit faster by hardening the system with a familiar technology than trying to learn a whole new technology from scratch.
