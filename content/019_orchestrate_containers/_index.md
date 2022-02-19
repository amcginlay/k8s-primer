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

Generators also supports a **dry run** facility, which only writes the manifest but does not actually create the object, enabling them to serve as crude but effective manifest builders.
This behavior can be observed when executing the following command.

{{< step >}}Generate a Kubernetes `Pod` manifest.{{< /step >}}
```bash
kubectl run dummy-pod --image dummy --dry-run=client -o yaml
```

{{% notice note %}}
Dry runs are a client-side operation.
They will never alter the running state of your cluster.
{{% /notice %}}

Generators are incredibly useful for when you are getting started and need reminding of what goes where but they quickly become an inadequate crutch that holds you back as your requirements expand beyond the simplest use cases.
For this reason you will *not* be using them today.
Whenever you require an object you will, with the help of this tutorial, build a YAML manifest.

## Namespace creation

<!-- Imagine you need to address two individuals named, for example, Elizabeth and Elizabeth.
To uniquely identify them you might choose to ask for their surnames and refer to them by their full names.
Kubernetes objects can encounter the same type of identity problem.
For example, if you have two pods named **demo** that need to run concurently, you must place them in separate namespaces. -->
Most Kubernetes objects expect to be placed into a namespace.
If you fail to explicitly place any such object it will be sent to the **default** namespace.
You will almost always want to place objects into an explicit namespace, so you need to know how to create one.
While you're at it, create two, one for development and one for testing.
{{% expand "Bonus: Which kinds of objects are \"namespaced\"?" %}}
Each `kind` of Kubernetes object is either put in a namespace, i.e. **namespaced**, or *not* put in a namespace which means that such objects have global scope across the cluster, i.e. **non-namespaced**.
The `kubectl` command `api-resources` shows the `NAME`, `SHORTNAME`, `APIVERSION`, `NAMESPACED`, and `KIND` of each of the kinds of Kubernetes objects known in your cluster. You can use the classic UNIX `awk` command to collate the namespaced and non-namespaced kinds of objects.
```bash
kubectl api-resources | awk 'BEGIN{ns="";nonns=""};/true/{ns=ns FS $1};/false/{nonns=nonns FS $1};END{print "namespaced: " ns; print ""; print "non-namespaced:" nonns}'
```
In which list do pods appear?
{{% /expand %}}

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
You'll use the **test** namespace later; **dev** is enough for this lesson.
You now have a custom namespace which will provide a home for your app.

{{% notice note %}}
If you ever need to dispose of this namespace object you could use either `kubectl delete -f ~/environment/001-dev-test-namespaces.yaml` or simply `kubectl delete namespace <NAME>`
{{% /notice %}}

## Image registries (not today, thanks!)

