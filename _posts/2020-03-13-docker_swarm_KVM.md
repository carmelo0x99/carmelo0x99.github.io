---
layout: post
title: "Docker Swarm on KVM (CentOS)"
categories: misc
---

## Docker Swarm on KVM (CentOS)
Fast re-run of the [previous post](https://carmeloc.github.io/misc/2020/03/10/docker_swarm_clusterhat.html), this time streamlined to match VM's on [KVM](https://www.cyberciti.biz/faq/how-to-install-kvm-on-centos-7-rhel-7-headless-server/).

One thing I like about KVM is the ability to have _templates_ and quickly install VM's through [Kickstart](https://docs.centos.org/en-US/centos/install-guide/Kickstart2/).

Let's just assume we have two VM's: swarm-master and swarm-node1. Installation of Docker follows the usual [guidelines](https://docs.docker.com/install/linux/docker-ce/centos/):
```
$ sudo yum upgrade -y && sudo yum install -y git yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum install -y docker-ce docker-ce-cli containerd.io
$ sudo systemctl start docker && sudo systemctl enable docker
```

After the steps above a new "network" will be configured in the guest OS, e.g.:
```
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:10:e6:53:1d brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
This is how containers will ultimately connect with the outside world.

The following step is recommended to run Docker as an ordinary user:
```
$ sudo usermod -aG docker <userID>
```

Finally, rather than recommending to disable the firewall (acceptable in a lab environment, but...), run the following:
```
sudo firewall-cmd --permanent --add-port=2377/tcp --add-port=4789/udp --add-port=7946/tcp --add-port=7946/udp
sudo firewall-cmd --reload
```

The following code will be run:
`main.go`
```
package main

import (
    "net/http"
    "os"
    "runtime"
    "strings"
)

func sayHello(w http.ResponseWriter, r *http.Request) {
    hostname, _ := os.Hostname()
    message := r.URL.Path
    message = strings.TrimPrefix(message, "/")
    message = "This is " + hostname + " running on " + runtime.GOOS + "/" + runtime.GOARCH + " saying: " + message + "\n"

    w.Write([]byte(message))
}

func main() {
    http.HandleFunc("/", sayHello)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}
```

A container is automatically built by means of Docker Hub's [automated builds](https://docs.docker.com/docker-hub/builds/) feature. The code is stored on [GitHub: carmeloc/GoWeb](https://github.com/carmeloc/GoWeb) while the container can be pulled from [Docker Hub: ccarmelo/goweb](https://hub.docker.com/repository/docker/ccarmelo/goweb).

Now to the juiciest part, running the app through Docker Swarm.
```
$ docker service create --name=s_goweb --publish 8080:8080 --replicas=2 mellowiz/goweb:latest
```

The image will be automatically pulled for us and run.
```
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                   PORTS
l0poe478uir2        s_goweb             replicated          2/2                 mellowiz/goweb:latest

$ docker service ps s_goweb
ID                  NAME                IMAGE                   NODE                              DESIRED STATE       CURRENT STATE            ERROR               PORTS
tozde8s3q02b        s_goweb.1           mellowiz/goweb:latest   swarm-master-kvm.ext.network.mw   Running             Running 19 seconds ago
tixu0tms1fdu        s_goweb.2           mellowiz/goweb:latest   swarm-node1-kvm.ext.network.mw    Running             Running 19 seconds ago
```

Testing the app:
```
$ curl http://10.0.2.170:8080/Hello
This is 7be668bab228 running on linux/amd64 saying: Hello

$ curl http://10.0.2.170:8080/Hello
This is 9d19216cb808 running on linux/amd64 saying: Hello
```

