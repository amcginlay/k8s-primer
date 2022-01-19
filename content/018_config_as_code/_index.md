---
title: "Move Configuration out of Pod"
chapter: false
weight: 018
draft: false
---

## Configuration as Code

You can implement configuration values for your apps and microservices in environment variables, mounted volumes, or behind other services. Configuration as Code (CaC) is the philosophy which starts with moving configuration values out of the container images. As a best practice, you should consider going further. You can pull your configuration values out of the pod specification itself. 
Where should your configuration values go? There's a kind of Kubernetes object for that!

## `ConfigMap` objects

A [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) is a kind of Kubernetes resource that implements as **key-value** store, like many other configuration file and key-value database technologies. The values in a `ConfigMap` can be simple values or complicated structures of values.

Let's start simple. Make a `ConfigMap` with the value you used to directly put in the `env` portion of a container definition in a `Pod` spec.

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

Delete the prior version of the pod.

```bash
kubectl -n dev delete -f ~/environment/003-demo-pod.yaml 
```

Then deploy this replacement pod which uses the `ConfigMap`.

```bash
kubectl -n dev apply -f ~/environment/005-demo-pod.yaml 
```


`exec` into your new pod to test it from inside as follows.
```bash
kubectl -n dev exec demo -it -- curl http://localhost:80
```

If successful you will see the following.
{{< output >}}
Bonjour de demo
{{< /output >}}

## Check when a container picks up changes

If you change a value in a `ConfigMap`, does the container in your pod automatically receive the change? To find out, change the value in the configmap:

```bash
sed <~/environment/004-greeting-configmap.yaml -e 's/Bonjour de/Hallo aus/' >~/environment/006-greeting-configmap.yaml
kubectl apply -f ~/environment/006-greeting-configmap.yaml
```

which is confirmed as follows:

{{< output >}}
configmap/greeting configured
{{< /output >}}

Confirm the change is in your "greeting" `ConfigMap`:
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

Run the web query to see the current value in the running container:

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

```bash
kubectl get pod -n dev demo
```

{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   0          10h
{{< /output >}}

How do you get the container to restart, which will restart the process which has the stale value?

```bash
kubectl -n dev exec demo -it -- kill 1
kubectl get pods -n dev
```

{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   1          10h
{{< /output >}}

Now that the container has restarted, check if it picked up the new value from the `ConfigMap`.

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

Now change the way your container refers to that `ConfigMap`. Select the value `messages.greeting` this time.

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

Delete the prior version of the pod.

```bash
kubectl -n dev delete -f ~/environment/005-demo-pod.yaml 
```

Then deploy this replacement pod which uses the revised `ConfigMap`.

```bash
kubectl -n dev apply -f ~/environment/008-demo-pod.yaml 
```


`exec` into your new pod to test it from inside as follows.
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

<!-- TODO: add ConfigMap(s) mapped to volume(s) for more complex data as files -->

## Success

In this lesson, you:
- Created a `ConfigMap` and populated it with data.
- Changed the direct value in the `env` section of a `Pod`'s container to use a *reference* to your `ConfigMap`.
- Reprovisioned the pod with this new definition.
- Confirmed that the value from the `ConfigMap` appears in the pod.
- Made changes to the `ConfigMap` and showed that they do not appear in the pod.
- Restarted the pod to get the new value to be used.
- Created a `ConfigMap` with the illusion of hierarchy, and pointed your pod to a such a value.