Once a container image has been successfully built and tagged (e.g. `docker build`) it is typically published (**push**ed) to an image registry.
The most well known registry is [Docker Hub](https://hub.docker.com/) but others such as [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) exist.
Registries are to your container images what GitHub is to your source code - a place where a wider range of consumers can fetch (**pull**) your wares.
You will ultimately need to know how image registries work.
{{% expand "Bonus: Some CNCF-listed container registries" %}}
For future reference, you can visit the [Cloud Native Computing Foundation (CNCF) *landscape* container registry category](https://landscape.cncf.io/card-mode?category=container-registry&grouping=category) to find descriptions and links to other container registry offerings. Some popular ones are: [Harbor](https://goharbor.io), [Quay](https://quay.io), and [JFrog Artifactory](https://jfrog.com/artifactory/).
{{% /expand %}}

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
***Pods*** are the smallest deployable units of compute resource you can create and manage in Kubernetes.
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
A large majority of the operations you perform with `kubectl` are **"namespaced"**.
That is to say they only make sense in the context of a namespace.
In such cases, failure to explicitly provide the `-n` (or `--namespace`) switch will cause `kubectl` to implicitly engage with the `default` namespace.
It is possible to [override this behavior](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-preference) but think carefully before doing so.
You may become more productive but you will also be creating an environment in which painful mistakes becomes harder to avoid.
It is advantageous to build some muscle-memory around the consistent and explicit use of non-default namespaces.
{{% /notice %}}

{{% expand "Why not specify the namespace in the manifest instead of the pipeline?" %}}
You could have specified the `namespace` of your pod in the manifest in the `metadata` section. That is certainly a viable option. In fact, in some cases it may be preferred, because the manifest is more self-contained and complete. In far fancier scenarios than this introduction, you may use a package manager such as `helm` to inject variables into your manifests, and the namespace is one of many such possibilities there. However, without such a package manager, think about modularity and reusability. By *not* specifying the namespace within this pod manifest, you could easily reuse it and deploy many pods with that same manifest to different namespaces just by changing the command line. That's the tradeoff: change the imperative command line, or change the declarative manifest. Which approach is more manageable for your devops pipelines and methodologies? The answer may vary from customer to customer and project to project.
{{% /expand %}}

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

We could depict what you have just done like this:

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph pod[demo Pod]
  subgraph container[demo Container]
    demo((php in<br>demo<br>container))
    curl((curl in<br>demo<br>container))
  end
end
cloud9 -->|kubectl exec| curl
curl -->|curl :80| demo
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class pod yellow;
class container yelloworange;
class demo,curl blue2;
{{< /mermaid >}}

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

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph pod[demo Pod]
  subgraph container[demo Container]
    apache((PID 1<br>apache2<br>root))
    demo((PID 23<br>php<br>www-data))
    kill((PID 35<br>kill 1<br>root))
  end
end
cloud9 -->|kubectl exec| kill
kill -->|kill 1| apache
apache -.-|child| demo
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class pod yellow;
class container yelloworange;
class apache,demo,kill blue2;
{{< /mermaid >}}

{{< step >}}Then recheck the `RESTARTS`{{< /step >}}
```bash
kubectl -n dev get pods
```

Example output:
{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   1          16m
{{< /output >}}

{{% expand "Bonus: How can you check what processes are in your container in your pod?" %}}
You could use `ps` within a `kubectl exec` to see what is running in the container in your pod. While this assumes that you have `ps` installed in your container image, that is the case with your `demo` image. Try this: `kubectl -n dev exec demo -it -- ps -eo pid,ppid,cmd` to focus on the process id, parent process id, and command properties.
{{% /expand %}}

This reveals evidence that Kubernetes automatically restarted your pod.
You have witnessed something simple but very important about Kubernetes.
When Kubernetes ingests your manifests it becomes duty-bound to **continually** honour those requirements, until those requirements are altered.
This strategy is known as the [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) and is prevalent throughout Kubernetes in the form of [controllers](https://kubernetes.io/docs/concepts/architecture/controller/).
Pod restarts are evidence of this strategy at work and an indicator that Kubernetes is more than a simple container runtime, it is a **container orchestrator**.

Obviously there is more to being a container orchestrator than just automatic restarts but this will suffice for now.
You will learn more as you progress.

## Wave goodbye to PodSpecs

Really?
So soon?
So what was all the fuss about?

Whilst it is important to understand the elemental role PodSpecs play, the truth is we rarely see PodSpecs in the wild, 
The DNA of the PodSpec lives on in the `template` section of a more functional and forgiving object type called a **Deployment**.

If it helps, recall when you first started using EC2 [Auto Scaling groups (ASG)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html).
You needed to define a [Launch Template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html) which is like your declaration of intent - it says, "when I want an instance, *this* is what I want it to look like".
If you imagine the `template` section of a Deployment as being similar your EC2 Launch Template then the `replicas` attribute is like the **desired** size of your ASG.
If you set the `replicas` attribute to "1", you have a ready-made replacement for your PodSpec.

You will learn more about the capabilities of Deployments as we progress but, for now, let's just do the switch.

{{< step >}}First, dispose of your existing pod.{{< /step >}}
```bash
kubectl -n dev delete pod demo
```

Example output:
{{< output >}}
pod/demo deleted
{{< /output >}}

{{< step >}}Now replace it with the equivalent Deployment object.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/003-demo-deployment.yaml | kubectl -n dev apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 1             # think, ASG "Desired" size
  selector:
    matchLabels:
      app: demo
  template:               # think, ASG "launch template" setting
    metadata:             #
      labels:             #
        app: demo         #
    spec:                 #
      containers:         #
      - name: demo        #
        image: demo:1.0.0 #
EOF
```

Example output:
{{< output >}}
deployment.apps/demo created
{{< /output >}}

{{% notice note %}}
Recall our earlier discussion about manifest generation.
If you are struggling to recall the many attributes of a deployment manifest, this may help fill in the gaps for you: `kubectl create deployment dummy --image dummy:1.0 --dry-run=client -o yaml`
{{% /notice %}}

{{< step >}}Now check you can `kubectl exec` into your singleton pod via its deployment.{{< /step >}}
```bash
kubectl -n dev exec deployment/demo -it -- curl http://localhost:80
```

Example output:
{{< output >}}
Hello from demo-7c9cd496db-zsx5l
{{< /output >}}

We will be typically using **Deployment** objects from this point on.
Time to get back to the plot.

## Overriding environment variables
When you deployed your containerized app into Docker you observed how it was possible to provide overrides for environment variables which meant you were able to separate your code from its config.
You can also provide [environment variable overrides in Kubernetes](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/).
Your Kubernetes manifests are a form of [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code).
As such you would normally be storing versioned YAML files under source control.
Today, we will create the reconfigured manifest under a **new** file name so you can observe the difference.

{{< step >}}re-deploy the manifest with the environment variable override in place.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/004-demo-deployment.yaml | kubectl -n dev apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: demo:1.0.0
        env:
        - name: GREETING         # the overridden environment variable
          value: Hi from
EOF
```

Example output:
{{< output >}}
deployment.apps/demo configured
{{< /output >}}

{{< step >}}`exec` into your new pod to test it from inside as follows.{{< /step >}}
```bash
kubectl -n dev exec deployment/demo -it -- curl http://localhost:80
```

Example output:
{{< output >}}
Hi from demo-658bfb548-ts7px
{{< /output >}}

{{% notice note %}}
You will observe that when using **deployments** the names of pods are partially system generated and therefore subject to change during restarts.
You can view this behavior as an example of the [Pets vs Cattle](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/) analogy.
The reasoning for this will become more evident later on when you scale-out your deployment.
{{% /notice %}}

You have assigned a value for your `GREETING` environment variable at the pod specification level, which overrides the value for that variable in your `demo` container image you built with a `Dockerfile` earlier.
This is immensely powerful.
You have not changed or overwritten you immutable version `1.0.0` container image, but you have transcended it.
Your Kubernetes manifest for your pod overrode what was baked into the image.
You are in control of your configuration to extend how you use the container images you have.

## Orchestrate Containers Quiz

Please take the following quiz to review your knowledge of running containers in Kubernetes.

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## Logical Subdivisions

What are the two logical subdivisions--`dev` and `test`--you created in your Kubernetes cluster before deploying containers?

> You wrote a YAML manifest with two object declarations at the beginning of the lesson. What `kind` did you use?

- [ ] `Service`
- [ ] `Deployment`
- [x] `Namespace`
- [ ] `ConfigMap`

## Kubernetes smallest deployable compute

What is the smallest deployable unit of compute power in Kubernetes?

> What was the `kind` you used to deploy your `demo` app?

- [ ] Thread
- [ ] Process
- [ ] Container
- [x] Pod

## Pod access

How did you test access to your `demo` container?

> Which command gave you the results "Hello from demo"?

- [ ] `kubectl -n dev exec demo -it 80:8081 curl http://localhost:8081`
- [ ] `kubectl -n dev exec demo -p 8081:80 -- curl http://localhost:80`
- [ ] `kubectl -n dev exec demo -it -- curl http://localhost:8081`
- [x] `kubectl -n dev exec demo -it -- curl http://localhost:80`


## Ephemerality

What did Kubernetes do when you killed the root process `kill 1` in your pod?

> What did `get pods` show in after you had killed the process?

- [ ] Kubernetes marked the container as stopped and changed pod status to failed
- [x] Kubernetes restarted the container in the pod and kept pod status as running
- [ ] Kubernetes ignored the failure and did not change the container or pod status
- [ ] Kubernetes deleted and redeployed the whole pod, setting its age back to zero

## Wave Goodbye to Bare Pods

With which Kubernetes workload did you replace your pod?

> When you moved you pod spec into a `template`, what was the `kind` of object you wrapped that in?

- [ ] Task
- [x] Deployment
- [ ] DaemonSet
- [ ] ReplicaSet

## Environment Variables

When you added an `env` section to your container definition in your `Deployment` spec, what was the result?

> What is the behavior of environment variables between container image and Kubernetes pod?

- [x] the `env` of the deployment's pod(s) overrides the container image value from `ENV` in the Dockerfile
- [ ] the container image value from `ENV` in the Dockerfile overrides the `env` of the deployment's pod(s)
- [ ] both values get applied from the `env` of the deployment's pod(s) and the container image value from `ENV` in the Dockerfile
- [ ] an error occurs and neither value is assigned to an environment variable; you must delete the `ENV` value from the image to avoid conflict

{{< /quizdown >}}

## Success

In this exercise you did the following.

- Became acquainted with Kubernetes generators and image registries ... and why you will not use them today.
- Created your first namespace manifest in YAML and deployed it with `kubectl apply`.
- Built your first pod manifest and used it to roll out a single instance of your app into Kubernetes.
- Used `kubectl exec` to shell into your running app and tested it from within that session.
- Witnessed Kubernetes restarting a pod.
- Traded in PodSpecs for Deployments ... because you're a professional.
- Saw how manifests permit overrides for environment variables.
