---
title: "Pause Containers"
chapter: false
weight: 715
draft: false
---

## Pause Containers

In the [Pod Infrastructure lesson]({{< ref "appendix/714_pod" >}}), you created a pod and invesigated what it is made of.

Hopefully you learned:
- each container has its own `pid` and `mnt` namespaces
- Kubernetes adds a `pause` container to each of your pods
- each pod has its own `net`, `uts`, and `ipc` namespaces

## Purpose

In this lesson, you will answer the question: "What do pause containers do?"

## The Invisible Sibling

The `pause` container in each Kubernetes pod is like a silent watcher or invisible sibling to your pod's app containers.

{{< step >}}Start with a fresh pod. Delete your `demo` pod if it is still running, then create it again.{{< /step >}}

```bash
kubectl delete pod demo -n dev
kubectl apply -n dev -f ~/environment/002-demo-pod.yaml 
```

Example output (may vary if you were not still running a `demo` pod already):
{{< output >}}
pod "demo" deleted
pod/demo created
{{< /output >}}

{{< step >}}Does the pause container count in the container stats of a pod? Look.{{< /step >}}

```bash
kubectl get pods -n dev demo -o wide
```

Example output:
{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
demo   1/1     Running   0          3m30s   10.244.3.6   kind-worker3   <none>           <none>
{{< /output >}}

The `READY` column lists that one container you created for `php`. 
That's it. The `pause` container is *not* counted.

{{% notice note %}}
The `kubectl get pods -n dev` command would have been sufficient above.
The `-o wide` included the `IP` address of the pod and other information.
{{% /notice %}}

{{< step >}}Save the IP address of the pod in a shell variable `demoip`.{{< /step >}}

```bash
demoip=$(kubectl get pods -n dev demo -o jsonpath={.status.podIP}) 
```

{{< step >}}Kill your container's root process to restart it.{{< /step >}}

```bash
kubectl exec -n dev demo -- kill 1
```

{{< step >}}Check if the IP address is the same.{{< /step >}}

```bash
echo original IP $demoip
echo hostname $(kubectl exec -n dev demo -- hostname)
echo
kubectl get pods -n dev demo -o wide
```

Example output:
{{< output >}}
original IP 10.244.3.6
hostname demo
NAME   READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
demo   1/1     Running   1          5m17s   10.244.3.6   kind-worker3   <none>           <none>
{{< /output >}}

{{< step >}}Is the IP address after the restart the same as the original IP?{{< /step >}}

It should be. This behavior is utterly important.

If your container's processes had been the only ones in its `net`, `uts`, and `ipc` namespaces, 
when your container restarted, those Linux namespaces would have been released. 

{{% notice note %}}
The purpose of the `pause` container is to hold on to the same consistent network configuration for the pod,
even when the other containers in the pod start, terminate, or restart.
{{% /notice %}}

{{< step >}}Is the `pause` container listed by `kubectl describe`?{{< /step >}}

```bash
kubectl describe pod -n dev demo
```

The answer should be no. Kubernetes adds and maintains the `pause` container for the lifecycle of the pod.
As a managed component by Kubernetes, the `pause` container is not normally managed directly by people or devops pipelines.

## Success

You have explored the `pause` container and discovered:
- The `pause` container is not counted in the `kubectl get pod` number of containers
- The `pause` container keeps the networking aspects of a pod consistent across app container restarts.
- The `pause` container is not listed by `kubectl describe pod`
