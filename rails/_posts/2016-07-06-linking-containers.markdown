---
title: Docker Container Networking
category: rails
step: 8
tags:
- docker
- rails
- ruby
- linking
excerpt: Learn how to link containers together using Compose and other methods.
description: How to link your Ruby on Rails Docker containers together using Docker Compose and Swarm.
---

Dockerized applications typically consist of several components, each isolated in a separate type of container. This architecture introduces some practical questions:
 
- How do containers connect to each other, and how do they detect connection endpoints? 
- How is the application interface exposed? 
- What is the recommended method for SSL offloading? 
- Do clustering and load balancing require significant changes in application design? 
 
Answering such questions is relatively easy with a basic understanding of Docker networking. This article explains the key concepts and overviews the connectivity options that are available on various cloud platforms. 

## Completely Separated Services
 
A straight-forward solution to the inter-container connectivity problem is to deploy each component as an independent Docker container that is exposed to a predefined port on the Docker host with a resolvable address. 
 
With this solution, connection endpoints are definite and can be passed to appropriate containers as launch parameters. Of course, containers should be able to recognize these parameters (see an example of *database.yml* with overridable connection endpoints in [Running your Rails Application](../running-apps/)).
 
However, deploying and managing related components as a single stack is often more convenient. 

## Basic Setup with Container Linking

To demonstrate traditional connectivity techniques such as ports publishing and container linking, let’s begin with the simplest example.
 
