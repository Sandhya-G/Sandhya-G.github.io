---
layout: post
title: "Breaking out of Docker"
subtitle: "Docker Escapes"
date: 2021-06-06 20:15:13 -0400
---

 Breaking out of Docker





https://syzkaller.appsspot.com/upstream

How to know shell you have access to is running in a docker container `/proc/1/cgroup``

capsh —print 



Linux Security  Modules (AppArmor(strongest defense for docker  ) and SELinux)

LSM prevents mounting a writable /proc or /sys entries and restricts access to various system calls and sensitive files. 

Docker has default seccomp policy blocks syscalls in container also blocks unshare which requires caps as ADMIN this one limits loading of kerner modules. we need unshare for many kernel exploits.


## Container Engine Vulnerabilities

weak /proc permissions (how do you remove appArmor restrictions)

Host File Descriptor Leakage (some symlinks must be followed only on container not on the hosts)

Symslinks

## runC exploit

containered → containerd-shim → runc(unshare pid 1 etc)

if anything symlnk to runC then you can access runC which is outside container

don't use rkt(now deprecated?) it doesn't drop any privileges and creates a container with root access better alternative Podman


  

if a process running in container points to proc then you have access to runC running on host 

## Bad Ideas

- do not exposes Docker socket which is used to by client to talk to docker daemon you can talk to sockets using curl  (these are usually used to run docker inside of docker)
- privileged flag (no appArmor no Linux SE) —> use user mode helper programs to escape from privilege containers  these are invoked by kernel in a privileged context  (overlay fs)  more like VCS for containers? release_Agent, binfmt_misc, core_pattern, uevent_helper,modprobe


## Attack Scenario #1

Pre req:

- Root access
- mount Syscall

### Notify_on_release (executed by the host)

Intuition: find a host process that is interacting with container(maybe for monitoring) and see how you can interact with it. 

when the last task in the cgroup leaves then kernel runs the command specified in the release_agent file. what's the point of this?? maybe once use case is to notify the kernel all tasks are terminated hence this group can be removed 

```bash
docker run --rm -it --cap-add=SYS_ADMIN --security-opt apparmor=unconfined ubuntu bash
```




This command gives us the shell with root privileges of the container and SYSADMIN capabilities 

```bash
mkdir /tmp/cgrp && mount -t cgroup -o memory cgroup /tmp/cgrp && mkdir /tmp/cgrp/x

```

## Vulnerable base image

- Shell shock
- Dirty cow use this to modify   Virtual dynamic shared objects (vDSO) mapping

I am part of docker group (user/ group in linux)

this is an interesting case in a CI/CD pipeline Jenkins can run docker with root if you have access to Jenkins which can create docker then you can escalate to root on the host system.

Steps to reproduce you can mounts volumes and create a file with set uid bit set and mount to that path.

so you will have that setuid file on host and execute it for root access


Capbilities: SYS_ADMIN/ SYS_MODULE

Dangerous mount points

## Docker Sock

/var/run/docker.sock (docker daemon)  if it has this volume mounted then you can interact with docker and break out

this volume mounted usually for monitoring other containers when you access to docker container see if you can access this monitoring container

Does Docker now have a user namespace so if I am root inside the container and if I escape docker container i am  root on host due to lack of user namespace

No kernel keyspace namespace where crypto secrets are stored is not a separate namespace

## Hardening

if there are mutiple docker containers ensure talk to each other through network and not through host system

Don't mount /etc /dev  /proc  AppArmor Protects from /proc but you can mount it as /abc/proc/ to bypass protection

set U limits?

Don't run on privileged port? privileged port can access some capabilities 

have one Network interface?

why you shouldn't ssh into your docker contianer 

why do you need to signed containers when you have signed manifests?

Don't use online config management in your container?

containers are immutable 

if you want to make change spin up new container and stop the old one

 


## Auditing

docker-benchmark tool on github
look into 2375 and 2376 API host access like Kubernetes 10250 and 10255
