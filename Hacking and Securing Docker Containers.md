# Hacking and Securing Docker Containers

### User namespaces for isolation between containers and hosts

Normally we run containers with our own users, with or without sudoers.
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
If we start a container sharing a folder from the machine, you can login inside the container and have root privileges, as I show above:

```bash

```

