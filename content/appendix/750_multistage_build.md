---
title: "Multistage Build"
chapter: false
weight: 750
draft: false
---


## Alternative Container Build Tooling

You do not have to use `docker build` to build all your container images. There are alternatives.

In [Docker Images Without Docker — A Practical Guide](https://codefresh.io/devops/docker-images-without-docker-practical-guide/), posted on Dec 02, 2020, Anais Urlichs describes alternatives to building docker and OCI images for your containers.
- Buildah - you can use `buildah` instead of `docker build` to create container images.
- Podman - you can use `podman` to list, use, and manage images throughout their lifecycle.
- Kaniko - you can use `kaniko` to build images within a Kubernetes cluster.
In [Building containers without Docker](https://blog.alexellis.io/building-containers-without-docker/) posted 25 JANUARY 2020, Alix Ellis describes some other alternatives for building container images, including the Buildah and Podman combination.
- Buildkit - is a toolset you can use with docker, standalone with buildctl, or with a subset of buildctl such as img. It features a buildkit daemon `buildkitd`.
- Buildctl - `buildctl` is a `docker build` CLI alternative which uses `buildkitd`.
- Img - `img` is a subset of `buildctl` with fewer options and capabilities.
- Kaniko again - like Anais Urlichs' article, Alix Ellis suggests Kaniko, and describes it that it "aims to sandbox container builds." That is an apt summary.

However, as use of `docker build` is still quite common and since we also started this workshop with that technique, let's look at a few ways you can be more effective with `docker build`.

## Cut Down the Bloat

The Docker documentation describes how to [Use multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/). The primary goals of multi-stage builds are efficiency and security. Many blogs and courses suggest using multi-stage builds as a best practice. Let's take a look.

Not all apps are written as shell scripts and single-page PHP apps. 
We just used those techniques as quick and simple demonstrations. 
Multi-stage builds can be useful when:
- you need a build environment to transform, substitute, filter, compile, or link your code into an executable form
- you have a base image which has extra components your app doesn't need
- you have tests you want to do within the build process

Like multi-stage rockets, the lower stages can be jettisoned to reduce the mass and volume of the resultant final stage. 
Your payload is in that final stage. 
Your app code is all that remains.

In short, multi-stage builds can cut down the bloat.
You know you want containers that are:
- secure
- durable
- efficient
- right-sized
- cost-optimized
- performant, and
- sustainable.
Prune out the dead growth.

## Start with a Single Stage

You can use base images such as `Ubuntu`, `Debian`, `Alpine` Linux distributions or many others.
You can choose base images which have not just an operating system installed, but app frameworks too, such as Node.JS, Python, PHP, MySQL, etc. You can even decide to deploy container images which already have all the right components installed except your already-runnable app code.

However, this is not always the case.
Sometimes you have to build your app. Using a compiler or similar tooling.
Perhaps you're using Go, Rust, C++, C, or other languages and tools that compile and link your app into a binary. 
Maybe you're using shared libraries in the execution environment or opting for a static single binary. 
You could choose languages with compilation into an intermediate language such as Java, Kotlin, C#, F#, or VB.net. 
You may have more build side effects and side products than the artifacts you need.

Let's look at an example of a simple app with two programs written in C that are based on two classic UNIX commands--`cat` and `kill`. These two little apps are called:
- `kc` -- short for `kittycat`, which is based on the classic `cat`.
- `cn` -- short for `catnip`, which is based on the classic `kill` and also on `cp` or `cat`. But it also interprets a little bit of HTTP so can serve as a crude web server for demonstration or test purposes.
These two can be used independently, together, or in conjunction with `nc` (netcat).

{{< step >}}Download the source code for the `kittycat` project.{{< /step >}}
```bash
git clone https://github.com/bwer432/kittycat
```

Example output:
{{< output >}}
Cloning into 'kittycat'...
remote: Enumerating objects: 184, done.
remote: Counting objects: 100% (184/184), done.
remote: Compressing objects: 100% (109/109), done.
remote: Total 184 (delta 103), reused 137 (delta 59), pack-reused 0
Receiving objects: 100% (184/184), 136.97 KiB | 2.21 MiB/s, done.
Resolving deltas: 100% (103/103), done.
{{< /output >}}

{{% notice note %}}
This includes both `kittycat` and `catnip`.
{{% /notice %}}

{{< step >}}Go in and take a look.{{< /step >}}
```bash
cd kittycat
ls
```

Example output:
{{< output >}}
body               example-drawing.html  quickscan.c
cat.c              index2.html           README.md
catnip.c           index.html            req1.http
catnip.h           kc                    req2.http
catnip-kitty.awk   kill.c                req3.http
catweb             kitty                 req4.http
catweb1            kittycat.c            req-head.http
cn                 kittycat.html         req-trace-hello.http
Dockerfile         LICENSE               req-trace.http
Dockerfile.1stage  Makefile              response.http
Dockerfile.bak     quickscan             TODO
{{< /output >}}

{{< step >}}Build those for your host system (e.g. Amazon Linux).{{< /step >}}
```bash
make
```

{{< output >}}
cc -c kittycat.c
cc -o kc kittycat.o
cc -c catnip.c
cc -o cn catnip.o
{{< /output >}}

{{< step >}}Check how big these executables are.{{< /step >}}
```bash
$ ls -1s kc cn                                                      
```

Example output:
{{< output >}}
24 cn
20 kc
{{< /output >}}

{{% notice note %}}
These may be about 24 KiB and 20 KiB respectively.
They are pretty small, simple commands.
They can be used as app components.
{{% /notice %}}


{{< step >}}Look at the 1-stage Dockerfile.{{< /step >}}
```bash
cat Dockerfile.1stage
```

Example output:

{{< output >}}
FROM alpine
RUN apk update && apk add curl && apk add build-base
RUN mkdir /src
WORKDIR /src
COPY ./kittycat.c ./catnip.c ./catnip.h ./
RUN cc -c kittycat.c \
    && cc -c catnip.c \
    && cc -static -o kc kittycat.o \
    && cc -static -o cn catnip.o
RUN cp cn /usr/bin/
RUN cp kc /usr/bin/
WORKDIR /var/local/kitty
COPY kitty/* ./
CMD ["sh","-c","kc -w 86400 response.http body | nc -l 80 | cn -w /var/local/kitty"]
EXPOSE 80
{{< /output >}}

{{< step >}}Now build those apps within Alpine Linux for your container image.{{< /step >}}
```bash
docker build -t bwer432/kittycat:thick -f ./Dockerfile.1stage .
```

Example output:
{{< output >}}
Sending build context to Docker daemon  393.7kB
Step 1/12 : FROM alpine
latest: Pulling from library/alpine
59bf1c3509f3: Pull complete 
Digest: sha256:21a3deaa0d32a8057914f36584b5288d2e5ecc984380bc0118285c70fa8c9300
Status: Downloaded newer image for alpine:latest
 ---> c059bfaa849c
Step 2/12 : RUN apk update && apk add curl && apk add build-base
 ---> Running in 8fdbb711c5d9
fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
v3.15.0-289-g3306c05324 [https://dl-cdn.alpinelinux.org/alpine/v3.15/main]
v3.15.0-295-gdfd3f5c6f5 [https://dl-cdn.alpinelinux.org/alpine/v3.15/community]
OK: 15858 distinct packages available
(1/5) Installing ca-certificates (20211220-r0)
(2/5) Installing brotli-libs (1.0.9-r5)
(3/5) Installing nghttp2-libs (1.46.0-r0)
(4/5) Installing libcurl (7.80.0-r0)
(5/5) Installing curl (7.80.0-r0)
Executing busybox-1.34.1-r3.trigger
Executing ca-certificates-20211220-r0.trigger
OK: 8 MiB in 19 packages
(1/20) Installing libgcc (10.3.1_git20211027-r0)
(2/20) Installing libstdc++ (10.3.1_git20211027-r0)
(3/20) Installing binutils (2.37-r3)
(4/20) Installing libmagic (5.41-r0)
(5/20) Installing file (5.41-r0)
(6/20) Installing libgomp (10.3.1_git20211027-r0)
(7/20) Installing libatomic (10.3.1_git20211027-r0)
(8/20) Installing libgphobos (10.3.1_git20211027-r0)
(9/20) Installing gmp (6.2.1-r1)
(10/20) Installing isl22 (0.22-r0)
(11/20) Installing mpfr4 (4.1.0-r0)
(12/20) Installing mpc1 (1.2.1-r0)
(13/20) Installing gcc (10.3.1_git20211027-r0)
(14/20) Installing musl-dev (1.2.2-r7)
(15/20) Installing libc-dev (0.7.2-r3)
(16/20) Installing g++ (10.3.1_git20211027-r0)
(17/20) Installing make (4.3-r0)
(18/20) Installing fortify-headers (1.1-r1)
(19/20) Installing patch (2.7.6-r7)
(20/20) Installing build-base (0.5-r2)
Executing busybox-1.34.1-r3.trigger
OK: 198 MiB in 39 packages
Removing intermediate container 8fdbb711c5d9
 ---> d7e4afdd1309
Step 3/12 : RUN mkdir /src
 ---> Running in 59d3b6475cf1
Removing intermediate container 59d3b6475cf1
 ---> ecd60f6489ac
Step 4/12 : WORKDIR /src
 ---> Running in c75194a4bda7
Removing intermediate container c75194a4bda7
 ---> 0b8b1b25dc90
Step 5/12 : COPY ./kittycat.c ./catnip.c ./catnip.h ./
 ---> 66c47629807d
Step 6/12 : RUN cc -c kittycat.c     && cc -c catnip.c     && cc -static -o kc kittycat.o     && cc -static -o cn catnip.o
 ---> Running in 074a0415de35
Removing intermediate container 074a0415de35
 ---> 3353876d7cb1
Step 7/12 : RUN cp cn /usr/bin/
 ---> Running in 1dae32d193ae
Removing intermediate container 1dae32d193ae
 ---> c4ba00221ce9
Step 8/12 : RUN cp kc /usr/bin/
 ---> Running in a51e8155e621
Removing intermediate container a51e8155e621
 ---> 3da5189651db
Step 9/12 : WORKDIR /var/local/kitty
 ---> Running in 78589398a399
Removing intermediate container 78589398a399
 ---> 4dc1f6f2df30
Step 10/12 : COPY kitty/* ./
 ---> 519af3e343ec
Step 11/12 : CMD ["sh","-c","kc -w 86400 response.http body | nc -l 8000 | cn -w /var/local/kitty"]
 ---> Running in 6c78d16c7b37
Removing intermediate container 6c78d16c7b37
 ---> 7b9eb2645d9d
Step 12/12 : EXPOSE 8000
 ---> Running in 4f56429f17d6
Removing intermediate container 4f56429f17d6
 ---> de9b3b4d017c
Successfully built de9b3b4d017c
Successfully tagged bwer432/kittycat:thick
{{< /output >}}

{{< step >}}List out the image you created.{{< /step >}}
```bash
docker images bwer432/kittycat
```

{{< output >}}
REPOSITORY         TAG       IMAGE ID       CREATED         SIZE
bwer432/kittycat   thick     de9b3b4d017c   4 minutes ago   207MB
{{< /output >}}

You have created a container image which includes the build artifacts.
But it also contains the build environment tools.

## Add a Separate Stage

After you have built an image, you could prune it down to a smaller footprint.
Again, your motivations for such trimming could be security, cost, resiliency, or performance efficiency.

{{< step >}}Look at a Dockerfile that takes your thick image and prunes it.{{< /step >}}
```bash
cat Dockerfile.prune 
```

{{< output >}}
FROM bwer432/kittycat:thick AS base-image

FROM alpine
RUN apk update && apk add curl
COPY --from=base-image /usr/bin/cn /usr/bin
COPY --from=base-image /usr/bin/kc /usr/bin
WORKDIR /var/local/kitty
COPY kitty/* ./
CMD ["sh","-c","kc -w 86400 response.http body | nc -v -l -p 80 | cn -w /var/local/kitty"]
EXPOSE 80
{{< /output >}}

{{< step >}}Now build your ***thin*** image from the first one.{{< /step >}}
```bash
docker build -t bwer432/kittycat:thin -f ./Dockerfile.prune .     
```

Example output:
{{< output >}}
Sending build context to Docker daemon  394.8kB
Step 1/9 : FROM bwer432/kittycat:thick AS base-image
 ---> de9b3b4d017c
Step 2/9 : FROM alpine
 ---> c059bfaa849c
Step 3/9 : RUN apk update && apk add curl
 ---> Running in a952c888bd93
fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
v3.15.0-289-g3306c05324 [https://dl-cdn.alpinelinux.org/alpine/v3.15/main]
v3.15.0-295-gdfd3f5c6f5 [https://dl-cdn.alpinelinux.org/alpine/v3.15/community]
OK: 15858 distinct packages available
(1/5) Installing ca-certificates (20211220-r0)
(2/5) Installing brotli-libs (1.0.9-r5)
(3/5) Installing nghttp2-libs (1.46.0-r0)
(4/5) Installing libcurl (7.80.0-r0)
(5/5) Installing curl (7.80.0-r0)
Executing busybox-1.34.1-r3.trigger
Executing ca-certificates-20211220-r0.trigger
OK: 8 MiB in 19 packages
Removing intermediate container a952c888bd93
 ---> e1b53502b443
Step 4/9 : COPY --from=base-image /usr/bin/cn /usr/bin
 ---> 4663b48bef8a
Step 5/9 : COPY --from=base-image /usr/bin/kc /usr/bin
 ---> d2eddb772597
Step 6/9 : WORKDIR /var/local/kitty
 ---> Running in 1f0a1bd0d5d2
Removing intermediate container 1f0a1bd0d5d2
 ---> 6d024eb55561
Step 7/9 : COPY kitty/* ./
 ---> 52a1fd524cd0
Step 8/9 : CMD ["sh","-c","kc -w 86400 response.http body | nc -v -l -p 80 | cn -w /var/local/kitty"]
 ---> Running in d63dd95e7654
Removing intermediate container d63dd95e7654
 ---> 8a7cba5770c4
Step 9/9 : EXPOSE 80
 ---> Running in 94730e0c6c4d
Removing intermediate container 94730e0c6c4d
 ---> 8960b1c0e9fb
Successfully built 8960b1c0e9fb
Successfully tagged bwer432/kittycat:thin
{{< /output >}}

{{< step >}}Check the size difference between your ***thick*** and ***thin*** images.{{< /step >}}
```bash
docker images bwer432/kittycat                                 
```

Example output:
{{< output >}}
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
bwer432/kittycat   thin      8960b1c0e9fb   14 seconds ago   10.9MB
bwer432/kittycat   thick     de9b3b4d017c   9 minutes ago    207MB
{{< /output >}}

How do the sizes of the images compare?
Why?

{{< step >}}Check how big the `alpine` image is.{{< /step >}}
```bash
docker images alpine
```

{{< output >}}
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
alpine       latest    c059bfaa849c   2 months ago   5.59MB
{{< /output >}}

How big in your thin image compared with the `alpine` base image?
Why?

## Make it Multi-Stage

Do you need to save the thick image?
Unless it serves some testing need, jettison it. 
You don't need to tag it. 
You don't need to keep it.

You can choose to perform a multi-stage build within one Dockerfile.

{{< step >}}Inspect the default `Dockerfile` included in the `kittycat` project.{{< /step >}}
```bash
FROM alpine AS build-stage
RUN apk update && apk add curl && apk add build-base
RUN mkdir /src
WORKDIR /src
COPY ./kittycat.c ./catnip.c ./catnip.h ./
RUN cc -c kittycat.c \
    && cc -c catnip.c \
    && cc -static -o kc kittycat.o \
    && cc -static -o cn catnip.o
RUN cp cn /usr/bin/
RUN cp kc /usr/bin/

FROM alpine
RUN apk update && apk add curl
COPY --from=build-stage /usr/bin/cn /usr/bin
COPY --from=build-stage /usr/bin/kc /usr/bin
WORKDIR /var/local/kitty
COPY kitty/* ./
CMD ["sh","-c","kc -w 86400 response.http body | nc -l 80 | cn -w /var/local/kitty"]
EXPOSE 80
```

{{< step >}}Inspect the part of the project's `Makefile` which builds an image.{{< /step >}}
```bash
grep ^image: -A 6 Makefile                                        
```

Example output:
{{< output >}}
image:
        @echo "+ $@"
        @docker build -t ${HUB_NAMESPACE}/${IMAGE_NAME}:${VERSION} -f ./${DOCKERFILE} .
        @docker tag ${HUB_NAMESPACE}/${IMAGE_NAME}:${VERSION} ${HUB_NAMESPACE}/${IMAGE_NAME}:latest
        @echo "Done."
        @docker images --format '{{.Repository}}:{{.Tag}}\t\t Built: {{.CreatedSince}}\t\tSize: {{.Size}}' | grep ${IMAGE_NAME}:${VERSION}
{{< /output >}}

{{< step >}}
Build the image using that default Dockerfile.
{{< /step >}}
```bash
make image
```

{{< output >}}
make image
+ image
Sending build context to Docker daemon  394.8kB
Step 1/16 : FROM alpine AS build-stage
 ---> c059bfaa849c
Step 2/16 : RUN apk update && apk add curl && apk add build-base
 ---> Using cache
 ---> d7e4afdd1309
Step 3/16 : RUN mkdir /src
 ---> Using cache
 ---> ecd60f6489ac
Step 4/16 : WORKDIR /src
 ---> Using cache
 ---> 0b8b1b25dc90
Step 5/16 : COPY ./kittycat.c ./catnip.c ./catnip.h ./
 ---> Using cache
 ---> 66c47629807d
Step 6/16 : RUN cc -c kittycat.c     && cc -c catnip.c     && cc -static -o kc kittycat.o     && cc -static -o cn catnip.o
 ---> Using cache
 ---> 3353876d7cb1
Step 7/16 : RUN cp cn /usr/bin/
 ---> Using cache
 ---> c4ba00221ce9
Step 8/16 : RUN cp kc /usr/bin/
 ---> Using cache
 ---> 3da5189651db
Step 9/16 : FROM alpine
 ---> c059bfaa849c
Step 10/16 : RUN apk update && apk add curl
 ---> Using cache
 ---> e1b53502b443
Step 11/16 : COPY --from=build-stage /usr/bin/cn /usr/bin
 ---> Using cache
 ---> 4663b48bef8a
Step 12/16 : COPY --from=build-stage /usr/bin/kc /usr/bin
 ---> Using cache
 ---> d2eddb772597
Step 13/16 : WORKDIR /var/local/kitty
 ---> Using cache
 ---> 6d024eb55561
Step 14/16 : COPY kitty/* ./
 ---> Using cache
 ---> 52a1fd524cd0
Step 15/16 : CMD ["sh","-c","kc -w 86400 response.http body | nc -v -l -p 80 | cn -w /var/local/kitty"]
 ---> Using cache
 ---> 8a7cba5770c4
Step 16/16 : EXPOSE 80
 ---> Using cache
 ---> 8960b1c0e9fb
Successfully built 8960b1c0e9fb
Successfully tagged bwer432/kittycat:0.1.6
Done.
bwer432/kittycat:0.1.6           Built: 16 minutes ago          Size: 10.9MB
{{< /output >}}

{{< step >}}Compare the results of this build with the separate `thick` and `thin` images.{{< /step >}}
```bash
docker images bwer432/kittycat
```

Example output:
{{< output >}}
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
bwer432/kittycat   0.1.6     8960b1c0e9fb   16 minutes ago   10.9MB
bwer432/kittycat   latest    8960b1c0e9fb   16 minutes ago   10.9MB
bwer432/kittycat   thin      8960b1c0e9fb   16 minutes ago   10.9MB
bwer432/kittycat   thick     de9b3b4d017c   25 minutes ago   207MB
{{< /output >}}

You have:
- built a container image using two stages in one `Dockerfile`.
- tagged that image with a version number such as `0.1.6`.
- also tagged that image as `latest`.

Thus, when you and others consume this image, you can choose the `latest` or a specific version.

## Create Custom Tailored Versions

TODO

## Success

TODO
