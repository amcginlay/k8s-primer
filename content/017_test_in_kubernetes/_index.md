---
title: "Test Your App in Kubernetes"
chapter: false
weight: 017
draft: false
---

## Manifest generation (a side note)

The running state of a Kubernetes cluster is defined by objects such as [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) and [pods](https://kubernetes.io/docs/concepts/workloads/pods/) which require YAML manifests in order to be created.
Familiarizing oneself with the structure of these manifests can be a significant barrier for beginners.
`kubectl` supports a number of convenient factory-style commands, known as [generators](https://kubernetes.io/docs/reference/kubectl/conventions/#generators), which can build Kubernetes objects without you needing to code their underlying manifests by hand.

Generators also supports a **dry run** facility, enabling them to serve as crude but effective manifest builders.
This behaviour can be observed when executing the following command.
```bash
kubectl run dummy-deployment --image dummy --dry-run=client -o yaml
```

{{% notice note %}}
Dry runs are a client-side operation.
They will never alter the running state of your cluster.
{{% /notice %}}

Generators are incredibly useful for when you are getting started and need reminding of what goes where but they quickly become an inadequate crutch that holds you back as your requirements expand beyond the simplest use cases.
For this reason you will not be using them today.
Whenever you require an object you will, with the help of this tutorial, build a YAML manifest.

## Image registries

Once a container image has been successfully built and tagged it is typically published (**push**) to an image registry.
The most well known registry is [Docker Hub](https://hub.docker.com/) but others such as [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) exist.
Registries are to your container images what GitHub is to your source code - a place where a wider range of consumers can fetch from (**pull**).
You will ultimately need to know how image registries work.
Today, to keep your TODO list manageable, you will inject your container image **directly** from your local Docker image cache into your Kubernetes cluster.
This allows you to develop your images locally and completely bypass the **push** and **pull** mechanisms.
```bash
kind load docker-image demo:1.0.0
```

The output you see will be as follows.
{{< output >}}
Image: "demo:1.0.0" with ID "sha256:893d88f41297dc99cf7a294ba7edfedca5d063facf6e89a59a608cc5f7502688" not yet present on node "kind-control-plane", loading...
{{< /output >}}

{{% notice note %}}
This image injection technique is specific to `kind` and is just for use in dev environments.
You would **never** normally attempt to bypass your image registry like this.
{{% /notice %}}

## Namespace creation

<!-- Imagine you need to address two individuals named, for example, Elizabeth and Elizabeth.
To uniquely identify them you might choose to ask for their surnames and refer to them by their full names.
Kubernetes objects can encounter the same type of identity problem.
For example, if you have two pods named **demo** that need to run concurently, you must place them in separate namespaces. -->
Most Kubernetes objects expect to be placed into a namespace.
If you fail to explicitly place any such object it will be sent to the **default** namespace.
You will almost always want to place objects into an explicit namespace, so you need to know how to create one.

Create your namespace manifest then instruct Kubernetes to ingest it using `kubectl apply` which will cause Kubernetes to translate your manifests into objects.
This manifest defines two namespaces, `dev` and `test`.
```bash
cat > ~/environment/001-namespaces.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---                          # "---" marker allows YAML to support mulitple documents per file
apiVersion: v1
kind: Namespace
metadata:
  name: test
EOF

kubectl apply -f ~/environment/001-namespaces.yaml
```

`kubectl` will respond as follows.
{{< output >}}
namespace/dev created
{{< /output >}}


{{% notice note %}}
The result of the commands you ran are identical to using `kubectl create namespace dev` (a generator) except the one-line version would not capture and/or persist the associated manifest to the local filesystem (Cloud9).
{{% /notice %}}

Inspect the current list of namespaces as follows.
```bash
kubectl get namespaces
```

The list returned will include the newly created **dev** namespace.
{{< output >}}
NAME                 STATUS   AGE
default              Active   17h
dev                  Active   8m
kube-node-lease      Active   17h
kube-public          Active   17h
kube-system          Active   17h
local-path-storage   Active   17h
{{< /output >}}

For the time being you only care about the new **dev** namespace so forget you have seen any others.
You now have a namespace to host your app.

{{% notice note %}}
If you ever need to dispose of this namespace object you could use either `kubectl delete -f ~/environment/001-dev-namespace.yaml` or simply `kubectl delete namespace dev`
{{% /notice %}}

## Container orchestration

The daemon component of Docker is an example of a [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).
Kubernetes extends the functionailty of container runtimes and is more appropriately referred to as a [container orchestrator](https://docs.docker.com/get-started/orchestration/).
You will start by proving that Kubernetes can replicate the functionality of Docker (i.e. run a single container).
Pods are the smallest deployable units of compute resource you can create and manage in Kubernetes.
Pods can be comprised of multiple containers but since we are starting with just one, you can think about these two terms as being interchangeable.

To run a pod you need a YAML manifest which represents the pod.
You can neatly create, persist and apply your minimum viable product pod like this.
```bash
cat << EOF | tee ~/environment/002-demo-pod.yaml | kubectl -n dev apply -f -
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

`kubectl` will respond as follows.
{{< output >}}
pod/demo created
{{< /output >}}

{{% notice note %}}
Many Kubernetes objects, including pods, are destined for a namespace.
Recognise how the above invocation reads `kubectl -n dev ...` which ensures the pod is created in the `dev` namespace.
It is advantageous to build some muscle-memory around the consistent and explicit use of namespaces.
{{% /notice %}}

Compared to Docker, Kubernetes is more conservative when it comes to exposure of your apps at the host (VM) level.
How Kubernetes handles pod networking is deferred until a later section on [services](https://kubernetes.io/docs/concepts/services-networking/service/).
In the meantime, we can `exec` into your running pod to test it from inside as follows.
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

If successful you will see the following.
{{< output >}}
Hello from demo
{{< /output >}}

Congratulations.
You have now reproduced the standard behaviour of Docker but there is an important difference we need to discuss.
Kubernetes supports **restarts** by default.

To see this, begin by inspecting the single running pod.
```bash
kubectl -n dev get pods
```

`kubectl` will respond as follows.
Check the `RESTARTS` column.
It is unlikely that your pod will have been restarted yet.
{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   0          5m14s
{{< /output >}}

As you did with Docker, run the following command to simulate a failure in the pod then recheck the running processes.
```bash
kubectl -n dev exec demo -it -- kill 1
```
```bash
kubectl -n dev get pods
```

You will now see how Kubernetes has automatically restarted your pod.
{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   1          16m
{{< /output >}}

You have witnessed something simple but very important about Kubernetes.
When Kubernetes ingests your manifests it becomes duty-bound to honour your requirements, continually, until your requirements are altered.
This strategy is known as the [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) and is prevalent throughout Kubernetes in the form of [controllers](https://kubernetes.io/docs/concepts/architecture/controller/).
Pod restarts are evidence of this strategy at work and an indicator that Kubernetes is more than a simple container runtime, it is a **container orchestrator**.

## Overriding environment variables
When you deployed your containerized app into Docker you observed how it was possible to provide overrides for environment variables which meant you were able to separate your code from its config.
Your Kubernetes manifests are a form of [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code).
As such you would normally be storing these YAML files under source control.
Today, we will create the reconfigured manifest in a **new file** so you can observe the difference.

First, dispose of your existing pod.
```bash
kubectl -n dev delete pod demo
```

Then deploy its replacement with the override in place.
```bash
cat << EOF | tee ~/environment/003-demo-pod.yaml | kubectl -n dev apply -f -
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

`exec` into your new pod to test it from inside as follows.
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

If successful you will see the following.
{{< output >}}
Hi from demo
{{< /output >}}

{{% notice note %}}
Only a select few fields are supported by Kubernetes update/patch operations and the `env` field is not (yet?) one of them.
This is why we deleted the original pod rather than applying a patch to the original - it would have failed otherwise.
THis is also why the pursuit of **zero-downtime** strategies is such a hot topic!
{{% /notice %}}

## Success

In this exercise you did the following.

- Became acquainted with Kubernetes generators and image registries ... and why you will not use them today.
- Created your first namespace manifest in YAML and deployed it with `kubectl apply`.
- Built your first pod manifest and used it to deploy a single instance of your app into Kubernetes.
- Used `kubectl exec` to shell into your running app and tested it from within that session.
- Witnessed Kubernetes restarting a pod.
- Saw how manifests permit overrides for environment variables.
