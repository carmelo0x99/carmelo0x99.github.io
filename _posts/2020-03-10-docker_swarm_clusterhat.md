---
layout: post
title: "Docker Swarm on ClusterHAT"
categories: misc
---

INDEX
- [The basics: hardware architecture](#basics)
- [Our app, an HTTP _server_](#app)
- [Docker Swarm, finally!](#swarm)

**NOTE**: during this tutorial I'll be using **ARM32v6** images since the Controller and the Nodes do not share a common hardware architecture. This topic, along with [Buildx](https://www.docker.com/blog/multi-arch-images/), shall wait for another article.

[Installation of ClusterHAT](https://carmeloc.github.io/misc/2020/03/09/raspi_clusterhat_install.html) has been described in a [previous post](https://carmeloc.github.io/misc/2020/03/09/raspi_clusterhat_install.html).

<a name="basics"></a>
### The basics: hardware architecture
Controller:
```
pi@ctrl $ docker info | grep Architecture
 Architecture: armv7l
```

Pi Zeros:
```
pi@zero $ docker info | grep Architecture
 Architecture: armv6l
``` 

Notice how the Controller is based on **ARMv7** while the Zeros are based on **ARMv6**.

<a name="app"></a>
### Our app, an HTTP _server_:
`Dockerfile.arm`:
```
FROM arm32v6/golang:alpine AS builder
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build -o main.go .

FROM arm32v6/alpine
RUN mkdir /app
WORKDIR /app
COPY --from=builder /app/main.go ./
CMD ["./main.go"]
```

**NOTE**: the code above is for a [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/). Read more on the topic on [docs.docker.com](https://docs.docker.com/develop/develop-images/multistage-build/).

`main.go`:
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

The first step is to build an image on the Controller and run it:
```
pi@ctrl $ docker build -t carmeloc/goweb:1.0 -f Dockerfile.arm .
Sending build context to Docker daemon  110.6kB
...
Successfully built 1f4eef919f7b
Successfully tagged carmeloc/goweb:1.0

pi@ctrl $ docker image ls
REPOSITORY       TAG         IMAGE ID         CREATED           SIZE
carmeloc/goweb   1.0         11c04ac64701     11 minutes ago    11.5MB   <<< new image
<none>           <none>      bc2d8f7dafec     11 minutes ago    349MB
arm32v6/golang   alpine      af1c8ccb1a0f     2 weeks ago       342MB    <<< intermediate layers
arm32v6/alpine   latest      8093515ca679     8 weeks ago       4.81MB   <<< intermediate layers
```

**NOTE**: a comparison between the size of our image and the arm32v6/golang image shows a dramatic reduction is size. This is the main benefit of [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/).

Let's run our image:
```
pi@ctrl $ docker run -d -p 8888:8080 carmeloc/goweb:1.0
a8c0d0b5c784251f2afb374c3ed156095ed82004f8501bb4887bb5dedecb07e4
```

... and from any host connected to the same subnet:
```
user@host $ curl http://10.0.2.207:8888/Hello!
This is a8c0d0b5c784 running on linux/arm saying: Hello!
```

Notice how the _hostname_ matches the container ID on the controller:
```
pi@ctrl $ docker container ls
CONTAINER ID     IMAGE                COMMAND       CREATED          STATUS          PORTS                    NAMES
a8c0d0b5c784     carmeloc/goweb:1.0   "./main.go"   14 minutes ago   Up 14 minutes   0.0.0.0:8888->8080/tcp   happy_spence
```

Once we're satified with our image, we can push it to Docker Hub:
```
pi@ctrl $ docker push carmeloc/goweb:1.0
```
**NOTE**: the step above is quite important because Docker Hub doesn't currently build images for ARM processors.

<a name="swarm"></a>
### Docker Swarm, finally!
[Docker swarm mode](https://docs.docker.com/engine/swarm/key-concepts/) refers to the cluster management and orchestration features embedded in the Docker Engine.
I'll skip the swarm creation part, well documented, and focus on running and scaling our application.

Let's take a look at our cluster:
```
pi@ctrl $ docker node ls
ID                  HOSTNAME  STATUS      AVAILABILITY    MANAGER STATUS    ENGINE VERSION
74u4z2qw95w0ctz *   ctrl      Ready       Active          Leader            19.03.8
u0ayftfqi4z728p     zero1     Ready       Active                            19.03.8
go7epn59usoc50f     zero2     Ready       Active                            19.03.8
kz267mzscz0ol8h     zero3     Ready       Active                            19.03.8
rv24m6fuipawylf     zero4     Ready       Active                            19.03.8
```

Let's create a new service as follows:
```
pi@ctrl $ docker service create --name goweb --replicas 3 --publish published=8888,target=8080 carmeloc/goweb:1.0
ox51ocpvar7sx1f
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```

The service can be displayed and inspected as follows:
```
pi@ctrl $ docker service ls
ID              NAME           MODE                REPLICAS       IMAGE                PORTS
htlgxrtss1ox    goweb          replicated          3/3            carmeloc/goweb:1.0   *:8888->8080/tcp

pi@ctrl $ docker service ps goweb
ID              NAME           IMAGE                NODE      DESIRED STATE     CURRENT STATE                ERROR      PORTS
fmm9uf112y6j    goweb.1        carmeloc/goweb:1.0   zero2     Running           Running about a minute ago
w44yl47vtnzi    goweb.2        carmeloc/goweb:1.0   ctrl      Running           Running about a minute ago
i5ky88du3kbp    goweb.3        carmeloc/goweb:1.0   zero1     Running           Running about a minute ago

pi@ctrl $ docker service inspect --format="{{json .Endpoint.Spec.Ports}}" goweb
[{"Protocol":"tcp","TargetPort":8080,"PublishedPort":8888,"PublishMode":"ingress"}]

pi@ctrl $ docker service inspect --format="{{json .Spec.TaskTemplate.ContainerSpec.Image}}" goweb
"carmeloc/goweb:1.0@sha256:***"
```

The test is, once agin, run from a different host. Notice how three replicas are responding in round-robin fashion:
```
user@host $ curl http://10.0.2.207:8888/Hello\!
This is 0801fa03baa6 running on linux/arm saying: Hello!

user@host $ curl http://10.0.2.207:8888/Hello\!
This is d3ffa4006aa8 running on linux/arm saying: Hello!

user@host $ curl http://10.0.2.207:8888/Hello\!
This is fbd04731918f running on linux/arm saying: Hello!

user@host $ curl http://10.0.2.207:8888/Hello\!
This is 0801fa03baa6 running on linux/arm saying: Hello!
```
Just like before, the hostnames match the actual containers running on the various nodes.
