---
layout: post
title: "Buildx: building multi-arch images"
categories: misc
---

## Buildx: building multi-arch images
In this post I'll explore how to build images that can run on multiple architectures. My scenario is composed of the two following computers:
1.
  ```
  $ docker info | grep Architecture
   Architecture: x86_64

  $ cat /proc/cpuinfo | grep "model name" | uniq
  model name	: Intel(R) Core(TM) i5-3317U CPU @ 1.70GHz
  ```
2.
  ```
  $ docker info | grep Architecture
   Architecture: armv7l

  $ cat /proc/cpuinfo | grep "model name" | uniq
  model name	: ARMv7 Processor rev 4 (v7l)
  ```

### On x86_64
`hello.go`:
```
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!\n")
}
```

`Dockerfile`:
```
FROM golang:alpine AS builder
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build -o hello.go .

FROM alpine
RUN mkdir /app
WORKDIR /app
COPY --from=builder /app/hello.go ./
CMD ["./hello.go"]
```

```
user@x86_64 $ docker build -t ccarmelo/hellogo:1.0 .

user@x86_64 $ docker run ccarmelo/hellogo:1.0
Hello, world!
```

### on ARM
```
user@arm $ docker run ccarmelo/hellogo:1.0
standard_init_linux.go:211: exec user process caused "exec format error"
```

### buildx
```
user@x86_64 $ export DOCKER_CLI_EXPERIMENTAL=enabled

user@x86_64 $ docker buildx version
github.com/docker/buildx v0.3.1-tp-docker 6db68d029599c6710a32aa7adcba8e5a344795a7

user@x86_64 $ docker run --rm --privileged docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64
Unable to find image 'docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64' locally
a7996909642ee92942dcd6cff44b9b95f08dad64: Pulling from docker/binfmt
5d6ca6c8ba77: Pull complete
b26a8e2c75fc: Pull complete
3436361ddd98: Pull complete
Digest: sha256:758ca0563f371b384cfd67b6590b5be2dc024fef45bc14a050ae104f0caad14e
Status: Downloaded newer image for docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64

user@x86_64 $ docker buildx create --use --name builderx
builderx

user@x86_64 $ docker buildx ls
NAME/NODE   DRIVER/ENDPOINT             STATUS   PLATFORMS
builderx *  docker-container
  builderx0 unix:///var/run/docker.sock inactive
default     docker
  default   default                     running  linux/amd64, linux/386
```

### load generates an error
```
user@x86_64 $ docker buildx build -t ccarmelo/hellogo:1.1 --platform linux/amd64,linux/386,linux/arm/v7,linux/arm/v6 --load .
...
 => ERROR exporting to oci image format	0.0s
------
 > exporting to oci image format:
------
failed to solve: rpc error: code = Unknown desc = docker exporter does not currently support exporting manifest lists
```


### push works
```
user@x86_64 $ docker buildx build -t ccarmelo/hellogo:1.1 --platform linux/amd64,linux/386,linux/arm/v7,linux/arm/v6 --push .
[+] Building 14.1s (48/48) FINISHED
 => [internal] load build definition from Dockerfile
...
 => => pushing layers	7.7s
 => => pushing manifest for docker.io/ccarmelo/hellogo:1.1
```

Problem is we need to pull the image back from Docker Hub.
```
user@x86_64 $ docker pull ccarmelo/hellogo:1.1
1.1: Pulling from ccarmelo/hellogo
c9b1b535fdd9: Already exists
90cc5bd1f8b7: Pull complete
2860aaf1e041: Pull complete
a4691a108fa3: Pull complete
Digest: sha256:7adc59c558db656cc8bb5671877046e78bae2d7af70234f2e728290fcf18ebd6
Status: Downloaded newer image for ccarmelo/hellogo:1.1
docker.io/ccarmelo/hellogo:1.1
```

It's interesting to notice how the image _embeds_ several different flavours
```
user@x86_64 $ docker buildx imagetools inspect ccarmelo/hellogo:1.1
Name:      docker.io/ccarmelo/hellogo:1.1
MediaType: application/vnd.docker.distribution.manifest.list.v2+json
Digest:    sha256:7adc59c558db656cc8bb5671877046e78bae2d7af70234f2e728290fcf18ebd6

Manifests:
  Name:      docker.io/ccarmelo/hellogo:1.1@sha256:5924e16e74ed3751e7c698ab3f9285f9d69b9aa6e9187d05d60deb23d4c83643
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/amd64

  Name:      docker.io/ccarmelo/hellogo:1.1@sha256:b4a44b430b5ca972c99749ce5e9beb8dbd9b54834cf44f75e115a184bfa0e8a8
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/386

  Name:      docker.io/ccarmelo/hellogo:1.1@sha256:c98687020268c600cd646a24ab3572125bbf8b27e73b9730c0048356943316c0
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm/v7

  Name:      docker.io/ccarmelo/hellogo:1.1@sha256:2d71794ce89d3907ce2028965fbca3db9937ef07d1f451b3c23c461d9a806b7c
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm/v6
```


### on ARM after buildx
We're good to go now. Same image we'd run on AMD/Intel/x86_64...
```
user@arm $ docker pull ccarmelo/hellogo:1.1
1.1: Pulling from ccarmelo/hellogo
3a2c5e3c37b2: Pull complete
1737a74a0cfb: Pull complete
1ee496fbaa96: Pull complete
41d5537292f3: Pull complete
Digest: sha256:7adc59c558db656cc8bb5671877046e78bae2d7af70234f2e728290fcf18ebd6
Status: Downloaded newer image for ccarmelo/hellogo:1.1
docker.io/ccarmelo/hellogo:1.1


user@arm $ docker run ccarmelo/hellogo:1.1
Hello, world!
```
