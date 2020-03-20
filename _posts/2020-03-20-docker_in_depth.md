---
layout: post
title: "Docker in depth"
categories: misc
---

INDEX
- [Images and containers](#images)
- [Namespaces](#ns)
- [Cgroups](#cg)

<a name="images"></a>
### Images and containers
Let's take a closer look at what's in an image.

**NOTE**: I'm filtering out lots of info by using Docker's [format command](https://docs.docker.com/config/formatting/).
```
user@laptop$ docker inspect --format="{{json .RootFS.Layers}}" ccarmelo/goweb:latest
["sha256:5216338b40a7b96416b8b9858974bbe4acc3096ee60acbc4dfb1ee02aecceb10",
 "sha256:a0b2ea330c61ec1ec3d25024a8ddaa6121e995e2e3dc2473c48bfdeb7adfab69",
 "sha256:4b7b5c980fbe0abe030c29236a05764ea3c32f898d56495b2bc146d6b82a2c3d"]
```

Likewise, `docker history` can show how the image had been made:
```
user@laptop$ docker history ccarmelo/goweb:latest
IMAGE          CREATED        CREATED BY                                      SIZE    COMMENT
ccf5bf7f5979   7 days ago     /bin/sh -c #(nop)  CMD ["./main.go"]            0B
<missing>      7 days ago     /bin/sh -c #(nop) COPY file:fe2451faf4c4dbce…   7.47MB
<missing>      7 days ago     /bin/sh -c #(nop) WORKDIR /app                  0B
<missing>      7 days ago     /bin/sh -c mkdir /app                           0B
<missing>      2 months ago   /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>      2 months ago   /bin/sh -c #(nop) ADD file:e69d441d729412d24…   5.59MB
```

Let's inspect the actual container now.
```
user@laptop$ docker container ls
CONTAINER ID   IMAGE                   COMMAND       CREATED          STATUS          PORTS   NAMES
66df9337ad51   ccarmelo/goweb:latest   "./main.go"   14 minutes ago   Up 14 minutes           beautiful_williamson
```

In reality, the application is obviously running on our host, we can see how it's identified by its PID:
```
user@laptop$ ps -ef | grep main.go | grep -v grep
root      3993  3970  0 17:38 ?        00:00:00 ./main.go

user@laptop$ docker inspect 66df9337ad51 | grep -i pid
            "Pid": 3993,
            "PidMode": "",
            "PidsLimit": null,
```

We can also connect to the running container. From within, it looks as if we're in a totally separate environment:
```
user@laptop$ docker exec -it 66df sh

/app # hostname
66df9337ad51

/app # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

/app # ls
main.go

/app # ls /
app    bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var

/app # exit
```

<a name="ns"></a>
### Namespaces
Docker uses a technology called _namespaces_ to provide the container with a layer of isolation. A few examples:
- *pid* namespace: Process isolation (PID: Process ID).
- *net* namespace: Managing network interfaces (NET: Networking).
- *ipc* namespace: Managing access to IPC resources (IPC: InterProcess Communication).
- *mnt* namespace: Managing filesystem mount points (MNT: Mount).
- *uts* namespace: Isolating kernel and version identifiers. (UTS: Unix Timesharing System).

Let's analyze namespaces by, for instance, inheriting the _hostname_:
```
user@laptop$ sudo nsenter --target 3993 --uts
root@66df9337ad51:~# hostname
66df9337ad51

root@66df9337ad51:~# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp14s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000
    link/ether 30:f9:ed:fe:b1:28 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DORMANT group default qlen 1000
    link/ether 08:ed:b9:ce:e2:cf brd ff:ff:ff:ff:ff:ff
4: docker_gwbridge: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:51:dc:ce:e9 brd ff:ff:ff:ff:ff:ff
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:7c:5c:a8:3e brd ff:ff:ff:ff:ff:ff
7: veth30cdacf@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether 52:6a:51:42:8f:b8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
9: veth52187b1@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether 82:b7:dc:96:1b:58 brd ff:ff:ff:ff:ff:ff link-netnsid 1

root@66df9337ad51:~# exit
```

Let's now try something different and borrow the container's network settings:
```
user@laptop$ sudo nsenter --target 3993 --net

root@laptop:~# hostname
laptop

root@laptop:~# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0

root@laptop:~# exit
```

<a name="cg"></a>
### Cgroups
Docker Engine on Linux also relies on another technology called _control groups_ (or _cgroups_). A cgroup limits an application to a specific set of resources. Control groups allow Docker Engine to share available hardware resources to containers and optionally enforce limits and constraints.

Let's see an example of how memory can be limited for a container.
```
user@laptop$ cat /proc/3993/cgroup
12:net_cls,net_prio:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
11:memory:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
10:pids:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
9:devices:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
8:cpuset:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
7:blkio:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
6:hugetlb:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
5:rdma:/
4:perf_event:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
3:freezer:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
2:cpu,cpuacct:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
1:name=systemd:/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86
0::/system.slice/containerd.service

user@laptop$ cat /sys/fs/cgroup/memory/docker/66df9337ad51dd25ed8befe778bfe19698df8636a3fbfb45c4257899d93d9a86/memory.limit_in_bytes
9223372036854771712

user@laptop$ docker run -d --memory 4m --name test4m ccarmelo/goweb:latest
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
051f309ee14220936dd6a746341cde687c94bca45108a9413ba7fcc1ee323520

user@laptop$ cat /sys/fs/cgroup/memory/docker/051f309ee14220936dd6a746341cde687c94bca45108a9413ba7fcc1ee323520/memory.limit_in_bytes
4194304
```
