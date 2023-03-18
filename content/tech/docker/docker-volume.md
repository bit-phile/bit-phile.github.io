---
title: "Docker Volume"
date: 2023-03-12T10:38:56+05:30
draft: false
author: Nitin
tags: ["docker", "infra", "kubernetes", "docker volume", "volume", "storage", "bind mounts", "tmpfs"]
---

![Docker volume](/tech/docker/docker-volume.png "Docker Volume")

# Understanding Docker Volume Thoroughly

## Agenda

1. Abstract
2. Container layer and its data on docker host machine
3. What are the flaws of storing data in the container layer?
4. What is the solution?
5. Using docker volume
6. Conclusion

## Abstract

A Docker container is an independent process comprising the application and its dependencies. Container has its processes, file system, and networks independent of the host machine. 

Creating a resource in the running container stores that resource in the Read-Write layer or Container layer. This is a temporary storage mechanism. It is accessible until the container is running. Once the container is removed, all the data goes.

Docker provides a few mechanisms to tackle this problem and volume is one of them. Let's dig deeper into docker volumes.

> Note: All the examples are being done by using `minikube` for running docker daemon. Any mention of docker host machine refers to `minikube` node (You can go inside minikube node using, `minikube ssh`).

## Container Layer

I have explained about **container layer** in my previous post on Docker at [Docker Image Layers](/tech/docker/docker-image-layers). In this section, we would be seeing where actually the container layer is stored on the docker host machine. Excited? Seems like yes 😄. Let's start.

### Where does the container layer reside?

Container layer is a read-write layer stacked on top of read-only image layers. We can't modify read-only filesystem of read-only image layers. We can modify the contents in the file system of the container layer.

Let's spin up an `alpine` container with `sh` command.
```shell
docker run --rm -it alpine sh
```

Creating a file as
```
/ # echo "Docker is awesome - Nitin" > testimonials.txt
/ # ls
bin               home              mnt               root              srv               tmp
dev               lib               opt               run               sys               usr
etc               media             proc              sbin              testimonials.txt  var
```

Now, let's see where this data is stored on the docker host machine.

Inspecting the above container of ID `385f23abffa3` gives the following

```shell
$ docker inspect --format '{{ .GraphDriver.Data }}' 385f23abffa3
map[LowerDir:/var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032-init/diff:/var/lib/docker/overlay2/71129bf0c1fb93d386d1abd9390e68eb08b64e65cb31186a0e710931359adb72/diff MergedDir:/var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032/merged UpperDir:/var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032/diff WorkDir:/var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032/work]
$
```

Woo! It gives a map with a bunch of paths. Let me example you each of these paths. The following descriptions are based on `overlay2` storage driver.

1. `LowerDir` is the read-only file system for lower image layers. Any changes made to the file system are reflected in the new file system of the read-write container layer. `LowerDir` acts as a base file system.
2. `UpperDir` is the read-write file system for the container layer where we can create and delete files and directories. Container layer changes are stored in this location. 
3. `MergedDir` is the merge of `LowerDir` and `UpperDir`. It gives a unified view of the file system for the container.
4. `WorkDir` is something `overlay2` driver uses for its internal operations such as copy-on-write process.

As you can see `LowerDir` has two paths separated by `:`.

First one is the path of the lower image layer read-only file system.

```
/var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032-init/diff
```

Second one is the path of the lower image layer read-only file system after changes made by previous running containers.

```
/var/lib/docker/overlay2/71129bf0c1fb93d386d1abd9390e68eb08b64e65cb31186a0e710931359adb72/diff
```

These two directories serve as a base read-only file system for the container layer read-write file system.

As we create a new file `testimonials.txt` in the container, let's peek into `UpperDir` to see this new file. Remember, this is on the docker host machine.

```shell
$ sudo ls -la /var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032/diff
total 16
drwxr-xr-x 3 root root 4096 Mar 12 06:45 .
drwx--x--- 5 root root 4096 Mar 12 06:44 ..
drwx------ 2 root root 4096 Mar 12 06:44 root
-rw-r--r-- 1 root root   26 Mar 12 06:45 testimonials.txt
$
```

Yay! We have `testimonials.txt` on our docker host machine. If we `cat` we see,

```shell
$ pwd
/var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032/diff
$ cat testimonials.txt
Docker is awesome - Nitin
```

Great. If you have noticed when we do `ls` inside the container, it gives more directories like,
```shell
$:/ ls
bin               home              mnt               root              srv               tmp
dev               lib               opt               run               sys               usr
etc               media             proc              sbin              testimonials.txt  var
```
However, we didn't see these directories on `UpperDir`. Guess why?

The reason is, `UpperDir` only has the changes made to the container file system. This is where `MergedDir` comes into existence which provides a unified view of the file system. `MergedDir` is the merge of `LowerDir` and `UpperDir` to give a unified file system.

Let's see what is inside `MergedDir`.

```shell
$ sudo ls /var/lib/docker/overlay2/7cb62392ec2ace2c1ea07bbdee8c4976105f4f081531a29b7c5a0320aa5ae032/merged
bin  etc   lib    mnt  proc  run   srv  testimonials.txt  usr
dev  home  media  opt  root  sbin  sys  tmp               var
$
```

