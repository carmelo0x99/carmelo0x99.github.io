---
layout: post
title: "Docker Swarm on ClusterHAT"
categories: misc
---

## Docker Swarm on ClusterHAT
I've had these nice Raspberry Pi Zero (vanilla, not 'W') lying around for quite some time and I wanted to give them a purpose in life.

By chance (meaning, during my _endless browsing of the Twitter-verse and the Reddit-verse_) I have discovered the ClusterHAT (**H**ardware **A**ttached on **T**op) which "_interfaces a (Controller) Raspberry Pi A+/B+/2/3 with 4 Raspberry Pi Zeros configured to use USB Gadget mode_". To discover more: https://clusterhat.com/.

After installing, breaking, fixing, re-installing multiple times I've found myself with a nice-and-cheap Docker Swarm cluster.

This is the (short) story of how I've put them all to work together and how I've learned a bit about the following topics:
- [Docker Swarm](https://docs.docker.com/engine/swarm/)
- [Docker multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)
- [Docker swarm mode routing mesh](https://docs.docker.com/engine/swarm/ingress/)

INDEX
- [The basics: hardware architecture](#basics)
- [Quick fix: start from a compatible base layer, arm32v6 images!](#quick)
- [Docker Swarm, finally](#swarm)

**NOTE**: during this tutorial I'll be using **ARM32v6** images since the Controller and the Nodes do not share a common hardware architecture. This topic, along with [Buildx](https://www.docker.com/blog/multi-arch-images/), shall wait for another article.
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

**WARNING**: the test app is a (very) silly one. Please do not run any reality checks on the app itself, just focus on the overall process :)
`Dockerfile`:
```
FROM golang:alpine AS builder
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build -o hello .

FROM alpine
RUN mkdir /app
WORKDIR /app
COPY --from=builder /app/hello .
CMD ["./hello"]
```

**NOTE**: the code above is for a [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/). Read more on the topic on [docs.docker.com](https://docs.docker.com/develop/develop-images/multistage-build/).

`hello.go`:
```
package main

import (
    "fmt"
    "os"
    "runtime"
)

func main() {
    hostname, _ := os.Hostname()
    fmt.Printf("Hello World, I'm %s running on %s/%s\n", hostname, runtime.GOOS, runtime.GOARCH)
}
```

The first step is to build an image on the Controller and run it:
```
pi@ctrl $ docker build -t carmeloc/multiarch:1.0 .
Sending build context to Docker daemon  3.072kB
Step 1/10 : FROM golang:alpine AS builder
...
Successfully tagged carmeloc/multiarch:1.0

pi@ctrl $ docker run --rm --name multiarch carmeloc/multiarch:1.0
Hello World, I'm 61b41404280c running on linux/arm
```

Let's do the same on the Pi Zeros.
```
pi@ctrl $ docker push carmeloc/multiarch:1.0

pi@zero $ docker run --name multiarch10 carmeloc/multiarch:1.0

pi@zero $ docker logs multiarch10
```

**No output, no logs... not good!**

<a name="quick"></a>
### Quick fix: start from a compatible base layer, arm32v6 images!
Let's edit `Dockerfile` as follows:
```
pi@ctrl $ diff Dockerfile.ori Dockerfile
1c1
< FROM golang:alpine AS builder
---
> FROM arm32v6/golang:alpine AS builder
7c7
< FROM alpine
---
> FROM arm32v6/alpine
```

Let's build the image again:
```
pi@ctrl $ docker build -t carmeloc/multiarch:1.1 .
Sending build context to Docker daemon  4.096kB
Step 1/10 : FROM arm32v6/golang:alpine AS builder
...
Successfully tagged carmeloc/multiarch:1.1
```

Notice how the output shows we're starting from an arm32v6 image now.
Nothing has changed on the Controller:
```
pi@ctrl $ docker run --rm --name multiarch carmeloc/multiarch:1.1
Hello World, I'm 7c6a45c23504 running on linux/arm
```

Let's push the new image onto Docker Hub...
```
pi@ctrl $ docker push carmeloc/multiarch:1.1
```

... then pull it from the Zeros and run it:
```
pi@zero $ docker pull carmeloc/multiarch:1.1

pi@zero $ docker run --name multiarch11 carmeloc/multiarch:1.1
Hello World, I'm 574f49b7c3b9 running on linux/arm
```

**Problem resolved**!?!

Er, yes, but that leaves us a bit unimpressed, no? There's not a lot of information on the why-and-wherefore, right?

<a name="swarm"></a>
### Docker Swarm, finally
Evidently, the container prints one message then exits. Let's make one last change to `Dockerfile` as follows:
```
pi@ctrl $ diff Dockerfile.ori.2 Dockerfile
11c11
< CMD ["./hello"]
---
> CMD ["sh", "./loop.sh"]
```

... and add `loop.sh` to our files:
```
#!/bin/sh

while true
do
    ./hello
    sleep 1
done
```

The deployment onto Docker Swarm can now take place.
```
pi@ctrl $ docker service create --name=testbuildx --replicas=3 carmeloc/multiarch:2.1
ignplys08j32yfp3u2cu7ngbf
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged

pi@ctrl $ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                    PORTS
ignplys08j32        testbuildx          replicated          3/3                 carmeloc/multiarch:2.1

pi@ctrl $ docker service ps testbuildx
ID                  NAME                IMAGE                    NODE                     DESIRED STATE       CURRENT STATE            ERROR               PORTS
wv0v5x89w1xb        testbuildx.1        carmeloc/multiarch:2.1   ctrl                     Running             Running 8 minutes ago
ri209phiw7j1        testbuildx.2        carmeloc/multiarch:2.1   zero1                    Running             Running 31 seconds ago
uxcrmterlogp        testbuildx.3        carmeloc/multiarch:2.1   zero2                    Running             Running 11 seconds ago

pi@ctrl $ docker service logs testbuildx
testbuildx.1.wv0v5x89w1xb@ctrl    | Hello World, I'm f4f3089c4024 running on linux/arm
...
testbuildx.2.ri209phiw7j1@zero1    | Hello World, I'm 7d5a79895987 running on linux/arm
...
testbuildx.3.uxcrmterlogp@zero2    | Hello World, I'm ab8a4c337712 running on linux/arm
...
```

The final result is, as anticipated, that one application is run, replicated across multiple nodes/architectures, by means of Docker Swarm + ClusterHAT.

