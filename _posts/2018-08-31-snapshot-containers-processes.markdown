---
layout: post
author: Valentin Rothberg
title:  "Snapshot Container Processes in Golang"
date:   2018-08-31
tags: containers snapshot go golang podman ps psgo kubernetes
---

This blog post is an introduction to the [psgo library](https://github.com/containers/psgo), a library implemented in [Go](https://golang.org) to snapshot processes and display various kinds of process-related data.  The `psgo` library is inspired by the old-but-gold [ps(1)](http://man7.org/linux/man-pages/man1/ps.1.html) tool that is commonly used on UNIX and Linux systems but `psgo` adheres to the concept of Linux containers (and Kubernetes PODs) and can therefore extract a much richer data set that can come in handy for container engines and container runtimes.

### PS(1) Is Hard To Parse

I started implementing the [psgo library](https://github.com/containers/psgo) as part of [SUSE Hack Week 17](https://hackweek.suse.com/).  Shortly before SUSE Hack Week, I was working on a [Podman](https://github.com/containers/libpod) issue that was caused by `podman-top` using `ps` to extract data from processes running within a container.  `podman-top` displays the running processes of a container and prints it in an easy to read columnized format similar to `ps -ef`:

```
USER   PID   PPID   %CPU    ELAPSED        TTY   TIME   COMMAND
root   1     0      0.000   5.997152239s   ?     0s     sleep 100
```

The problem with using `ps` for listing container processes is that its output cannot be parsed reliably when certain format descriptors or combinations of flags are specified.  The output is split into columns but `ps` splits columns only with whitespaces which some data fields may contain as well (see the `sleep 100` command above).  Other tools such as `grep(1)` allow splitting columns with special characters, for instance a null byte, but `ps` does not support that.  The `podman` maintainers and I were quite unhappy about the ps-parsing problem, so I wanted to work on the issue; I found it to be a great occasion to look a bit deeper into Linux container internals and the [proc filesystem](http://man7.org/linux/man-pages/man5/proc.5.html) in particular.  Notice that the `ps(1)` parsing problem is not specific to Podman but to many other container engines and runtimes, including Moby/Docker and runc.

# The `psgo` Library

The [psgo library](https://github.com/containers/psgo) is now a community project -- and I am really happy about it -- and is part of the larger [github.com/containers](https://github.com/containers) umbrella that is home to many libraries and tools in the containers ecosystem, such as [Podman](https://github.com/containers/libpod) (container engine) and [Skopeo](https://github.com/containers/skopeo) (container-image management).  The API of `psgo` is build around the concept of using ps(1) AIX-format descriptors with some useful additions for containers.  The default descriptors are identical to the ones when running `ps -ef` (as shown in the snippet above):

- **user**: for the effective user
- **pid**: for the process ID
- **ppid**: for the parent process ID
- **pcpu**: for the CPU consumption of the process
- **etime**: for the elapsed time since process creation
- **tty**: for the used TTY device
- **time**: for the cumulative CPU time
- **args**: for the executed program including its arguments

I do not want to be too redundant, so please refer to [github.com/containers/psgo](https://github.com/containers/psgo) to watch the details of `psgo`'s API and the complete set of descriptors.  However, there is a number of descriptors that I want to highlight in the context of containers:

- **capamb**: set of ambient capabilities
- **capbnd**: set of bounding capabilities
- **capeff**: set of effective capabilities
- **capinh**: set of inheritable capabilities
- **capprm**: set of permitted capabilities
- **hgroup**: the corresponding effective group of a container process on the host
- **hpid**: the corresponding host PID of a container process
- **huser**: the corresponding effective user of a container process on the host
- **label**: current security attributes of the process
- **seccomp**: seccomp mode of the process (i.e., disabled, strict or filter)
- **state**: process state codes (e.g, **R** for *running*, **S** for *sleeping*)

Those descriptors can come in handy when debugging issues but they are also great for the purpose of **showing what a container actually is**: it's just a Linux process (potentially with child processes) running in some dedicated namespaces under some cgroups with additional security layers such as a restricted set of capabilities, a restricted set of syscalls as constrained by seccomp, plus additional security (e.g., access controls) imposed by AppArmor or SELinux.

# Using The `psgo` Library

I want to keep this post rather compact, so let's dive directly into some usage examples of the `psgo` library by showing how [Podman](https://github.com/containers/libpod) uses it for the `podman-top` command:

## Default Descriptors
By default, `podman-top` lists the same data as `ps -ef`.  Just the time is displayed in the typical Go-ish way.
```
$ sudo podman run -d alpine sleep 100
$ sudo podman top -l
USER   PID   PPID   %CPU    ELAPSED        TTY   TIME   COMMAND
root   1     0      0.000   5.997152239s   ?     0s     sleep 100
```

## Specifying Descriptors
The default output can be changed by specifying the desired descriptors on the command line, just as we do below to check the user, security label (AppArmor or SELinux) and the seccomp mode.
```
$ sudo podman run -d alpine sleep 100
$ sudo podman top -l user label seccomp
USER   LABEL                            SECCOMP
root   libpod-default-0.8.4 (enforce)   filter
```

## Namespaces
The `psgo` library is pretty good at detecting the various namespaces a container is running in.  To show an example, let's run a container with `Podman` in a user namespace and extract the effective user and group, both from within the container and how the process looks like on the host.
```
$ sudo podman run -d --uidmap=0:300000:70000 --gidmap=0:100000:70000 alpine sleep 100
$ sudo podman top -l pid, user, huser, group, hgroup
PID   USER   HUSER    GROUP   HGROUP
1     root   300000   root    100000
```

You can see that `podman-top` can extract the desired data both, from within the container and outside of it.  Same works also for PODs by using `podman-pod-top` which works pretty much the same way as shown above.

So much to the `psgo` library and how `Podman` is using it.  At that point, I want to thank SUSE for giving its employees the chance to work on anything they want during Hack Week.  I also want to thank Red Hat's [Giuseppe Scrivano](https://twitter.com/gscrivano) and [Daniel Walsh](https://twitter.com/rhatdan) for helping figuring out some hairy namespace details and for the great collaboration on the mentioned projects.  As I have mentioned before: this is how open source should be.
