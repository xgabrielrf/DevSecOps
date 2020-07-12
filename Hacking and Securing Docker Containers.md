# Hacking and Securing Docker Containers



## Namespaces

### Editing read-only files as root from containers

Normally we run containers with our users, with or without sudoers.
Even if we are only in a single docker group, it allows you to run docker.
But we have a problem in this case. A **security** problem.

```bash
gle003@vb-ubuntu:~$ ll /tmp/file.txt
-rw-r--r-- 1 root root 36 jul 11 14:53 /tmp/file.txt
```
> ​	*file.txt* was created with root user.

As you can see, *file.txt* has **no write** to other users than root.

```bash
gle003@vb-ubuntu:~$ cat /tmp/file.txt
File from VM. Out of the container.
gle003@vb-ubuntu:~$ echo "Edit from non-root user" >> /tmp/file.txt
-su: /tmp/file.txt: Permission denied
```
If we start a container **sharing a folder from the machine**, you can login inside the container and have root privileges, as I show below:

```bash
#Creating a new container and sharing /tmp/
gle003@vb-ubuntu:~$ docker run -itd -v /tmp/:/shared/ --name alpine_broken alpine
3be059a0473ddc9cbe33c1a95ba4e52d25525d912cd8ef5a01f1da0066c7a6eb
gle003@vb-ubuntu:~$ docker exec -it alpine_broken sh

#Testing if it is possible to edit file.txt from the container
~ # ls -l /shared/file.txt
-rw-r--r--    1 root     root            36 Jul 11 17:53 /shared/file.txt
~ # echo "Edit from container." > /shared/file.txt
~ # cat /shared/file.txt
Edit from the container.
~ # exit

#Seeing that it changed from the machine
gle003@vb-ubuntu:~$ cat /tmp/file.txt
Edit from the container.
```

As you can see, even if the file is writable only for root user, I can edit it from the container.

> **Why?**
>
> Because root user from the container has the same uid that root user has in the machine.



### User namespaces for isolation between containers and hosts

Now I'll tell you how to *fix* this problem.

We will work with **user namespaces** for isolation between containers and hosts.

Let's edit **docker.service** to start with **userns-remap**:

```bash
gle003@vb-ubuntu:~$ sudo vim /lib/systemd/system/docker.service
```

Look for **ExecStart** and, at the end of the line add **--userns-remap=default**.

It will looks like the line below:

```sh
[Service]
Type=notify
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --userns-remap=default
```

After you change it, you need to reload the daemon and restart the docker.service.

```bash
gle003@vb-ubuntu:~$ sudo systemctl daemon-reload
gle003@vb-ubuntu:~$ sudo systemctl restart docker
gle003@vb-ubuntu:~$ ps aux | grep docker
root     10479  0.4  4.0 831120 83004 ?        Ssl  03:30   0:00 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --userns-remap=default
gle003   10625  0.0  0.0  14428  1048 pts/0    S+   03:30   0:00 grep --color=auto docker
```

> You can see that docker already started with our new information, **--userns-remap=default**.



Now we will check if we still can edit files from the machine inside the containers.

```bash
#Remove old containers
gle003@vb-ubuntu:~$ docker rm `docker ps -aq`
67f92151a83d
dc20ce8d6903
gle003@vb-ubuntu:~$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

#Start a new one and share /tmp/ from machine
gle003@vb-ubuntu:~$ docker run -itd -v /tmp/:/shared/ --name alpine_hard alpine
2e852e1f0a0ece3b5be1e1baee79b51d4242751d2918cf6c06aadc1b398de91c
gle003@vb-ubuntu:~$ docker exec -it alpine_hard sh

#Try to edit the file.txt from the container
/ # ls -l /shared/file.txt
-rw-r--r--    1 nobody   nobody          21 Jul 12 06:12 /shared/file.txt
/ # echo "Edit again from container." > /shared/file.txt
sh: can't create /shared/file.txt: Permission denied
```

Now it is no more possible to edit outside files from containers using **-v** option.