As we can see it is the same as the file system of the container. You can go ahead and try creating files in this location and see if it turns up in the container.

## The Flaws of storing data in the container layer

1. First thing first, storing data in the container layer is not permanent. Once you delete the container, all the data stored goes away.

2. `storage-driver` is getting used to dealing with the file system on the container. Using container file system for an IO-intensive application is always not a good idea. It causes performance issues and increases latency when accessing files stored on a container file system.

3. Container size gets increase when we start to store files on the container file system. I would give the example of a use case. Sometimes we get the need to create an image from the container itself. And if we have a huge amount of data inside the container, the resulting image would be big.

4. Sharing of data across containers is hard as it is highly coupled with the container itself. 

These are the problems we are considering here. There may be different other problems.

Then what is the solution, huh?

## Solution - Using Docker volume

Volume is not the only solution. It is one of the solutions docker provides. Some other solutions are `bind mound`, `tmpfs` etc. 

### What is Docker Volume?

This is the way of storing and managing persistent data outside of container file system. It is stored on a host file system managed by Docker. These volumes can be shared across different containers. 

Cool, Huh?

### Let's create a volume

Volumes are not tied to containers. So, we can create and manage volumes without even touching containers.

`docker volume` command is used to manage volumes.

```shell
$ docker volume

Usage:  docker volume COMMAND

Manage volumes

Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  prune       Remove all unused local volumes
  rm          Remove one or more volumes

Run 'docker volume COMMAND --help' for more information on a command.
$
```


Let's create a volume of the name `bitphile`,  😁.
```shell
$ docker volume create bitphile
bitphile
$ docker volume ls
DRIVER    VOLUME NAME
local     bitphile
$
```

Inspecting this volume gives us,
```shell
$ docker volume inspect bitphile
[
    {
        "CreatedAt": "2023-03-12T07:38:38Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/bitphile/_data",
        "Name": "bitphile",
        "Options": {},
        "Scope": "local"
    }
]
$
```

`Mountpoint` is where this volume is mounted. That is the location on the host machine where volume data will be stored.

Let's peek inside that location and see what's there.
```shell
$ sudo ls -la /var/lib/docker/volumes/bitphile/_data
total 8
drwxr-xr-x 2 root root 4096 Mar 12 07:38 .
drwx-----x 3 root root 4096 Mar 12 07:38 ..
$
```

NOTHING? Well, we just created it 😂!

### Mount Volume to container
There are two methods of mounting volume to the container. 
1. `--mount` option
2. `-v` option

`-v` option is like shorthand. `--mount` is verbose and explicit, which is why we going to use `--mount` throughout this post. 

Let's create a container and mount it to this volume we have just created.

```shell
$ docker run --rm -it --mount type=volume,source=bitphile,target=/bitphile-vol alpine sh
/ #
```

`--mount` option takes parameters are `key=value` pairs separated by `,`.
- `type` specifies the `volume` as type. `--mount` is being used by `bind mounts`, and `tmpfs`  also. By default, `type` is `volume`. But for the sake of being explicit, I have used it.
- `source` is the volume name.
- `target` is the path in the container file system that will be mounted with the volume. There is alias of this option as `dest`, `destination` etc.

Listing contents in the container file system `root` we see,
```
/ # ls
bin           home          opt           sbin          usr
bitphile-vol  lib           proc          srv           var
dev           media         root          sys
etc           mnt           run           tmp
/ #
```

We have `bitphile-vol` directory mounted to volume `bitphile`.

Let's create a file inside that directory.

```
/ # echo Good morning > bitphile-vol/greetings.txt
/ # cat bitphile-vol/greetings.txt
Good morning
/ #
```

Peeking on the volume mount location on the host machine, we see,

```shell
$ sudo ls -la /var/lib/docker/volumes/bitphile/_data
total 12
drwxr-xr-x 2 root root 4096 Mar 12 07:48 .
drwx-----x 3 root root 4096 Mar 12 07:38 ..
-rw-r--r-- 1 root root   13 Mar 12 07:48 greetings.txt
$
```

Cool! We have `greetings.txt`.

### Mount the same volume to another container

Let's spin up a new container with the same volume.

```shell
$ docker run --name bitphile-second -it --mount source=bitphile,target=/bit-vol alpine sh
/ $ ls
bin      etc      media    proc     sbin     tmp
bit-vol  home     mnt      root     srv      usr
dev      lib      opt      run      sys      var
/ $
```

Let's see what `bit-vol` has,

```
/ # ls bit-vol
greetings.txt
/ # cat bit-vol/greetings.txt
Good morning
/ #
```

Nice.

## Conclusion

Finally, we are at the end (the happiest moment when the zoom meeting ends 😂). 
So, docker volumes are used as one of the methods to store data permanently. There are some other methods of doing the same. We will have look at those sometime later. 

Until then,

![Bye by Mr Bean](/bye-mr-bean.gif "Bye bye, see you on next post")

## References
1. [Docker Volumes](https://docs.docker.com/storage/volumes)
