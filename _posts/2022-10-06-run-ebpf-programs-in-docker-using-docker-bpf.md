---
layout: page
title: Run eBPF Programs in Docker using docker-bpf
permalink: /run-ebpf-programs-in-docker-using-docker-bpf/
description: |
  Run eBPF programs in Docker containers with the new tool docker-bpf,
  including fixes for common issues.
---

eBPF programs are very powerful.
They can be used to profile system calls, manipulate network packets, and enhance security.
However, running eBPF programs in Docker containers is not as straightforward as it seems,
especially Docker Desktop for Mac/Windows.

In this post, I will show you how to run eBPF programs in Docker containers,
and introduce my new tool [docker-bpf](https://github.com/hemslo/docker-bpf) to simplify the process.

We can use bpftrace as an example to demonstrate how to run eBPF programs in Docker containers.
According to the [bpftrace documentation](https://github.com/iovisor/bpftrace/blob/master/INSTALL.md#docker-images),
it should run as this:

```shell
docker run -ti -v /usr/src:/usr/src:ro \
        -v /lib/modules/:/lib/modules:ro \
        -v /sys/kernel/debug/:/sys/kernel/debug:rw \
        --net=host --pid=host --privileged \
        quay.io/iovisor/bpftrace:latest \
        bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

However, the command will fail on Docker Desktop for Mac/Windows.
The error may like this:

```text
stdin:1:1-34: ERROR: tracepoint not found: raw_syscalls:sys_enter
tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Let's see how to fix it.

## debugfs

`debugfs` is not mount by default on Docker Desktop for Mac/Windows. We can mount it using command

```shell
mount -t debugfs debugfs /sys/kernel/debug
```

But this command is not convenient to run, we either need to run it every time we start a container, or need to run it in the host VM. There is a better way, using docker volume.

```shell
docker volume create --driver local --opt type=debugfs --opt device=debugfs debugfs
```

Then we can mount it in the container using

```shell
docker run -ti -v /usr/src:/usr/src:ro \
        -v /lib/modules/:/lib/modules:ro \
        -v debugfs:/sys/kernel/debug:rw \
        --net=host --pid=host --privileged \
        quay.io/iovisor/bpftrace:latest \
        bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

Now the error message will be:

```text
/bpftrace/include/clang_workarounds.h:14:10: fatal error: 'linux/types.h' file not found
```

## Linux Kernel headers

Some eBPF programs require Linux kernel headers to compile.
It's usually available in `/usr/src/linux-headers-$(uname -r)`, which is generated when compiling the kernel with `CONFIG_IKHEADERS` enabled.
However, the default kernel [linuxkit](https://github.com/linuxkit/linuxkit) in Docker does not enable this option.
The good news is that the headers are available in the [linuxkit/kernel](https://hub.docker.com/r/linuxkit/kernel) image.
So we can use the following Dockerfile to build a new image with kernel headers:

```dockerfile
ARG KERNEL_VERSION
FROM linuxkit/kernel:${KERNEL_VERSION} as kernel
FROM busybox
WORKDIR /
COPY --from=kernel /kernel-dev.tar .
RUN tar xf kernel-dev.tar
```

The kernel version can be get by

```shell
docker run --rm -it busybox uname -r
```

so we can build the image with

```shell
docker build --build-arg KERNEL_VERSION="${KERNEL_VERSION}" -t linuxkit-kernel-headers:"$KERNEL_VERSION" .
```

Then we can create a new data volume with the headers

```shell
docker run --rm -v linuxkit-kernel-headers:/usr/src linuxkit-kernel-headers:"$KERNEL_VERSION"
```

Now we can mount the headers in the container using command

```shell
$ docker run -ti -v linuxkit-kernel-headers:/usr/src:ro \
        -v /lib/modules/:/lib/modules:ro \
        -v debugfs:/sys/kernel/debug:rw \
        --net=host --pid=host --privileged \
        quay.io/iovisor/bpftrace:latest \
        bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

Attaching 1 probe...
^C

@[docker-init]: 2
@[dockerd]: 11
@[vpnkit-bridge]: 15
@[bpftrace]: 18
@[containerd-shim]: 22
@[containerd]: 439
```

It works!

## BTF

BTF (BPF Type Format) provides type information for eBPF programs, so it can run without kernel headers. Linuxkit kernel does not enable BTF by default, if you are on Windows, you can switch to WSL 2 based engine, the latest kernel in WSL 2 has BTF enabled.

The BTF information is available in `/sys/kernel/btf/vmlinux`, we can mount it in the container using command

```shell
docker run -ti -v /usr/src:/usr/src:ro \
        -v /lib/modules/:/lib/modules:ro \
        -v debugfs:/sys/kernel/debug:rw \
        -v /sys/kernel/btf/vmlinux:/sys/kernel/btf/vmlinux:ro \
        --net=host --pid=host --privileged \
        quay.io/iovisor/bpftrace:latest \
        bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

## docker-bpf

To do all above steps, we need to run a lot of commands, and it's not easy to remember them,
so I created [docker-bpf](https://github.com/hemslo/docker-bpf) to simplify the process.

It's a simple wrapper around `docker` command, it will mount the required volumes automatically.

```shell
docker run --rm -ti -v /var/run/docker.sock:/var/run/docker.sock ghcr.io/hemslo/docker-bpf:latest
```

This will launch a container with bcc and bpftrace installed, it also supports custom bpf program image and other options.

A fully customized example:

```shell
docker run --rm -ti \
    -e DATA_MOUNT=$PWD:/data \
    -e BPF_IMAGE=quay.io/iovisor/bpftrace:latest \
    -v /var/run/docker.sock:/var/run/docker.sock \
    ghcr.io/hemslo/docker-bpf:latest \
    bpftrace --info
```

Hope this tool will make eBPF more accessible to developers. Happy eBPF hacking!
