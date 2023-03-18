---
title: "Getting Started With Docker"
date: 2023-02-26T11:53:43+05:30
lastmod: "2023-03-18"
tags: ["docker", "infra", "kubernetes", "container", "containerisation", "containerization"]
author: "Nitin"
draft: false
---

![Dockerfile-build-docker-image-docker-container](/tech/docker/getting-started-with-docker-banner.png "Dockerfile to docker image to docker container")

# Introduction

Developing software and applications is getting more complex day by day. This introduces the complexity of running and deploying the applications in different machines or environments. It also includes making sure it works everywhere no matter what the architecture of the host machine is.

Applications running in the same environment can conflict with other applications. This can cause malfunctions and make application development less effective. 

To solve this problem we need isolation and a virtual machine is one way to achieve this. We can have different virtual machines to run the applications that will be independent of other applications environments.

However, using a virtual machine has its own pros and cons. Some of these are the followings:

## Virtual Machine Pros

1. Stronger isolation and security as it provides hardware-level isolation which makes them more secure and isolated.

## Virtual Machine Cons

1. Requires more resources. Virtual machine requires guest system operation to run which consumes more resources.
2. Startup time is more in virtual machines as it has to boot the whole operating system.
3. Portability is less as virtual machines are tied to specific hardware and require additional configurations to run.

So, what alternative do we have to achieve this isolated environment? The answer is **containerization**.

# Containerization

Just to get started with this concept, I would like to give an example. We have seen containers that are used on ships to transport things. These containers have isolated environments and temperatures specific to the items they are containing. 

Similarly, containers can contain applications and supporting libraries required to run the application. This helps in running applications in any environment no matter what host system OS and architecture are.

# Docker

Docker is a containerization tool that is used to create and run containers. There are several other alternatives to docker like Podman, openVZ, etc.

## Architecture

Docker uses client-server architecture where the docker client talks with the docker daemon which can run on either a host machine, a virtual machine, or a machine on the cloud. Docker clients can connect to multiple daemons.

![https://docs.docker.com/engine/images/architecture.svg](/tech/docker/docker-architecture.svg "Docker architecture")
Source: https://docs.docker.com/engine/images/architecture.svg

### Docker client

Docker client is an interface that lets users create and run containers. Client talks with docker daemon using Docker API requests. It is not more than an application that can be web-based or cli that sends requests to the server and gets responses back. 

The server or daemon performs the requested job. 

### Docker daemon

A Docker daemon is the server that accepts requests from the docker client and performs the respective job. It can run on a host machine or virtual machine. Docker clients can connect to multiple docker daemons and change the daemon using docker client-provided commands.

For example,

```bash
docker context ls
```

It shows available contexts or daemons to which a client can connect to.

```
NAME                DESCRIPTION                               DOCKER ENDPOINT                                     KUBERNETES ENDPOINT   ORCHESTRATOR
default             Current DOCKER_HOST based configuration   tcp://localhost:2376                                                      swarm
desktop-linux                                                 unix:///Users/test/.docker/run/docker.sock                         
rancher-desktop *   Rancher Desktop moby context              unix:///Users/test/.rd/docker.sock
```

To change context, use

```bash
docker context use rancher-desktop
```

> Note: Rancher desktop is an alternative to Docker desktop you can use. Docker desktop requires a license. Rancher desktop creates a lightweight virtual machine on the host system which contains docker components. Docker daemon runs inside that virtual machine.
> 

### Docker Container and Image

**Container**

As I mentioned before, a container is a self-contained unit having all the necessary things required for the item container is containing. In this context, a container is a self-contained runnable unit that comprises the application and supporting libraries that are required to run that application.

**Image**

An image is a blueprint of a container that defines a container. An image contains all the instructions and resources required to create a container. 

If you have worked on OOPs, you must have heard about classes and objects. You can think of images as classes and objects as containers.

You can create your image by using other images as a base. I will show you shortly how to do it.

### Registry

This is the place where you can push your docker images. It's like GitHub where you push source code onto GitHub and here you would be pushing docker images. You can also pull docker images stored on the docker registry. 

Docker provides a docker hub register where you can get almost all docker images pushed by other people. Visit [hub.docker.com](http://hub.docker.com) for more details.

## Dockerfile

`Dockerfile` is used to build docker images. It contains all the instructions required to build an image. Following is an example of a `Dockerfile`. 
<!---TODO: mention syntax prefixes for Dockerfile-->

```Dockerfile
FROM fedora:latest
RUN dnf install figlet -y
ENTRYPOINT ["figlet"]
CMD ["bitPhile"]
```

### Explanation

1. `FROM` specifies to use `fedora:latest` as the base image. The base image is the parent image on which your image is based. A new image will be built on top of this base image.
2. `RUN` instruction is used to run the given command while building the image. 
3. `ENTRYPOINT` is the starting script that will run when the container is started.
4. `CMD` specifies the default command to run when the container is started.

> There is a difference between `ENTRYPOINT` and `CMD`. When `ENTRYPOINT` is provided, whatever argument `CMD` instruction has will be append as arguments to `ENTRYPOINT`. For instance, if we will see the above example, the final command will be `figlet hello world`.

### Building Docker Image

Save the above file and run the following docker command.

``` shell
docker build -t docker-fedora-test .
```

`build` is the subcommand that tells to build a docker image. `-t` is a tag where `docker-fedora-test` is passed as a tag name. `.` is the context.

> **What is context?** Context the part of the host file system that is required to build a docker image. For example, there may be some file or data which is required in image building. So, we pass those files using context. 
> 
> We can only pass one context.

A docker image is built through several layers of images. Each of the instructions in `Dockerfile` runs in a separate image layer. To learn more about docker image layers, refer to my blog [docker image layers](/tech/docker/docker-image-layers/).

### Running Docker Image

To run the image we've just built, run the following docker command.

``` shell
docker run docker-fedora-test
```

This gives output as,

``` shell
 _     _ _   ____  _     _ _      
| |__ (_) |_|  _ \| |__ (_) | ___ 
| '_ \| | __| |_) | '_ \| | |/ _ \
| |_) | | |_|  __/| | | | | |  __/
|_.__/|_|\__|_|   |_| |_|_|_|\___|                              
```

Awesome, right?

We can also pass arguments while we are running the container.

``` shell
docker run docker-fedora-test hello world
```

`hello world` gets append as an argument to `fedora` entrypoint command.

Folks, this is it for now. There are lots of things that come in docker and it is hard to put everything in one place both for you and me. So, see you in the next post on docker. Stay tuned. 

Please provide your feedback as it helps me improve the posts and get more information to you.

## References

1. https://docs.docker.com/engine/reference/builder/ for `Dockerfile`
2. https://docs.docker.com/get-started/overview/ for overview and architecture
