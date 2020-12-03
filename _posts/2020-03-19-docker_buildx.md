---
layout: post
title: "Buildx: building multi-arch images"
categories: misc
---

In this post I'll explore how to build images that can run on multiple architectures. My scenario is composed of the two following computers:
- x86_64
  ```
  $ docker info | grep Architecture
   Architecture: x86_64

  $ cat /proc/cpuinfo | grep "model name" | uniq
  model name	: Intel(R) Core(TM) i5-3317U CPU @ 1.70GHz
  ```
- ARM
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
user@x86_64 $ docker build -t carmelo0x99/hellogo:x86_64 .

user@x86_64 $ docker run carmelo0x99/hellogo:x86_64
Hello, world!
```

### Now, running the same image on ARM leads to...
```
user@arm $ docker run carmelo0x99/hellogo:x86_64
standard_init_linux.go:211: exec user process caused "exec format error"
```
Unsurprisingly:
```
user@armi $ docker inspect carmelo0x99/hellogo:x86_64 | grep Architecture
        "Architecture": "amd64",
```

### Enters buildx
```
user@x86_64 $ export DOCKER_CLI_EXPERIMENTAL=enabled

user@x86_64 $ docker buildx version
github.com/docker/buildx v0.4.2-tp-docker fb7b670b764764dc4716df3eba07ffdae4cc47b2

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

### `load` generates an error
```
user@x86_64 $ docker buildx build -t carmelo0x99/hellogo:2.0 --platform linux/amd64,linux/386,linux/arm/v7,linux/arm/v6 --load .
...
 => ERROR exporting to oci image format	0.0s
------
 > exporting to oci image format:
------
failed to solve: rpc error: code = Unknown desc = docker exporter does not currently support exporting manifest lists
```

### `push` works instead
```
user@x86_64 $ docker buildx build -t carmelo0x99/hellogo:2.0 --platform linux/amd64,linux/386,linux/arm/v7,linux/arm/v6 --push .
[+] Building 14.1s (48/48) FINISHED
 => [internal] load build definition from Dockerfile
...
 => => pushing layers	7.7s
 => => pushing manifest for docker.io/ccarmelo/hellogo:1.1
```

Problem is we need to pull the image back from Docker Hub.
```
user@x86_64 $ docker pull carmelo0x99/hellogo:2.0
2.0: Pulling from carmelo0x99/hellogo
188c0c94c7c5: Already exists
7a30352f7640: Pull complete
7eb7155945d7: Pull complete
cc4e0f765d0b: Pull complete
Digest: sha256:ee0ccf88180d071eb242814233a32aae78ca1c5b1281742fff80113aed5cf864
Status: Downloaded newer image for carmelo0x99/hellogo:2.0
docker.io/carmelo0x99/hellogo:2.0
```

It's interesting to notice how the image _embeds_ several different flavours
```
user@x86_64 $ docker buildx imagetools inspect carmelo0x99/hellogo:2.0
Name:      docker.io/carmelo0x99/hellogo:2.0
MediaType: application/vnd.docker.distribution.manifest.list.v2+json
Digest:    sha256:ee0ccf88180d071eb242814233a32aae78ca1c5b1281742fff80113aed5cf864

Manifests:
  Name:      docker.io/carmelo0x99/hellogo:2.0@sha256:baf37cc593cf0818a7afeb1059b7b7d9324232a0f31e17aaeb80b2c73f2a56c5
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/amd64

  Name:      docker.io/carmelo0x99/hellogo:2.0@sha256:b8f62e3da564e65a67b737c6230b22f2bff7279f3f89c5e7677466f9b8f4ebb6
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/386

  Name:      docker.io/carmelo0x99/hellogo:2.0@sha256:784480a45d79d3812608b9dd626909a863f72f307cf16115d9ee94c95db9da5e
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm/v7

  Name:      docker.io/carmelo0x99/hellogo:2.0@sha256:74d0ee688f789968ae967ddd3aaa3aee71102fc5ed0d5b1315cc4ece56c5b9ef
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm/v6
```


### on ARM after buildx
We're good to go now. Same image we'd run on AMD/Intel/x86_64 runs **seamlessly** on ARM.
```
user@arm $ docker pull carmelo0x99/hellogo:2.0
2.0: Pulling from carmelo0x99/hellogo
5f2023fd85a4: Pull complete
807a035978f6: Pull complete
5ca0ae22a091: Pull complete
8ca028498186: Pull complete
Digest: sha256:ee0ccf88180d071eb242814233a32aae78ca1c5b1281742fff80113aed5cf864
Status: Downloaded newer image for carmelo0x99/hellogo:2.0
docker.io/carmelo0x99/hellogo:2.0

user@arm $ docker run carmelo0x99/hellogo:2.0
Hello, world!
```
