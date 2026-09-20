---
title: 'Docker'
date: 2024-07-03T09:50:09-04:00
draft: false
tags:
  - docker
categories: notes
keywords:
  - docker
---

- [Docker](#docker)
  - [Installation](#installation)
  - [Storage](#storage)
    - [Storage Drivers](#storage-drivers)
  - [Running Images](#running-images)
  - [Logging Driver](#logging-driver)
  - [Docker Swarm](#docker-swarm)
  - [Namespaces](#namespaces)
  - [Control Group](#control-group)
  - [Image](#image)
    - [Dockerfile](#dockerfile)
      - [Multi-Stage Build](#multi-stage-build)
    - [Managing Image](#managing-image)
    - [Flattening an Image](#flattening-an-image)

## Docker

### Installation

We have two versions of Docker.

- Docker Community Version (Docker CE)
- Docker Enterprise (Docker EE)

**Instalation Guide**

- [Ubuntu](https://docs.docker.com/desktop/install/ubuntu/)
- [Debian](https://docs.docker.com/desktop/install/debian/)

### Storage

By default all files created inside a container are stored on a writable container layer. This means that:

- The data doesn't persist when that container no longer exists, and it can be difficult to get the data out of the container if another process needs it.
- A container's writable layer is tightly coupled to the host machine where the container is running. You can't easily move the data somewhere else.
- Writing into a container's writable layer requires a storage driver to manage the filesystem. The storage driver provides a union filesystem, using the Linux kernel. This extra abstraction reduces performance as compared to using data volumes, which write directly to the host filesystem.

#### Storage Drivers

The storage driver controls how images and containers are stored and managed on your Docker host.

There are multiple storage drivers supported by Docker:

- `overlay2`
- `fuse-overlays`
- `btrfs` and `zfs`
- `vfs`

### Running Images

```bash
docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG..]
```

Some commonly-used options:

- `-d`: Run the container in detached mode.
- `--name`: To give a descriptive name.
- `--restart`: Specify when the container should restart.
  - `no` (Default): Never restart.
  - `on-failure`: Only if the container fails.
  - `always`: always restart the container.
  - `unless-stopped`: unless manually stopped.
- `-p <host port>:<container port>`: Expose port
- `--rm`: Automatically remove the container when it exits. Cannot be used with `--restart`
- `--memory`: Hard limit on memory usage.
- `--memory-reservation`: A soft limit on memory usage. It will activate when the host is running low on memory.

```bash
docker run hello-world
docker run nginx:1.15.11 # specifying the tag
docker run busybox echo hello world! # sending command to run on container
docker run -d nginx:1.15.11 # run the container in detached mode
docker run -d --name nginx --restart always nginx:1.15.11
docker run -d --name nginx --restart unless-stopped -p 8080:80 --memory 500M --memory-reservation 256M nginx:1.15.11
```

### Logging Driver

Docker includes multiple logging mechanisms to help you get information from running containers and services. These mechanisms are called logging drivers. Each Docker daemon has a default logging driver, which each container uses unless you configure it to use a different logging driver, or log driver for short.

**Overriding log driver for a Container**

```bash
docker run -rm --log-driver syslog nginx
docker run -rm --log-driver json-file --log-opt max-size=50m nginx
```

### Docker Swarm

Docker Swarm is a Docker-native clustering system that allows you to manage a group of machines as a single, virtual host. It enables you to run containers across multiple hosts and automates the deployment of containers across a cluster.

With Docker Swarm, you can easily scale your services, run the same container on multiple hosts, and manage the underlying infrastructure. It provides a simple command-line interface and a REST API for managing your services.

- To create swarm manager
  `docker swarm init`
- To join a swarm cluster
  `docker swarm join worker` # to get join command with token
- To get the node details
  `docker node ls`

### Namespaces

Docker uses namespaces to provide the isolated workspace called the container.
The Docker engine uses namespaces such as the following on Linux:

- The `pid` namespace: Process isolation
- The `net` namespace: Managing network interfaces
- The `ipc` namespace: Manage access to IPC resources
- The `mnt` namespace: Managing filesystem mounts
- The `uts` namespace: Isolating kernel and version identifiers.
- The `user` namespace: It allows a container process to run as root inside the container while mapping to a different unprivileged user on the host.

### Control Group

A `cgroup` limits an application to a specific set of resources. Control groups allow Docker Engine to share available hardware resources with containers and, optionally, limits and constraints.

### Image

An image is a read-only template with instructions for creating a Docker container. It contains the filesystem changes and configurations made when building a Docker image. An image typically contains the application and its dependencies. Images are created from Dockerfiles or can be pulled from a Docker registry. Once an image is created, it can be used to run multiple containers.

In Docker, an image is built using a Dockerfile, which is a text file that contains the instructions to build an image. The Dockerfile specifies the base image, installs dependencies, copies files, and sets environment variables.

To build an image, you can use the `docker build` command. Provide the path to the Dockerfile and, optionally, a tag for the image. The tag is used to identify the image later.

> Images are built in layers.

#### Dockerfile

It is used to create an image. A sample docker file example:

```dockerfile
# Simple nginx image
FROM ubuntu:bionic

ENV NGINX_VERSION 1.14.*

RUN apt-get update && apt-get install -y curl
RUN apt-get update && apt-get install -y nginx=$NGINX_VERSION

WORKDIR /var/www/html
# WORKDIR www # its a relative path since there is no /
ADD index.html ./
# COPY command can also be used, but add has some extra features

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

STOPSIGNAL SIGTERM
HEALTHCHECK CMD curl localhost:80
```

**Make the docker image faster**

- image should be ephemeral, easy to start, stop and restart
- Put things that are less likely to change on lower-level layers
- Don't create unnecessary layers
- Avoid including any unnecessary files, packages, etc. in the image.

##### Multi-Stage Build

Multi-stage builds have more than one `FROM` directive in the Dockerfile, with each `FROM` directive starting a new stage. Simple example:

```Dockerfile
# name the first compiler stage to be referenced in later layer
FROM golang:1.12.4 AS compiler
WORKDIR /helloworld
COPY helloworld.go.
RUN GOOS=linux go build -a -installsuffix cgo -o helloworld .

# start a new layer stage
FROM alpine:3.9.3
WORKDIR /root
COPY --from=compiler /helloworld/helloworld .
CMD L"
-/helloworld"]
```

#### Managing Image

```bash
# to pull the image
docker image pull <image name>
# to list all the images
docker image ls
# list all images
docker image ls -a
# inspect an image
docker image inspect nginx
docker image inspect nginx --format "{{.Architecture}} {{.Os}}"
# delete the image
docker image rm nginx
docker rmi nginx
# delete all dangling images
docker image prune
```

#### Flattening an Image

- Run a container from the image
- Export the container to an archive: `docker export`
- Import the archive as a new image using `docker import`

```bash
# run the image
docker run --name abc -d nginx
# export the container
docker export abc > abc.tar
# import the tar file
cat abc.tar | docker import - name:latest
```
