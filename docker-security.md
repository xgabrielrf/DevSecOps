> Being created

Pass through them all:

https://docs.docker.com/engine/security/security/



Start with docker bench. Trying to get the best score.

> REMEMBER TO USE DOCKER FILE AND ANSIBLE PLAYBOOKS!!!!!!!!!!



## Docker Bench for Security

The Docker Bench for Security is a script that checks for dozens of common best-practices around deploying Docker containers in production.



The easiest way to run your hosts against the Docker Bench for Security is by running our pre-built container:

```bash
docker run -it --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -v /etc:/etc:ro \
    -v /usr/bin/containerd:/usr/bin/containerd:ro \
    -v /usr/bin/runc:/usr/bin/runc:ro \
    -v /usr/lib/systemd:/usr/lib/systemd:ro \
    -v /var/lib:/var/lib:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    --label docker_bench_security \
    docker/docker-bench-security
```



More about it in [Bench for Security](https://github.com/docker/docker-bench-security).



## Security

https://docs.docker.com/engine/security/security/



## The apparmor profiles

using

include in a file called apparmor-profile

```bash
#include <tunables/global>
profile apparmor-profile flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  file,
  network,
  capability,
  deny /tmp/** w,
  deny /etc/passwd rwklx,
}
```

Then

```bash
sudo apparmor_parser -r -W apparmor-profile
#Running with security opt
docker run -itd --security-opt apparmor=apparmor-profile --name apparmor alpine
```



after that see if in the bench for security you got an greater score :)





More about it in [Apparmor Profiles](https://docs.docker.com/engine/security/apparmor/).



## seccomp

allows or blocks users sending commands