---
title: Managing Secrets During Builds
category: rails
permalink: /rails/managing-secrets-during-docker-builds
step: 4
tags:
- docker
- rails
- ruby
- build
- secrets
- gems
excerpt: Fix problems that occur when sourcing private gems.
description: The methods available to secure the secrets (passwords, SSH keys) necessary when building a Docker image for a Ruby on Rails application that sources private gems. 
---


Problem overview
----------------

The `RUN bundle install` instruction smoothly executes a Docker build as long as gems are sourced from public HTTPS repositories. If the Gemfile lists the URLs of private repositories without their credentials, the Docker build fails.



Build-time authentication is a well-known problem with Docker:

* Installation is an isolated, non-interactive process that does not inherit environment state or authentication artifacts (SSH keys, passwords, tokens) from the Docker client that invoked the build.

* Any data or parameters in the Dockerfile and build context are persistently baked into the image.

* Forwarding ports and mounting external volumes are not possible at build time.



Many developers implement either insecure or inconsistent cut-offs, for example, embedding secrets into the image or installing gems at run time instead of build time.


These approaches can be sufficient for quickly getting  applications up and running, but they create a technical  debt by introducing architecture that isn't consistent with common security requirements and continuous delivery principles.


This article describes solutions that are focused on both security and consistency, from quick hacks to advanced strategies.


Sourcing private gems: available options
----------------------------------------

Storing credentials with source URLs is strongly discouraged, so it is not considered here. Bundler offers some alternatives:



