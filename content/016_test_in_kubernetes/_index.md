---
title: "Test Your App in Kubernetes"
chapter: false
weight: 016
draft: false
---

## Load your containerized app image into `kind`

Although you have your containerized app image on your Cloud9 instance, it has not been loaded into your Kubernetes cluster hosted by `kind`. Load it there now.

```bash
kind load docker-image demo:1.0.0
```

{{< output >}}
Image: "demo:1.0.0" with ID "sha256:893d88f41297dc99cf7a294ba7edfedca5d063facf6e89a59a608cc5f7502688" not yet present on node "kind-control-plane", loading...
{{< /output >}}

## Run your containerized app in Kubernetes

You can now `run` your app as a containerized background process in Kubernetes.
```bash
kubectl run demo --image=demo:1.0.0 --port=80 
```

{{< output >}}
pod/demo created
{{< /output >}}

Kubernetes runs containers or collections of containers in pods.
Get a list of the pods running on your Kubernetes cluster:

```bash
kubectl get pods
```

{{< output >}}
NAME   READY   STATUS    RESTARTS   AGE
demo   1/1     Running   0          15m
{{< /output >}}

## Confirm that your app is running

Networking into your Kubernetes-based app with `curl` or other tools requires some ingress plumbing configuration. We'll look at that later. For Now, simply confirm that your app is running in a Kubernetes pod by checking the *logs* for that pod.

```bash
kubectl logs demo
```

The output should look similar to this:

{{< output >}}
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 10.244.0.7. Set the 'ServerName' directive globally to suppress this message
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 10.244.0.7. Set the 'ServerName' directive globally to suppress this message
[Wed Jan 12 14:04:02.633449 2022] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.38 (Debian) PHP/8.0.1 configured -- resuming normal operations
[Wed Jan 12 14:04:02.633508 2022] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'
{{< /output >}}

## Success

You have successfully run your containerized `demo` app in a Kubernetes pod.

<!-- repeated content from prior lesson; pick a good sequence -->

Now you are ready to create your own resources, such as your own Kubernetes:
- namespaces
- configmaps
- pods
- deployments
