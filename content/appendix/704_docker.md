---
title: "Docker and Kubernetes"
chapter: false
weight: 704
draft: false
---

## The Many Faces of Docker

Docker is *not* just one thing. Is it a daemon? Is the daemon also called an engine? A command? Can you script the command? A container image file format? An image *build* instruction file format? The build tool which uses that? A composition of multiple container specification format? The compose tool that uses that? An Internet based hub for both public and private registries and repositories of container images? Yes. Yes to all of those questions and more. Docker is everything. Well, not everything. But there is a lot to Docker. 

The [Docker reference documentation](https://docs.docker.com/reference/) is divided into sections covering the Docker platform's:
- two file formats: [`Dockerfile`](https://docs.docker.com/engine/reference/builder/) and [`Compose` file](https://docs.docker.com/compose/compose-file/).
- three command line interfaces: [Docker CLI](https://docs.docker.com/engine/reference/commandline/cli/), [Compose CLI](https://docs.docker.com/compose/reference/), and the [Docker Daemon (dockerd) CLI](https://docs.docker.com/engine/reference/commandline/dockerd/).
- three application programming interfaces: [Engine API](https://docs.docker.com/engine/api/), [Registry API](https://docs.docker.com/registry/spec/api/), and [Docker Hub API](https://docs.docker.com/docker-hub/api/latest/).
- a [Docker Image specification](https://docs.docker.com/registry/spec/manifest-v2-2/).
- a [Registry token authentication scheme](https://docs.docker.com/registry/spec/auth/).
- and [Registry storage drivers](https://docs.docker.com/registry/storage-drivers/).

Here are some relationships between a few of these Docker ingredients:
{{< mermaid >}}
graph
docker[docker<br>client<br>CLI]
dockerd[dockerd<br>docker daemon<br>runtime engine]
dockerfile([Dockerfile<br>build<br>instructions])
dockerbuild[docker build<br>command]
dockerimage[(docker/container/OCI<br>image)]
container((container<br>running<br>image))
docker -->|used for| dockerbuild
dockerbuild -->|uses| dockerfile
dockerbuild -->|produces| dockerimage
docker -->|references| dockerimage
docker -->|manages| dockerd
dockerd -->|uses| dockerimage
dockerd -->|runs| container
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class docker,dockerbuild green;
class dockerfile lavender;
class dockerd orange;
class dockerimage cyan;
class container blue2;
{{< /mermaid >}}

## Which Docker Components are Relevant for Kubernetes?

Traditionally, prior to Kubernetes version 1.20, `dockerd`--the docker ***daemon*** or ***engine***--had been used as a de facto container runtime in Kubernetes environments. However, when people heard news such as "Kubernetes no longer supports Docker," there was much confusion and panic. Here are a few tips:

1. Docker is still useful in its own right.
2. Most, perhaps all, of the other components of the docker ecosystem is still usable with, around, and before Kubernetes gets involved with your containers.
3. Yes, that includes the `Dockerfile`--still relevant.
4. Yes, that includes the `docker build` command--still relevant.
5. Yes, that includes your `docker image` files. 
6. Docker Hub and other image registries for your image repositories are still very much alive.

Several authors in the Kubernetes community, Jorge Castro and several others, published a blog article in late 2020 titled: [Don't Panic: Kubernetes and Docker](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/). The first two statements they made are: "Kubernetes is [deprecating Docker](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation) as a container runtime after v1.20," and "You do not need to panic. It's not as dramatic as it sounds" (Castro, et al., 2020).

Those authors included a TL;DR note: "Docker as an underlying runtime is being deprecated in favor of runtimes that use the [Container Runtime Interface (CRI)](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/) created for Kubernetes. Docker-produced images will continue to work in your cluster with all runtimes, as they always have" (Castro, et al., 2020).

Before we get into the relevance of this to the story of Docker and Kubernetes, we should visit what has happened with Docker over time.

## Divesting Docker: Monolithic `dockerd` to modular `containerd` and `runc`

Once upon a time, the docker daemon `dockerd` was monolithic. Yes, the container runtime engine was a single executable.

Then, `dockerd` was refactored into:
- the "upper" container runtime: `containerd` (or ***containerD*** with a capital D)
{{% expand "Show more" %}} - the **daemon** that replaces the protocol layer of `dockerd` for compatibility with the `docker` client CLI. `containerd` pulls images and maintains storage and networking integrations.{{% /expand %}}
- the "lower" container runtime: `runc` (or ***runC*** with a capital C) 
{{% expand "Show more" %}} - the **container runtime** component. This is the component which works with **namespaces** and **cgroups**. The `unshare` and `nsenter` functionality you used in the [Linux Namespaces section]({{< ref "014_linux_namespaces" >}}) is provided for you by `runc`.{{% /expand %}}

{{% notice note %}}
Technically, when some people say "docker," as in "docker is deprecated," what they really mean is use of "dockerd" is deprecated. However, as far as current releases, `dockerd` has not existed as the monolith it once was.
{{% /notice %}}

Revisiting the earlier Docker architecture with this split from the old `dockerd` into the more modern `containerd` and `runc`, we can depict as shown in this figure.

{{< mermaid >}}
graph
docker[docker<br>client<br>CLI]
containerd[containerd<br>docker daemon<br>runtime engine]
dockerfile([Dockerfile<br>build<br>instructions])
dockerbuild[docker build<br>command]
dockerimage[(docker/container/OCI<br>image)]
runC[runC<br>container<br>runtime]
container((container<br>running<br>image))
docker -->|used for| dockerbuild
dockerbuild -->|uses| dockerfile
dockerbuild -->|produces| dockerimage
docker -->|references| dockerimage
docker -->|manages| containerd
containerd -->|uses| dockerimage
containerd -->|drives| runC
runC -->|runs| container
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class docker,dockerbuild green;
class dockerfile lavender;
class containerd orange;
class dockerimage,runC cyan;
class container blue2;
{{< /mermaid >}}


## Open Container Initiative (OCI)

What should you call an image? Docker image? Container image? OCI image?

In 2015, Docker announced a collaboration with other companies and formed the [Open Container Initiative (OCI)](https://opencontainers.org). They announced "Docker is donating its container format and runtime, runC, to the OCI to serve as the cornerstone of this new effort. It is available now at [https://github.com/opencontainers/runc](https://github.com/opencontainers/runc)." These two components, a "Docker image" and `runc` are now managed by the OCI.

[About OCI](https://opencontainers.org/about/overview/) summarizes as follows: "Established in June 2015 by Docker and other leaders in the container industry, the OCI currently contains two specifications: the Runtime Specification (runtime-spec) and the Image Specification (image-spec). The Runtime Specification outlines how to run a “filesystem bundle” that is unpacked on disk. At a high-level an OCI implementation would download an OCI Image then unpack that image into an OCI Runtime filesystem bundle. At this point the OCI Runtime Bundle would be run by an OCI Runtime."

Therefore, you can treat these three phrases as compatible synonyms:
- Docker image
- container image
- OCI image

Because both the `image` format and `runc` are now managed by the OCI, you can amend the previous diagram.

{{< mermaid >}}
graph
docker[docker<br>client<br>CLI]
containerd[containerd<br>docker daemon<br>runtime engine]
dockerfile([Dockerfile<br>build<br>instructions])
dockerbuild[docker build<br>command]
subgraph oci[" "]
  dockerimage[(docker/container/OCI<br>image)]
  runC[runC<br>container<br>runtime]
  oci-note{{Docker originated<br>Now managed by OCI}}
end
container((container<br>running<br>image))
docker -->|used for| dockerbuild
dockerbuild -->|uses| dockerfile
dockerbuild -->|produces| dockerimage
docker -->|references| dockerimage
docker -->|manages| containerd
containerd -->|uses<br>OCI image-spec| dockerimage
containerd -->|drives<br>OCI runtime-spec| runC
runC -->|runs| container
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class docker,dockerbuild green;
class dockerfile lavender;
class containerd orange;
class dockerimage,runC cyan;
class container blue2;
{{< /mermaid >}}

Some important offshoots of this evolution are:
- `dockerd` is really now `containerd` and `runc`
- Docker "image" format is now the OCI image-spec
- Docker `runc` is now the OCI runtime-spec

## Kubernetes: Container Runtime Interface (CRI)

In 2016, Kubernetes standardized the interface to the upper container runtime by [Introducing Container Runtime Interface (CRI) in Kubernetes](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/). The [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/) allows you to have a choice of [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/). 

The [CRI](https://github.com/kubernetes/kubernetes/blob/242a97307b34076d5d8f5bbeb154fa4d97c9ef1d/docs/devel/container-runtime-interface.md) offers pluggable choice of container runtimes, including but not limited to:
- [containerd](https://containerd.io/) - a general purpose container runtime
- [cri-o](https://cri-o.io/) - a Kubernetes native container runtime

The CRI control program--[`crictl`](https://github.com/kubernetes-sigs/cri-tools)--allows you to [debug Kubernetes nodes](https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/) with commands such as:
- `crictl pods`
- `crictl images`
- `crictl ps`
- `crictl exec`
- `crictl logs`

Note that *both* the upper-layer `containerd` and `cri-o` container runtimes use the OCI **runtime-spec** to actually create, delete, and manage the containers in the pods. Most commonly, `containerd` and `cri-o` use:
- [`runc`](https://github.com/opencontainers/runc) as the lower-layer container runtime.

Each Kubernetes node must run the Kubernetes node agent, called the `kubelet`. This `kubelet` interacts with the rest of the Kubernetes cluster in a centralized-distributed think globally, act locally fashion. It is the `kubelet` which uses the CRI to access your chosen container runtime. In other words, a node's `kubelet` interfaces with either `containerd`, `cri-o`, or some other container runtime you have chosen for that node.

In the shipping metaphor, each node is a ship, the `kubelet` is the captain of the ship, and the container runtime is the captain's second-in-command. In Kubernetes, the `kubelet` is responsible for all operations on the node, and the container runtime (e.g. `containerd` or `cri-o` and ultimiately `runc` or similar) is responsible for carrying out their orders. This diagram illustrates the relationship between these components in each Kubernetes node. 

Note that `docker build` and the `Dockerfile` are shown only as a reminder that in Kubernetes you use container images like in Docker. Your devops pipelines would neither necessarily nor customarily run `docker build` nor store `Dockerfile` manifests directly on a Kubernetes node.

{{< mermaid >}}
graph
kubelet[kubelet<br>Kubernetes<br>Node Agent]
subgraph cri[" "]
  containerd[containerd<br>General purpose<br>runtime engine]
  crio[CRI-O<br>Kubernetes native<br>runtime engine]
  cri-note{{Kubernetes<br>standardized CRI}}
end
dockerfile([Dockerfile<br>build<br>instructions])
dockerbuild[docker build<br>command]
subgraph oci[" "]
  dockerimage[(docker/container/OCI<br>image)]
  runC[runC<br>container<br>runtime]
  oci-note{{Docker originated<br>Now managed by OCI}}
end
container((container<br>running<br>image))
dockerbuild -->|uses| dockerfile
dockerbuild -->|produces| dockerimage
kubelet -->|manages<br>via CRI| containerd
kubelet -->|manages<br>via CRI| crio
containerd -->|uses<br>OCI image-spec| dockerimage
containerd -->|drives<br>OCI runtime-spec| runC
crio -->|uses<br>OCI image-spec| dockerimage
crio -->|drives<br>OCI runtime-spec| runC
runC -->|runs| container
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class kubelet cyan;
class dockerbuild green;
class dockerfile lavender;
class containerd,crio orange;
class dockerimage,runC cyan;
class container blue2;
{{< /mermaid >}}

## Summary

In this section, you hopefully learned:
- Docker has many faces; it is not just one component.
- Many Docker-derived and branded components are still relevant in Kubernetes.
- The old Docker Daemon `dockerd` was split into `containerd` and `runc`.
- In 2015, the Docker image spec and `runc` runtime spec were given to the Open Container Initiative (OCI).
- You can call a container image a Docker image for historical reasons or an OCI image for standardized reasons, or just called it a container image. All are correct.
- In 2016, Kubernetes formalized the Container Runtime Interface (CRI).
- Therefore, you can use `containerd`, `cri-o`, or other container runtimes in Kubernetes.