* [Credentials in a configuration](http://bundler.io/man/gemfile.5.html#CREDENTIALS-credentials)

* Key-based SSH authentication

* Source paths pointing to the local file system (for Docker this means obtaining gems beforehand and adding them to the build context)

* Dedicated private gem server in the internal company network (no request for credentials)


Two recent options are technically trivial, but may not be universally acceptable. For instance, they are not very helpful for a developer who is trying to dockerize a Rails application for the first time.

Let's explore options for the Bundler configuration and SSH authentication.


Outline for fast solution
-------------------------

Both workarounds described below share the same conceptual approach:

* A custom script that performs necessary preparations before the “bundler install” command is executed in place of the `RUN bundle install` instruction

* Credentials (username and password, access token, or private SSH key) are provided to the script as a secret string

* Default implementation relies on a simple but insecure secret provisioning strategy for demonstration purposes

* Script allows switching to more advanced strategies with minimal modifications (uncommenting 1–2 lines of code) for production

* Script can be used for many projects without modification

* Secrets are never exposed to image-persistent filesystems

* Secrets are not explicitly included in Dockerfile instructions



Workaround #1: Bundler configuration
------------------------------------

The latest versions of Bundler can read gem source credentials from configuration parameters without putting them into Gemfile.lock. Docker configuration parameters can be defined as environment variables instead of being stored on the filesystem.


Gemfile records for private gem sources may look like the following:


```

gem 'private_gem_1’, git: 'https://bitbucket.org/username/private_gem_1‘'

source ‘https://gem.fury.io/username’ do
    gem ‘private_gem_2’
end

```

Let’s create a custom script that reads and processes credentials from the environment. It can be used across many projects, so we will put it into a docker/ subdirectory of the build context (which can eventually become a git submodule).


```

$ cat docker/bundle_installer.sh
#!/bin/bash

# CRED is the variable populated from build parameter
function get_credentials() {

    if [ n "$CRED" ]; then
        # for demonstration purposes (this is not 100% secure),
        # credentials are passed via build parameter as a plain text
        echo "$CRED"

        ## in a strategy based on secrets server, CRED is a temporary access token
        ## this expirable token is used to obtain actual credentials string via REST API
        # curl "https://my.secrets.server/bundler_credentials?token=$CRED"


        ## in a strategy based on FS squashing, CRED is a path to secrets file
        # cat “$CRED” && rm f “$CRED”
    fi;

}

# suggested format of credentials line:
# "user:password@bitbucket.org token@github.com token@gem.fury.io"
CREDENTIALS_STRING="$( get_credentials )"

# for each entry of the form “token@host”, add bundler configuration variable
for entry in $CREDENTIALS_STRING ; do

    # token is the part before '@'
    token=$( echo "$entry" | cut -f1 -d@ )

    # server is the part after '@'
    server=$(echo $entry | cut -f2 -d@ )

    # transform server name into a variable name: uppercase, underscores, prefix
    varname=$( echo "${server^^}" | sed -e 's/\./__/g' -r -e 's/^/BUNDLE_/g' )

    # configure bundler via environment variable
    export $varname="$token"
done

exec bundle install
```

In case you are not familiar with bash syntax, here’s the explanation:

1.	An adjustable get_credentials() function is defined. For demonstration purposes, it reads the credentials string directly from the environment.

2.	The credentials string is parsed according to custom rules and converted into a set of Bundler configuration variables.

3.	`bundle install` is executed.



To add the script to the build process, let’s modify the Dockerfile.

```

# place this instruction before any other COPY, as the script is not expected to change often
COPY docker/bundle_installer.sh /docker/bundle_installer.sh

…

# RUN bundle install # <-original instruction is replaced with lines below

# declare build parameter
ARG CREDENTIALS
# parameter is passed to the script as an environment variable.
RUN CRED="$CREDENTIALS" /docker/bundle_installer.sh

```

Now the build command requires an additional parameter, with the credentials serialized in a custom format:

```
$ docker built -t demo --build-arg CREDENTIALS=”user:password@bitbucket.org
token@gem.fury.io” .
```


**IMPORTANT NOTICE: Do not use the suggested script for production-like builds without the advanced strategies described below.**



Workaround #2: Key-based SSH authentication
-------------------------------------------

This variant is more universal because it is not Bundler-specific. The combination of SSH keys and an SSH agent is widely used to establish secure connections to protected resources.

In Gemfiles, SSH source URLs typically resemble the following:


```
gem 'private_gem', git: 'git@github.com:username/private_gem.git'
```


Before implementing the custom script, let's solve a minor problem that is specific to SSH connections.


Host key verification issues
----------------------------

With a Gemfile containing SSH-sourced gems, the `bundle install` step of the Docker build will fail immediately with the message *"Host key verification failed. fatal: Could not read from remote repository."*


This is a well-known problem with automated provisioning: by default, the SSH client refuses to establish a connection to untrusted hosts. A popular (but insecure) solution is to reconfigure SSH, disabling the StrictHostKeyChecking parameter.


Instead, let's compose a valid *known_hosts* file and deploy it in a Docker image. Creating the file, including checking fingerprints, takes just a few minutes and should be done only once.

```
$ ssh-keyscan -t rsa -H github.com > github.key
$ ssh-keyscan -t rsa -H bitbucket.org > bitbucket.key
$ ssh-keygen -l -f github.key
…
# ensure fingerprint matches the value listed on the page
# https://help.github.com/articles/what/are/github/s/ssh/key/fingerprints/
$  ssh-keyscan -l -f bitbucket.key
…
# ensure fingerprint matches the value listed on the page
# https://confluence.atlassian.com/bitbucket/use/the/ssh/protocol/with/bitbucket/cloud/221449711.html](https://confluence.atlassian.com/bitbucket/use-the-ssh-protocol-with-bitbucket-cloud-221449711.html)
$ mkdir -p docker/
$ cat github.key bitbucket.key > docker/known_hosts

```

Now, the *known_hosts* file can be committed to version control (preferably to a separate repository with few updaters) and used in all Docker-based projects.


Let’s add the following instruction to the Dockerfile, right after FROM:

```
COPY docker/known_hosts /root/.ssh/known_hosts
```

The “host key verification” problem is solved!


There may be warnings in the log about new records added to *known_hosts*. This is expected (the SSH client automatically creates copies of host keys for IPs that share a host name).


Support for key-based authentication
------------------------------------

Additional instructions in the Dockerfile are almost the same as those used for Bundler-configured credentials (the only difference is the deployment of the *known_hosts* file).

```
COPY docker/known_hosts /root/.ssh/known_hosts
COPY docker/bundle_installer.sh /docker/bundle_installer.sh

…

# RUN bundle install # <- original instruction is replaced with lines below


# declare build parameter
ARG CREDENTIALS
# parameter is passed to the script as an environment variable.
RUN CRED="$CREDENTIALS" /docker/bundle_installer.sh
```

The operation of `bundle_installer.sh` is conceptually similar to the previous solution, but there are some differences. The credentials string is supposed to contain the SSH private key, and, instead of a Bundler configuration, an SSH agent will be set up.

```
$ cat docker/bundle_installer.sh
#!/bin/bash


# CRED is the variable populated from build parameter
function get_credentials() {
   if [ -n "$CRED" ]; then
     # for demonstration purposes (this is not 100% secure),
     # credentials are passed via build parameter as a plain text
    echo "$CRED"

    ## in a strategy based on secrets server, CRED is a temporary access token
    ## this expirable token is used to obtain actual credentials string via REST API
    # curl "https://my.secrets.server/bundler_credentials?token=$CRED"

    ## in a strategy based on FS squashing, CRED is a path to secrets file
    # cat “$CRED” && rm -f “$CRED”
    fi;
}

CREDENTIALS_STRING="$( get_credentials )"

if [ -n "$CREDENTIALS_STRING" ]; then
    # /dev/shm is a temporary, memory-mapped FS
    TEMPFILE=/dev/shm/deployment.key
    echo "$CREDENTIALS_STRING" > $TEMPFILE
    chmod 0600 $TEMPFILE
    eval $(ssh-agent)
    ssh-add $TEMPFILE
    rm $TEMPFILE
fi

exec bundle install
```

The build is invoked with a CREDENTIALS parameter that contains the SSH private key.

```
docker build -t demo --build-arg CREDENTIALS="$( cat deployment.key )" .
```

**IMPORTANT NOTICE: Do not use the suggested script for production-like builds without the advanced strategies described below.**


Production security concerns
----------------------------

The solutions described allow images that depend on private gems to be built, but they are not exactly secure out of box. Although credentials are not stored on the image filesystem, the build parameters can still be determined from the image metadata with the command `docker inspect <imagename>`.

How does this relate to production-level security requirements?

**As of June 2016, [there’s no officially supported mechanism](https://github.com/docker/docker/issues/13490) to pass secrets to build steps without storing them permanently in the image.** A few advanced strategies are available, providing extra security at the cost of different overheads. They are overviewed in the next section.


The workarounds described above can be smoothly integrated with some of these techniques (namely, “FS squashing” and “Secrets server”), enhancing general secrets management with the following features:


* Lower-level logic of secrets processing (i.e., how secrets are read and used at each build step) is unified and can be shared across projects.

* Switching between strategies is supported out of box.

* Supported strategies include experimental mode with minimized security and overhead.


Advanced strategies
-------------------

The following strategies are widely used to compensate for the limited security of Docker build-time secrets management. All of them require actions beyond the standard build flow.



| Strategy | Details |
|:--------------|:-----------------------------------------|
| Private gems server | Gems are sourced from a private server inside a company network. Access restriction is based solely on network configuration, eliminating the need for credentials management.<br><br>**Advantages:** No changes in the standard build flow or source code are required (except to URLs in the Gemfile); security concerns are factored out of the Docker build.<br>**Disadvantages:** Server maintenance overhead; the need to manage private gems explicitly; Bundler-specific solution.|
| Build context pre-populated with gems | Dependencies (in binary or source form) are obtained from the authenticated environment (build server or workstation) and then added to the build context.<br><br>**Advantages:** Authentication is delegated to the build server or development environment.<br>**Disadvantages:** Complication of build flow; Bundler-specific solution.|
| Multi-phase build | There are two Dockerfiles, one each for the secret and public parts of the build scenario. Credentials are explicitly used to build a secret image; the secret image is used to launch a Docker container that generates a secrets-free build context for the public image. Only the public image is distributed.<br><br>**Advantages:** The credentials problem is solved with native Docker tools.<br>**Disadvantages:** Complication of build flow; a need to manage the secret and public images separately; non-zero risk of secret images being occasionally released.|
| FS squashing | Secret files are deployed into the image filesystem and deleted after<br>being used. To purge private data from intermediate layers, the final image is “squashed” with the [docker-squash](https://github.com/goldmann/docker-squash) tool, resulting in a new image with a compact history.<br><br>**Advantages:** Straight-forward solution to the credentials leak problem.<br>**Disadvantages:** Dependency on an external Docker-specific tool; extra step in the build flow.<br><br> **Integration with bundle_installer.sh:** The name of the secret file is passed as a build parameter; storing the file name in the image metadata is an acceptable trade-off as long as the file itself will not remain on the resulting filesystem.|
| Secrets server | Secrets server is a network service with an HTTPS-based API. It implements its own access management and can return secrets in exchange for an expirable bearer token. Implementations may vary from simple homemade solutions to enterprise-level platforms integrated with access management frameworks.<br><br>In this scenario, the build host or development workstation authenticates to the secrets server and obtains an expirable token with a short TTL. The token is used when the actual credentials are fetched via a network request through the secrets server API.<br><br>**Advantages:** Straight-forward solution to the credentials management problem; the server can be transparently upgraded to an enterprise-level platform; sensitive secrets never appear on the image FS or in the metadata.<br>**Disadvantages:** Server maintenance overhead; an extra step in the build flow (obtaining the  expirable token).<br><br>**Integration with bundle_installer.sh:** Expirable token is passed as a build parameter; storing the token in the image metadata is an acceptable trade-off as long as it has a short TTL.|



Summary
-------

Until the Docker platform provides a native API for secrets management, there are a number of alternatives, providing various combinations of complexity and security.


Regardless of the solution chosen, it always makes sense to follow best practices:


* Be aware of the risk of a private data leak posed by the Docker architecture

* Try to unify development and production builds techniques

* Use dedicated credentials (deployment keys, service accounts) instead of personal ones whenever possible

* Dedicated credentials should be restricted to the lowest possible level of permissions (read­only access granted only to repositories that are actually used)

* Regularly rotate credentials used for Docker builds
