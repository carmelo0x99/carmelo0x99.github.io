---
layout: post
title: "Docker Swarm on ClusterHAT"
categories: misc
---

INDEX
- [The basics: hardware architecture](#basics)
- [Our app, an HTTP _server_](#app)
- [Docker Swarm, finally](#swarm)

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
Our app, an HTTP _server_:
`Dockerfile`:
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

func sayHello(w http.ResponseWriter, r http.Request) {
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
$ docker build -t mellowiz/goweb:1.0 .
Sending build context to Docker daemon  101.9kB
...
Successfully built 11c04ac64701
Successfully tagged mellowiz/goweb:1.0

$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mellowiz/goweb      1.0                 11c04ac64701        11 minutes ago      11.5MB   <<< new image
<none>              <none>              bc2d8f7dafec        11 minutes ago      349MB
arm32v6/golang      alpine              af1c8ccb1a0f        2 weeks ago         342MB    <<< intermediate layers
arm32v6/alpine      latest              8093515ca679        8 weeks ago         4.81MB   <<< intermediate layers
```

**NOTE**: a comparison between the size of our image and the arm32v6/golang image shows a dramatic reduction is size. This, again, is one of the benefits [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/).

Let's run our image:
```
$ docker run -d -p 8888:8080 mellowiz/goweb:1.0
a8c0d0b5c784251f2afb374c3ed156095ed82004f8501bb4887bb5dedecb07e4
```


