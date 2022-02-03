---
title: "Move Configuration out of Pod"
chapter: false
weight: 20
draft: false
---

## Configuration as Code

You can implement configuration values for your apps and microservices in environment variables, mounted volumes, or behind other services. Configuration as Code (CaC) is the philosophy which starts with moving configuration values out of the container images. As a best practice, you should consider going further. You can pull your configuration values out of the pod specification itself. 
Where should your configuration values go? There's a kind of Kubernetes object for that!

## `ConfigMap` objects

A [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) is a kind of Kubernetes resource that implements as **key-value** store, like many other configuration file and key-value database technologies. The values in a `ConfigMap` can be simple values or complicated structures of values.

Let's start simple. 

{{< step >}}Make a `ConfigMap` with the value you used to directly put in the `env` portion of a container definition in a `Pod` spec.{{< /step >}}

```bash
cat > ~/environment/004-greeting-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting
  namespace: dev
data:
  greeting: Bonjour de
EOF

kubectl apply -f ~/environment/004-greeting-configmap.yaml
```

`kubectl` will respond as follows.
{{< output >}}
configmap/greeting created
{{< /output >}}

## Consume the `ConfigMap` as an Environment Variable

You can refer to `ConfigMap` resources within other Kubernetes resources. 
For example, you can change your `Pod` spec to [pull the value of a container's environment variables from a `ConfigMap`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data). Do that now.  

{{< step >}}Create this new version of your pod manifest with the `configMapKeyRef` section as follows.{{< /step >}}
```bash
cat << EOF >~/environment/005-demo-pod.yaml 
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

{{< step >}}Delete the prior version of the pod.{{< /step >}}

```bash
kubectl -n dev delete -f ~/environment/003-demo-pod.yaml 
```

{{< step >}}Then deploy this replacement pod which uses the `ConfigMap`.{{< /step >}}

```bash
kubectl -n dev apply -f ~/environment/005-demo-pod.yaml 
```


{{< step >}}`exec` into your new pod to test it from inside as follows.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

If successful you will see the following.
{{< output >}}
Bonjour de demo
{{< /output >}}

## Check when a container picks up changes

If you change a value in a `ConfigMap`, does the container in your pod automatically receive the change? 

{{< step >}}To find out, change the value in the configmap:{{< /step >}}

```bash
sed <~/environment/004-greeting-configmap.yaml -e 's/Bonjour de/Hallo aus/' >~/environment/006-greeting-configmap.yaml
kubectl apply -f ~/environment/006-greeting-configmap.yaml
```

which is confirmed as follows:

{{< output >}}
configmap/greeting configured
{{< /output >}}

{{< step >}}Confirm the change is in your "greeting" `ConfigMap`:{{< /step >}}
```bash
kubectl get cm -n dev greeting -o yaml
```

This shows much additional metadata, but confirms the change too:

{{< output >}}
apiVersion: v1
data:
  greeting: Hallo aus
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"greeting":"Hallo aus"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"greeting","namespace":"dev"}}
  creationTimestamp: "2022-01-19T01:10:04Z"
  name: greeting
  namespace: dev
  resourceVersion: "487510"
  uid: ac32805a-78fd-4f70-82c0-8678525f7825
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
In *processes*, and therefore the *containers* that contain them, values of environment variables ***bind*** (are assigned) their values at process start (`exec(2)`, not `fork(2)`) time. Therefore, because the pod's container and its processes continued to run, but was *not restarted*, it is still running with the old stale value.
{{% /notice %}}

{{< step >}}Check the state of your pod again.{{< /step >}}
```bash
kubectl get pod -n dev demo
```

{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   0          10h
{{< /output >}}

How do you get the container to restart, which will restart the process which has the stale value?

{{< step >}}Kill the current instance of the container by killing the process at the root of its process hierarchy.{{< /step >}}

```bash
kubectl -n dev exec demo -it -- kill 1
kubectl get pods -n dev
```

{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   1          10h
{{< /output >}}

{{< step >}}Now that the container has restarted, check if it picked up the new value from the `ConfigMap`.{{< /step >}}

```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

{{< output >}}
Hallo aus demo
{{< /output >}}

{{% notice note %}}
A process does not pick up new environment variable values until it is restarted. Restart pods to have them consume revised environment variable values, including but not limited to those which reference `ConfigMap` values.
{{% /notice %}}

## Reference more complex data

In the first `ConfigMap` example, you referred to one single datum within that map. `ConfigMap`s can be more complicated than that one, therefore you should know how to reference values from more complex structures. Let's take a small step. 

{{< step >}}Create another `greeting` configmap manifest with more interesting key-value pairs in the data section, and apply it.{{< /step >}}
```bash
cat > ~/environment/007-greeting-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting
  namespace: dev
data:
  locale: fr-FR
  greeting: Bonjour de            # retain for backward compatibility with initial reference
  messages.greeting: Bonjour de   # accessed as messages.greeting
  messages.goodbye: Adieu de      # accessed as messages.goodbye
EOF

kubectl apply -f ~/environment/007-greeting-configmap.yaml
```

{{< step >}}Now change the way your container refers to that `ConfigMap`. Select the value `messages.greeting` this time.{{< /step >}}

```bash
cat << EOF >~/environment/008-demo-pod.yaml 
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
          key: messages.greeting      # the qualified key within the (apparently) "hierarchical" data section 
EOF
```

{{< step >}}Delete the prior version of the pod.{{< /step >}}

```bash
kubectl -n dev delete -f ~/environment/005-demo-pod.yaml 
```

{{< step >}}Then deploy this replacement pod which uses the revised `ConfigMap`.{{< /step >}}

```bash
kubectl -n dev apply -f ~/environment/008-demo-pod.yaml 
```


{{< step >}}`exec` into your new pod to test it from inside as follows.{{< /step >}}
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

If successful you will see the following.
{{< output >}}
Bonjour de demo
{{< /output >}}

{{% notice note %}}
The supposed "hierarchy" of the keys within the `ConfigMap` is only an illusion. The key-value pairs in the `data` section of Kubernetes config maps do not use all the hierarchical aspects of YAML for their data values. Your values should be strings, not maps themselves. However, this is not a true limitation. You can provide keys with the flavor of hierarchy, such as `messages.greeting`, and refer to those keys as such when you refer to that key in the config map.
{{% /notice %}}


{{< mermaid >}}
graph TB
configmap/greeting-->metadata
metadata-->name
metadata-->namespace
configmap/greeting-->data
data-->locale
data-->greeting
data-->messages.greeting
data-->messages.goodbye
{{< /mermaid >}}

## Consuming `ConfigMap`s as `volumes`

Environment variables are just one of the ways you can use `ConfigMap` values. Another way that values from a `ConfigMap` can be accessed from with a container (or container(s)) in a pod is by mapping a `ConfigMap` as a `volume` in a pod, then *mount* that `volume` into a container.

{{% notice note %}}
While the "greeting" config map could be mounted in a volume, let's look at another style of `ConfigMap` as well.
{{% /notice %}}

{{< step >}}Create a "quotes" `ConfigMap` as follows:{{< /step >}}

```bash
cat > ~/environment/009-quotes-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: quotes
  namespace: dev
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

kubectl apply -f ~/environment/009-quotes-configmap.yaml
```

Take note of the two ***keys*** in the `data` section:
- `daisy-bell.txt` looks like a file name, but is simply a key in this map like any other
- `hamlet-nutshell.txt` is also in classic UNIX filename style

Each of these keys has a multi-line value. Why is the punctuation different after the colon and before those value lines? We'll refer back this source YAML later and look at the effects of the `|` used with `dasiy-bell.txt` and the `>` used with `hamlet-nutshell.txt`.

{{< step >}}Get a list of the current `ConfigMap` objects in the `dev` Kubernetes namespace:{{< /step >}}
```bash
kubectl get configmaps -n dev
```

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
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"daisy-bell.txt":"Daisy, Daisy,\nGive me your answer, do!\nI'm half crazy,\nAll for the love of you!\nIt won't be a stylish marriage,\nI can't afford a carriage,\nBut you'll look sweet on the seat\nOf a bicycle built for two!\n","hamlet-nutshell.txt":"\"O God,  I could be bound in a nutshell,  and count myself a king of infinite space –  were it not that I have bad dreams.\" - William Shakespeare, Hamlet\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"quotes","namespace":"dev"}}
  creationTimestamp: "2022-01-19T14:12:56Z"
  name: quotes
  namespace: dev
  resourceVersion: "498883"
  uid: cef82116-f001-446e-bf3d-250cdce8bee1
{{< /output >}}

Recall that the source YAML in the Kubernetes manifest for this "quotes" `ConfigMap` used different punctuation for the two values in the `data` section:
- `|` before the value lines ***retains*** the vertical spacing of the newlines. Preserving newlines is important in many configuration files and data files used by your app software, add-on daemons, and operating system components.
- `>` before the value lines ***folds*** the newlines into spaces. This makes the data value all a one line character string. This is useful when you simply need the string value, but don't care about preserving newlines.
The choice of which notation you use depends largely on the needs of the software or humans consuming those values.

Now for the ***fun*** part. You can create `volumes` in your pods which refer to config maps. Then you can mount any of that pod's volumes within each of the pod's containers' *filesystem* namespaces. The power of this should not be underestimated. Not only can you map *arguments* and *environment variables* into your running containers, but you can map `ConfigMap` data as files into your containers. Think about that for a moment. You do not have to build new container *images* when simply want containers to use different files. These files could be data files, configuration files, et cetera. Let's note in passing that a `ConfigMap` can not only have a `data` section, but a `binaryData` section as well. How do you want to get those beautiful `png` graphic images into your pods? More on that later. For now, let's simply mount a volume with some simple text files.

{{< step >}}Create yet another version of your pod manifest, this one with both the `env` reference, and now adding a `volumeMounts` to the newer configmap.{{< /step >}}
```bash
cat << EOF >~/environment/010-demo-pod.yaml 
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
          key: messages.greeting      # the qualified key within the (apparently) "hierarchical" data section 
    volumeMounts:            # the "volumeMounts" in a given container are declared parallel to "env"
    - name: quotes-volume    # the name of the volume you want to mount, from the pod's "volumes" section
      mountPath: /var/quotes # the container filesystem namespace path - refer to this *inside* the container
EOF
```

{{< step >}}Delete the prior version of the pod.{{< /step >}}

```bash
kubectl -n dev delete -f ~/environment/008-demo-pod.yaml 
```

{{< step >}}Now deploy this pod which uses one `ConfigMap` for the `env` variable and consumes another `ConfigMap` as a volume.{{< /step >}}

```bash
kubectl -n dev apply -f ~/environment/010-demo-pod.yaml 
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

## Deploy the same Pod spec with different ConfigMap objects

Whether you're using simple values like a greeting or configuration/data files in your `ConfigMap` objects mapped into your `Pod`s, how can you have two or more pods using the same pod specification, yet actually referring to different `ConfigMap` objects?

***Namespaces***.

Yes, you might have noticed that in the prior examples you did not specify the `namespace: dev` within the `metadata` of the `Pod` manifest files? You provided the `-n dev` namespace designation on the command line for `kubectl apply`. Why? So that you can *use the same pod manifest for different namespaces*. 

Create two new configmaps similar to the `~/environment/007-greeting-configmap.yaml` one. Put these in different namespaces. In these cases, the `namespace: test` and `namespace: staging` are specified in the manifests rather than on the command line. Your could do distinguish the namespace of each `ConfigMap` on the command line instead using `-n test` and `-n staging` respectively when you `apply` those manifests.

{{% notice note %}}
The following assumes that you have already created a `test` namespace. If not, create one as follows before making the next configmap.
{{% /notice %}}

{{< step >}}If you do not *already* have a `test` namespace, create one now. Skip this if you have one.{{< /step >}}
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: test
EOF
```

{{< step >}}Create the English version of the "greeting" config map in the "test" namespace.{{< /step >}}

```bash
cat > ~/environment/011-greeting-configmap-en.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting
  namespace: test
data:
  locale: en-GB
  greeting: Hello from                       # retain for backward compatibility with initial reference
  messages.greeting: Who's there?            # accessed as messages.greeting
  messages.goodbye: Goodnight, sweet prince  # accessed as messages.goodbye
EOF
kubectl -n test apply -f ~/environment/011-greeting-configmap-en.yaml 
```

{{< step >}}Make another namespace for "staging".{{< /step >}}

```bash
cat <<EOF | tee ~/environment/012-staging-namespace.yaml | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: staging
EOF
```

{{< step >}}And yet another config map for that staging namespace.{{< /step >}}

```bash
cat > ~/environment/013-greeting-configmap-de.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting
  namespace: staging
data:
  locale: de-DE
  greeting: Hallo aus                   # retain for backward compatibility with initial reference
  messages.greeting: Hallo aus          # accessed as messages.greeting
  messages.goodbye: Auf Wiedersehen von # accessed as messages.goodbye
EOF
kubectl -n staging apply -f ~/environment/013-greeting-configmap-de.yaml 
```

{{< step >}}If you already have the "demo" pod running in the `dev` namespace, simply:{{< /step >}}

```bash
kubectl -n test apply -f ~/environment/008-demo-pod.yaml 
kubectl -n staging apply -f ~/environment/008-demo-pod.yaml 
```

{{% notice note %}}
Be careful to use the pod manifest that does *not* have the "quotes" volume mapping, as the supporting "quotes" `ConfigMap` is only in the `dev` namespace, and *not* in the `test` and `staging` namespaces.
{{% /notice %}}

{{< step >}}Check which pods are running:{{< /step >}}
```bash
kubectl get pods -A | grep demo
```

{{% notice note %}}
The `-A` option is short for `--all-namespaces`. 
{{% /notice %}}

You should have a similar set of pods running as this:
{{< output >}}
dev              demo         1/1     Running   0          48m
staging          demo         1/1     Running   0          43s
test             demo         1/1     Running   0          43s
{{< /output >}}

{{< step >}}Query each of the pods using `exec` in and `curl` from the inside:{{< /step >}}
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

{{< output >}}
Bonjour de demo
{{< /output >}}

```bash
kubectl -n test exec demo -it -- curl http://localhost:80
```
{{< output >}}
Who's there? demo
{{< /output >}}

```bash
kubectl -n staging exec demo -it -- curl http://localhost:80
```
{{< output >}}
Hallo aus demo
{{< /output >}}

Behold, the power of Kubernetes namespaces. Truly, this was a simple example. Think about how you can leverage off this kind of independent configuration using the same pod spec to deploy pods tailored to specific needs.

## Success

In this lesson, you:
- Created a `ConfigMap` and populated it with data.
- Changed the direct value in the `env` section of a `Pod`'s container to use a *reference* to your `ConfigMap`.
- Reprovisioned the pod with this new definition.
- Confirmed that the value from the `ConfigMap` appears in the pod.
- Made changes to the `ConfigMap` and showed that they do not appear in the pod.
- Restarted the pod to get the new value to be used.
- Created a `ConfigMap` with the illusion of hierarchy, and pointed your pod to a such a value.

Then you:
- Mounted a `ConfigMap` as a `volume` in a pod, and mounted that volume inside one of the pod's containers. The key-value pairs within the config map appear as files within the mounted path in the container. Each key appears as a filename, and each corresponding value is the contents of those files.
- Created separate config maps in separate Kubernetes namespaces and witnessed the distinct output from each of the pods using those separate definitions.