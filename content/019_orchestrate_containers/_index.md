---
title: "Orchestrate Containers"
chapter: false
weight: 19
draft: false
---

With your cluster in place and your CLI operational, you are now poised to run some containerized workloads on Kubernetes.

## Manifest generation (a side note)

The running state of a Kubernetes cluster is defined by objects such as [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) and [pods](https://kubernetes.io/docs/concepts/workloads/pods/) which require YAML manifests in order to be created.
Familiarizing oneself with the structure of these manifests can be a significant barrier for beginners.
`kubectl` supports a number of convenient factory-style commands, known as [generators](https://kubernetes.io/docs/reference/kubectl/conventions/#generators), which can build Kubernetes objects without you needing to code their underlying manifests by hand.

Generators also supports a **dry run** facility, enabling them to serve as crude but effective manifest builders.
This behaviour can be observed when executing the following command.

{{< step >}}Generate a Kubernetes `Pod` manifest.{{< /step >}}
```bash
kubectl run dummy-pod --image dummy --dry-run=client -o yaml
```

{{% notice note %}}
Dry runs are a client-side operation.
They will never alter the running state of your cluster.
{{% /notice %}}

Generators are incredibly useful for when you are getting started and need reminding of what goes where but they quickly become an inadequate crutch that holds you back as your requirements expand beyond the simplest use cases.
For this reason you will not be using them today.
Whenever you require an object you will, with the help of this tutorial, build a YAML manifest.

## Namespace creation

<!-- Imagine you need to address two individuals named, for example, Elizabeth and Elizabeth.
To uniquely identify them you might choose to ask for their surnames and refer to them by their full names.
Kubernetes objects can encounter the same type of identity problem.
For example, if you have two pods named **demo** that need to run concurently, you must place them in separate namespaces. -->
Most Kubernetes objects expect to be placed into a namespace.
If you fail to explicitly place any such object it will be sent to the **default** namespace.
You will almost always want to place objects into an explicit namespace, so you need to know how to create one.

{{< step >}}Create your namespaces manifest then instruct Kubernetes to ingest it using `kubectl apply` which will cause Kubernetes to translate your manifests into objects.
This manifest defines two namespaces, `dev` and `test`.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/001-dev-test-namespaces.yaml | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: test
EOF
```

Example output:
{{< output >}}
namespace/dev created
namespace/test created
{{< /output >}}


{{% notice note %}}
Whilst building the manifest, you will have observed the use of the [tee](https://en.wikipedia.org/wiki/Tee_(command)) command which helps condense the file persistence and manifest application steps.
The result of the command you ran is identical to using `kubectl create namespace <NAME>` (i.e. a generator) except the one-line version would not capture and/or persist any associated manifest to the local filesystem (Cloud9).
{{% /notice %}}

{{< step >}}Inspect the current list of namespaces as follows.{{< /step >}}
```bash
kubectl get namespaces
```

Example output:
{{< output >}}
NAME                 STATUS   AGE
default              Active   17h
dev                  Active   8m
kube-node-lease      Active   17h
kube-public          Active   17h
kube-system          Active   17h
local-path-storage   Active   17h
test                 Active   8m
{{< /output >}}

The list returned will include, among others, the newly created namespaces.
For the time being you only care about your new **dev** namespace so forget the others you see.
You now have a custom namespace which will provide a home for your app.

{{% notice note %}}
If you ever need to dispose of this namespace object you could use either `kubectl delete -f ~/environment/001-dev-test-namespaces.yaml` or simply `kubectl delete namespace <NAME>`
{{% /notice %}}

## Image registries (not today, thanks!)

Once a container image has been successfully built and tagged (e.g. `docker build`) it is typically published (**push**ed) to an image registry.
The most well known registry is [Docker Hub](https://hub.docker.com/) but others such as [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) exist.
Registries are to your container images what GitHub is to your source code - a place where a wider range of consumers can fetch (**pull**) your wares.
You will ultimately need to know how image registries work.
Today, to keep your TODO list manageable, you will inject your container image **directly** from your Cloud9 Docker image cache into your Kubernetes cluster.
This allows you to develop your images locally and completely bypass the **push** and **pull** mechanisms.

{{< step >}}Confirm that `demo:1.0.0` is still cached locally.{{< /step >}}
```bash
docker images demo:1.0.0
```

{{< step >}}Load your container image into your `kind`-based Kubernetes cluster.
This step populates the node caches which suppresses the need to **pull** your image.{{< /step >}}
```bash
kind load docker-image demo:1.0.0
```

The output you see will be as follows.
{{< output >}}
Image: "demo:1.0.0" with ID "sha256:4de...913" not yet present on node "kind-worker3", loading...
Image: "demo:1.0.0" with ID "sha256:4de...913" not yet present on node "kind-worker2", loading...
Image: "demo:1.0.0" with ID "sha256:4de...913" not yet present on node "kind-worker", loading...
Image: "demo:1.0.0" with ID "sha256:4de...913" not yet present on node "kind-control-plane", loading...
{{< /output >}}

{{% notice note %}}
This image injection technique is specific to `kind` and is just for use in dev environments.
You would **never** normally attempt to bypass your image registry like this.
{{% /notice %}}

## Container orchestration

The daemon component of Docker is an example of a [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).
Kubernetes extends the functionailty of container runtimes and is more appropriately referred to as a [container orchestrator](https://docs.docker.com/get-started/orchestration/).
You will start by proving that Kubernetes can replicate the functionality of Docker (i.e. run a single container).
Pods are the smallest deployable units of compute resource you can create and manage in Kubernetes.
Pods can be comprised of multiple containers but since we are starting with just one you can, **for now**, think about the terms container and pod as being broadly interchangeable.

To run a pod you need a YAML manifest which represents the pod.
As per the namespaces created previously, with some judicious use of [piped commands](https://en.wikipedia.org/wiki/Pipeline_(Unix)) you can build and persist your pod manifest before asking Kubernetes to ingest it, as follows.

{{< step >}}Create a `Pod` manifest, then use `kubectl apply` to provision the pod in your Kubernetes cluster.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/002-demo-pod.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demo                 # a name for your POD
spec:
  containers:                # a pod CAN consist of multiple containers, but this one has only one
  - name: demo               # a name for your CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
EOF
```

Example output:
{{< output >}}
pod/demo created
{{< /output >}}

{{% notice note %}}
Many Kubernetes objects, including pods, are destined for a namespace.
Recognize how the above invocation includes `kubectl -n dev` which ensures the pod is created in the `dev` namespace.
It is advantageous to build some muscle-memory around the consistent and explicit use of namespaces.
{{% /notice %}}

Compared to Docker, Kubernetes is more conservative when it comes to exposure of your apps at the host (VM) level.
How Kubernetes handles pod networking is deferred until a later section on [services](https://kubernetes.io/docs/concepts/services-networking/service/).
{{< step >}}In the meantime, as you did with the Docker CLI, you can `kubectl exec` into your running pod to test it from inside as follows.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

Example output:
{{< output >}}
Hello from demo
{{< /output >}}

Congratulations.
You have now reproduced the standard behavior of Docker but there is an important difference we need to discuss.
Kubernetes supports **restarts** by default.

{{< step >}}To see this, begin by inspecting the single running pod.{{< /step >}}
```bash
kubectl -n dev get pods
```

Example output:
{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   0          5m14s
{{< /output >}}

`kubectl` will respond as shown.
Check the `RESTARTS` column.
It is unlikely that your pod will have been restarted yet.

{{< step >}}As you did with the Docker CLI, run the following `kubectl` command to simulate a failure in the pod.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- kill 1
```

{{< step >}}Then recheck the `RESTARTS`{{< /step >}}
```bash
kubectl -n dev get pods
```

Example output:
{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   1          16m
{{< /output >}}

This reveals evidence that Kubernetes automatically restarted your pod.
You have witnessed something simple but very important about Kubernetes.
When Kubernetes ingests your manifests it becomes duty-bound to **continually** honour those requirements, until those requirements are altered.
This strategy is known as the [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) and is prevalent throughout Kubernetes in the form of [controllers](https://kubernetes.io/docs/concepts/architecture/controller/).
Pod restarts are evidence of this strategy at work and an indicator that Kubernetes is more than a simple container runtime, it is a **container orchestrator**.

Obviously there is more to being a container orchestrator than just automatic restarts but this will suffice for now.
You will learn more as you progress.

## Overriding environment variables
When you deployed your containerized app into Docker you observed how it was possible to provide overrides for environment variables which meant you were able to separate your code from its config.
You can also provide [environment variable overrides in Kubernetes](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/).
Your Kubernetes manifests are a form of [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code).
As such you would normally be storing versioned YAML files under source control.
Today, we will create the reconfigured manifest under a **new** file name so you can observe the difference.

{{< step >}}First, dispose of your existing pod.{{< /step >}}
```bash
kubectl -n dev delete pod demo
```

Example output:
{{< output >}}
pod/demo deleted
{{< /output >}}

{{% notice note %}}
When reconfiguring running Kubernetes objects, such as your `demo` pod, there are some field mutations which can be handled in-place and some which cannot.
Your change falls into the latter variety, hence the `delete` operation.
Deletion implies downtime, and much of your future endeavors with Kubernetes will be in the pursuit of zero downtime.
{{% /notice %}}

{{< step >}}Then deploy its replacement with the override in place.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/003-demo-pod.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demo                 # a name for your POD
spec:
  containers:                # a pod CAN consist of multiple containers, but this one has only one
  - name: demo               # a name for your CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
    env:
    - name: GREETING         # the overridden environment variable
      value: Hi from
EOF
```

Example output:
{{< output >}}
pod/demo created
{{< /output >}}

{{< step >}}`exec` into your new pod to test it from inside as follows.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

Example output:
{{< output >}}
Hi from demo
{{< /output >}}

## Success

In this exercise you did the following.

- Became acquainted with Kubernetes generators and image registries ... and why you will not use them today.
- Created your first namespace manifest in YAML and deployed it with `kubectl apply`.
- Built your first pod manifest and used it to deploy a single instance of your app into Kubernetes.
- Used `kubectl exec` to shell into your running app and tested it from within that session.
- Witnessed Kubernetes restarting a pod.
- Saw how manifests permit overrides for environment variables.