We will use [*docker-compose*](https://docs.docker.com/compose/) to launch three containers: one each for a database (MySQL), a web application (a Rails­-devise image, as [described in an earlier article](../running-apps/)), and a reverse proxy (Nginx).
 
Let’s first create a minimal config file, which is necessary when Nginx is used as a reverse proxy. The config file has been simplified so that it only re-routes inbound traffic to the address “app:3000.”


#### nginx.conf
```yaml
server { 
  listen *:80; 
  location / { 
    proxy_pass http://app:3000; 
  } 
}
```

Here’s *docker-compose.yml*, which describes three linked containers.

#### docker-compose.yml
```yaml
database:
  image: mysql
  environment: 
    MYSQL_ALLOW_EMPTY_PASSWORD: "true" 
application: 
  image: rails­-devise  
  links:
    - database:dbserver # in database.yml, “dbserver” is default hostname 
proxy: 
  image: nginx 
  volumes: # mount custom config at runtime 
    - ./nginx.conf:/etc/nginx/conf.d/default.conf 
  links:
    - application:app # "app" is the hostname used in proxy_pass directive
  ports: 
    - 8080:80 
```    

All three containers can be launched with a single command: `docker-compose up ­-d`. Now the browser can access the web application that is exposed through Nginx on the Docker server port 8080.

Remarks:

* On Linux workstations, the appropriate URL will probably be [http://localhost:8080](http://localhost:8080/); it will be http://<docker­machine ip>:8080 for Docker Machine. 
* To see the web application interface instead of database configuration 
errors, initialize the database with the command `docker-compose run application rake db:create db:migrate db:seed`. 
 
This setup demonstrates several common patterns:

* **The external application endpoint is a port opened on a Docker server**. Port 80 of the proxy container is mapped to port 8080 of the Docker server with *ports publishing*.  
* **The reverse proxy is used to preprocess traffic**. In this demo, the proxy container simply forwards inbound traffic to the application container (which is referred to by the hostname “app”). In practice, a customized Nginx image could be used for tasks such as the following:
    * SSL offloading
    * Basic authentication
    * URL substitution
    * Conditionally routing requests to different versions (A/B testing)

* **Internal network isolation**. Communication between containers (proxy to application and application to database) happens inside the internal network. The proxy container is the only container that is externally exposed. 
* **Service discovery**. Containers do not rely on ephemeral internal IPs to establish connections to one another. Instead, they use symbolic names (“dbserver” and “app”), which are hardcoded in configuration files. In this demo, [container linking](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/) makes these aliases resolvable to appropriate containers. 
 
Port publishing and container linking are traditional Docker mechanisms for configuring basic connectivity. Appropriate options are available in not only [Docker Compose](https://docs.docker.com/compose/), but also in the [Docker CLI](https://docs.docker.com/compose/) and [API](https://docs.docker.com/compose/). 

## Links and Default Network Explained

When containers are launched, what happens under the hood?
 
All three containers were attached to the internal network and configured on the [bridge interface *docker0*](https://docs.docker.com/engine/userguide/networking/default_network/custom-docker0/).


#### $ docker inspect network bridge
```json 
...
"Containers": {  
  "66eb1525cf84b952455dee4070ef14089317fec9069b38391639e260d10e9533": {  
    "Name": "linking_database_1",  
    "EndpointID": "...",  
    "MacAddress": "02:42:ac:11:00:03",  
    "IPv4Address": "172.17.0.3/16",  
    "IPv6Address": ""  
  },  
  "95a53cd3a7417754a23ae8af31e9a41a03733e7b9e3c600de417ec8606e7d853": {  
    "Name": "linking_application_1",  
    "EndpointID": "...",  
    "MacAddress": "02:42:ac:11:00:04",  
    "IPv4Address": "172.17.0.4/16",  
    "IPv6Address": ""  
  },  
  "af1d1a083222cc042edd4b7423b8387ccfac15823e02df4f32f9c093c333b207": {
      "Name": "linking_proxy_1",  
      "EndpointID": "...",  
      "MacAddress": "02:42:ac:11:00:05",  
      "IPv4Address": "172.17.0.5/16",  
      "IPv6Address": ""  
    } 
}, 
```


Containers can connect to each other with internal IPs, but some form of service discovery is required to determine appropriate addresses. 
 
The link option enables the simplest form of service discovery: the file `/etc/hosts` (on the recipient container) is updated with records that point to the source container. 
 

```bash
$ docker­-compose exec proxy cat /etc/hosts

...
172.17.0.4  app 95a53cd3a741 linking_application_1 
...
```

Environment variables with link meta-information are injected into the recipient container's environment. Note that the linked container's environment variables are also injected. 

```bash
$ docker­-compose exec application env | grep ­-P ^DBSERVER_  
DBSERVER_PORT=tcp://172.17.0.3:3306 
DBSERVER_PORT_3306_TCP=tcp://172.17.0.3:3306 
DBSERVER_PORT_3306_TCP_ADDR=172.17.0.3 
DBSERVER_PORT_3306_TCP_PORT=3306 
DBSERVER_PORT_3306_TCP_PROTO=tcp 
DBSERVER_NAME=/linking_application_1/dbserver 
DBSERVER_ENV_MYSQL_ALLOW_EMPTY_PASSWORD=true 
DBSERVER_ENV_MYSQL_MAJOR=5.7 
DBSERVER_ENV_MYSQL_VERSION=5.7.11­1debian8 
```

For a long time, container linking was Docker's primary method of service discovery. However, it poses significant problems: 

1. The source container should be launched before the recipient. 
2. Links are unidirectional. It is impossible to simultaneously link container A to container B and container B to container A. 
3. If the source container is restarted or recreated (possibly with an IP change), the recipient container's environment variables are not updated. 
4. Variable injection may cause unintended configuration conflicts. For example, *database.yml* may use the variable `DB_NAME` to override default connection parameters, but `DB_NAME` would be set to the name of a linked container if the database container is linked under the alias “db”. 
5. All containers are connected to the same bridged network, which is neither segmentable nor scalable. 

Custom networks (introduced at the end of 2015) solve these problems.

## Custom Networking

[User-defined networks](https://docs.docker.com/engine/userguide/networking/dockernetworks/#user-defined-networks) are a relatively new feature, available since the release of Docker Engine 1.9:

1. Creating an arbitrary number of logical networks is possible; containers can be connected to and disconnected from the networks at any time.
2. Each container can be connected to an arbitrary number of logical networks. 
3. Only containers connected to the same logical network can communicate with each other. 
4. An embedded DNS server allows enhanced service discovery within a logical network. By default, containers are discovered by their names. A variety of aliasing options are available. The order in which containers are launched is not prescribed, two­-way service discovery is possible, and containers are replaced transparently. 
5. The [linking](https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks) option is still available as a legacy feature. In default bridge networks, the implementation is backwards-compatible. In user-defined networks, it is just an additional form of aliasing. 
 
User-defined networks support two alternative modes of operation: *bridge* mode for single Docker servers and *overlay* mode for clusters of Docker servers. In the latter case, different containers can run on different Docker hosts while connected to the same logical network. To containers, these modes are equivalent.
 
Below is an updated version of *docker-compose.yml* (from the previous example), with two custom networks. Note that the compose file format has been changed.

#### docker-compose.yml
```yaml
version: "2" # version matters, see https://docs.docker.com/compose/networking/
networks: 
  backend: 
  frontend: 
services: 
  database: 
    image: mysql 
    environment: 
      MYSQL_ALLOW_EMPTY_PASSWORD: "true" 
    networks: 
      backend:  
        aliases: 
        - dbserver  
  app: 
    image: rails­-devise  
    networks: 
      - backend 
      - frontend 
  proxy: 
    image: nginx 
    volumes: # mount custom config at runtime 
      - ./nginx.conf:/etc/nginx/conf.d/default.conf 
    networks: 
     - frontend 
    ports: 
      - 8080:80 
```

In the example above, the database container is connected to the "backend" network with the network-level alias “dbserver.” The application container is renamed "app," so that it can be discovered by the hostname "app" in any connected network.
 
## Container Replacement

If a container is stopped, its open connections are closed.
The connected container handles this situation, including possible IP address changes.

For example, Rails can handle changes in database IP transparently. On the other hand, Nginx correctly reconnects to the restarted application container (with the “proxy_pass” instruction) only if the IP address remains the same. Backend IP changes can be handled gracefully with alternative [Nginx configuration options](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#resolve), but they are more suitable for production. 

## Clustering and Cloud­-specific Orchestration

Production deployment on a cluster of Docker servers may invalidate assumptions derived from entry-level HOWTOs and experience with the single host. Moreover, networking and service discovery mechanisms may vary significantly with cloud platforms and the orchestration tools they support. 
 
This article provides an overview of some cloud connectivity options. This information can be taken into account to minimize differences between development, staging, and desired production environments. 

### Docker Swarm; platform-agnostic
[Swarm](https://docs.docker.com/swarm/) is a native Docker clustering technology that turns a pool of Docker hosts into a single logical Docker server. Such a cluster can be deployed on any available pool of hosts, from bare metal servers to cloud-hosted VMs. 
 
Docker Swarm fully supports Docker networking features and exposes a standard Docker API, allowing usage of Docker client tools, including Docker Compose.  
 
* By default, custom networks in a Swarm cluster are created in overlay mode. 
* Containers can discover one another with embedded DNS capabilities (described above). 
* However, the publishing container port only makes it externally accessible on the running container's node. It makes client configuration a bit tricky, and may require custom solutions for sophisticated DNS resolution. Supposedly, the problem will be solved with [swarm mode](https://docs.docker.com/engine/swarm/), which has been announced for Docker Engine v1.12. 
 
### Kubernetes; Google Container Engine 
[Kubernetes](http://kubernetes.io/) is one of the first widely used technologies for scaling and orchestrating Docker containers in the cloud. It is the primary tool for managing deployments in the [Google Container Engine](https://cloud.google.com/container-engine/). 
 
* One or more containers are included in a deployment unit called a [*Pod*](http://kubernetes.io/docs/user-guide/pods/). 
* Containers within a Pod share a network interface and should coordinate port 
usage. They can connect to one another's ports on “localhost” as if they were regular processes running inside a virtual machine. 
* An abstraction of a [*Service*](http://kubernetes.io/docs/user-guide/services/)  exposes Pods to one another and the outer world. A Service is associated with one or more Pods and has its own virtual IP. Requests to this virtual IP (as well as inbound connections to external ports associated with the Service) are transparently routed to the appropriate Pod. 
* The Google Container Engine supports additional load­balancing and traffic routing features with [ingresses](https://cloud.google.com/container-engine/docs/tutorials/http-balancer#step_3_create_an_ingress_object).
    
### Amazon Container Service; AWS
[Amazon ECS](https://aws.amazon.com/ecs/) provides a custom API and configuration language for Docker container management.
 
* Classic Docker links and port publishing can be described in container definitions. 
* Load balancing and external DNS resolution are supported by standard AWS 
components. 

## Summary  
 
1. Use ports publishing to expose frontend containers to inbound traffic through configurable ports on a Docker server. 
2. Consider using reverse proxy containers for traffic preprocessing tasks such as SSL offloading.
3. Container linking is the traditional mechanism for inter-container communication and internal service discovery, but it has many conceptual issues. Custom networking is a more consistent technology, and it also supports classic links for backwards­ compatibility. 
4. Relying on native Docker networking features is safe for deploying on a stand-alone Docker server.
5. The available connectivity options for clustered production deployments may vary with the target platform. 