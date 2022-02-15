---
title: "Configuration As Code"
chapter: false
weight: 20
draft: false
---

You can implement configuration values for your apps and microservices in environment variables, mounted volumes, or behind other services.
Configuration as Code (CaC) is the philosophy which starts with moving configuration values out of the container images.
As a best practice, you should consider going further. You can pull your configuration values out of the pod specification itself.
Where should your configuration values go? There's a kind of Kubernetes object for that!

## ConfigMap objects

A [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) is a kind of Kubernetes resource that implements as **key-value** store, like many other configuration file and key-value database technologies.
The values in a `ConfigMap` can be simple values or complicated structures of values.

Let's start simple. 

{{< step >}}Make a `ConfigMap` which encapsulates an intended value for your `GREETING` environment variable.{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/004-greeting-configmap.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting
data:
  greeting: Bonjour de
EOF
```

Example output:
{{< output >}}
configmap/greeting created
{{< /output >}}

## Mapped Environment Variables

You can refer to `ConfigMap` resources within other Kubernetes resources. 
For example, you can change your `Pod` spec to [pull the value of a container's environment variables from a `ConfigMap`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data).
Do that now.  

{{< step >}}First, dispose of your existing pod.{{< /step >}}
```bash
kubectl -n dev delete pod demo
```

Example output:
{{< output >}}
pod/demo deleted
{{< /output >}}


{{< step >}}Create this new version of your pod manifest with the `configMapKeyRef` section as follows.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/005-demo-pod.yaml | kubectl -n dev apply -f -
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
      valueFrom:             # not just value: directly here but valueFrom: as a reference
        configMapKeyRef:     # a ConfigMap is the source of the data
          name: greeting     # the name of the ConfigMap
          key: greeting      # the key within the data section of that ConfigMap
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
Bonjour de demo
{{< /output >}}

## Check when a container picks up changes

If you change a value in a `ConfigMap`, does the container in your pod automatically receive the change? 

{{< step >}}To find out, change the value in the configmap:{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/006-greeting-configmap.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting
data:
  greeting: Hallo aus
EOF
```

which is confirmed as follows:

{{< output >}}
configmap/greeting configured
{{< /output >}}

{{< step >}}Confirm the change is in your "greeting" `ConfigMap`:{{< /step >}}
```bash
kubectl -n dev describe configmap greeting
```

Example output:
{{< output >}}
Name:         greeting
Namespace:    dev
Labels:       <none>
Annotations:  <none>

Data
====
greeting:
----
Hallo aus
...
{{< /output >}}

{{< step >}}Run the web query to see the current value in the running container:{{< /step >}}

```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

{{< output >}}
Bonjour de demo
{{< /output >}}

This is the *old* value you had assigned previously. Why? 

{{% notice note %}}
In *processes*, and therefore the *containers* that contain them, values of environment variables ***bind*** (are assigned) their values at process start time.
Because the pod's container was not restarted it is still running with the old value.
{{% /notice %}}

As shown previously, it is possible to force a restart by killing PID 1 from inside the pod.

{{< step >}}Kill the current instance of the container by killing the process at the root of its process hierarchy.{{< /step >}}

```bash
kubectl -n dev exec demo -it -- kill 1
```

{{< step >}}Now that the container has restarted, check if it picked up the new value from the `ConfigMap`.{{< /step >}}

```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

{{< output >}}
Hallo aus demo
{{< /output >}}

{{% notice note %}}
A process does not pick up new environment variable values until it is restarted.
Restart pods to force them to consume revised environment variable values, including those sourced from a `ConfigMap`.
{{% /notice %}}

## Consuming ConfigMaps as volumes

Environment variables are just one of the ways you can use `ConfigMap` values.
Another way that values from a `ConfigMap` can be accessed from a pod is by referencing the `ConfigMap` as a `volume`.

{{% notice note %}}
While the "greeting" config map could be mounted in a volume, let's look at another style of `ConfigMap`.
{{% /notice %}}

{{< step >}}Create a "quotes" `ConfigMap` as follows:{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/009-quotes-configmap.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: quotes
data:
data:
  daisy-bell.txt: |
    Daisy, Daisy,
    Give me your answer, do!
    I'm half crazy,
    All for the love of you!
    It won't be a stylish marriage,
    I can't afford a carriage,
    But you'll look sweet on the seat
    Of a bicycle built for two!
  hamlet-nutshell.txt: >
    "O God, 
    I could be bound in a nutshell, 
    and count myself a king of infinite space – 
    were it not that I have bad dreams."
    - William Shakespeare, Hamlet
EOF
```

Example output:
{{< output >}}
configmap/quotes created
{{< /output >}}

Take note of the two ***keys*** in the `data` section:
- `daisy-bell.txt` looks like a file name, but is simply a key in this map like any other
- `hamlet-nutshell.txt` is also in classic UNIX filename style

Each of these keys has a multi-line value.
Why is the punctuation different after the colon and before those value lines?
We'll refer back this source YAML later and look at the effects of the `|` used with `dasiy-bell.txt` and the `>` used with `hamlet-nutshell.txt`.

{{< step >}}Get a list of the current `ConfigMap` objects in the `dev` Kubernetes namespace:{{< /step >}}
```bash
kubectl get configmaps -n dev
```

Example output:
{{< output >}}
NAME               DATA   AGE
greeting           4      13h
kube-root-ca.crt   1      16h
quotes             2      2m8s
{{< /output >}}

{{< step >}}Witness how Kubernetes has formatted and stored those two quotes:{{< /step >}}

```bash
kubectl get cm -n dev quotes -o yaml
```

Example output:
{{< output >}}
apiVersion: v1
data:
  daisy-bell.txt: |
    Daisy, Daisy,
    Give me your answer, do!
    I'm half crazy,
    All for the love of you!
    It won't be a stylish marriage,
    I can't afford a carriage,
    But you'll look sweet on the seat
    Of a bicycle built for two!
  hamlet-nutshell.txt: |
    "O God,  I could be bound in a nutshell,  and count myself a king of infinite space –  were it not that I have bad dreams." - William Shakespeare, Hamlet
kind: ConfigMap
metadata:
...
{{< /output >}}

Recall that the source YAML in the Kubernetes manifest for this "quotes" `ConfigMap` used different punctuation for the two values in the `data` section:
- `|` before the value lines ***retains*** the vertical spacing of the newlines.
Preserving newlines is important in many configuration files and data files used by your app software, add-on daemons, and operating system components.
- `>` before the value lines ***folds*** the newlines into spaces.
This makes the data value all a one line character string.
This is useful when you simply need the string value, but don't care about preserving newlines.
The choice of which notation you use depends largely on the needs of the software or humans consuming those values.

Now for the ***fun*** part.
You can create `volumes` in your pods which refer to ConfigMaps.
Then you can mount any of those volumes from within any of the pod's containers where it will present itself as a file.
The power of this should not be underestimated.
Not only can you map *arguments* and *environment variables* into your running containers, but you can map `ConfigMap` data as files into your containers.
Think about that for a moment.
You do not have to build new container *images* when you simply want containers to use different files.
These files could be data files, configuration files, localization resources, et cetera.
Let's note in passing that a `ConfigMap` can not only have a `data` section, but a `binaryData` section as well.
How do you want to get those beautiful `png` graphic images into your pods? More on that later.
For now, let's simply mount a volume with some simple text files.

{{< step >}}First, dispose of your existing pod.{{< /step >}}

```bash
kubectl -n dev delete pod demo
```

{{< step >}}Create yet another version of your pod manifest, this one with both the `env` reference, and now adding a `volumeMounts` to the newer configmap.{{< /step >}}
```yaml
cat <<EOF | tee ~/environment/010-demo-pod.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demo                 # a name for your POD
spec:
  volumes:                   # a pod CAN contain multiple volumes, but this on has only one
  - name: quotes-volume      # this name of the volume is used to refer to it--e.g. in a "mount"
    configMap:               # specify the type of volume this is--we're referring to a ConfigMap
      name: quotes           # the name of the ConfigMap this volume refers to
  containers:                # a pod CAN consist of multiple containers, but this one has only one
  - name: demo               # a name for your CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
    env:
    - name: GREETING         # the overridden environment variable
      valueFrom:             # not just value: directly here but valueFrom: as a reference
        configMapKeyRef:     # a ConfigMap is the source of the data
          name: greeting     # the name of the ConfigMap
          key: greeting      # the qualified key within the (apparently) "hierarchical" data section 
    volumeMounts:            # the "volumeMounts" in a given container are declared parallel to "env"
    - name: quotes-volume    # the name of the volume you want to mount, from the pod's "volumes" section
      mountPath: /var/quotes # the container filesystem namespace path - refer to this *inside* the container
EOF
```

{{< step >}}`exec` into your new pod and list the files in `/var/quotes`.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- ls /var/quotes
```

{{< step >}}Now see what that `daisy-bell.txt` file looks like inside the container.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- cat /var/quotes/daisy-bell.txt
```

{{< step >}}Remember that you used `>` to told the lines of the Hamlet quote. Take a look at the result.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- cat /var/quotes/hamlet-nutshell.txt
```

{{% notice note %}}
In both cases, what happened to the indentation in your original Kubernetes manifest for the "quotes" `ConfigMap`?
While the `|` and `>` **retain** and **fold** the newlines in the content respectively, both of these multi-line value notations in YAML strip the indentation of those lines of data, according to the indentation of the element.
This will be significant not only for your Kubernetes manifests you write from scratch, but any you automatically generate, and any Kubernetes manifests you author via tools such as the package management tool `helm`.
{{% /notice %}}

## ConfigMaps are namespaced

Before we move on, reflect upon why this separation of code from config is desirable.

Your namespaces could be used to represent different environments (e.g. `dev` or `test`) and the apps which run there would perhaps connect to environment-specific database instances.
When we ask a pod to bind to a configmap it has no option but to bind to a named object (e.g. `connection-strings`) located in its own namespace as it has no further visibility.
This means that the pod definition in both namespaces could be identical whilst the associated configmaps, which share the same name, could be quite different.

## Success

In this lesson, you:
- Created a `ConfigMap` and populated it with data.
- Changed the direct value in the `env` section of a `Pod`'s container to use a *reference* to your `ConfigMap`.
- Reprovisioned the pod with this new definition.
- Confirmed that the value from the `ConfigMap` appears in the pod.
- Made changes to the `ConfigMap` and showed that they do not appear in the pod.
- Restarted the pod to get the new value to be used.
- Observed how it is possible to use `volumes` to consume `ConfigMaps` as files.
