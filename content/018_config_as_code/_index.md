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

<!-- TODO: add ConfigMap(s) mapped to volume(s) for more complex data as files -->

## Success

In this lesson, you:
- Created a `ConfigMap` and populated it with data.
- Changed the direct value in the `env` section of a `Pod`'s container to use a *reference* to your `ConfigMap`.
- Reprovisioned the pod with this new definition.
- Confirmed that the value from the `ConfigMap` appears in the pod.
