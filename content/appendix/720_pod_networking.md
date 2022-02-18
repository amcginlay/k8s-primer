---
title: "Pod Networking"
chapter: false
weight: 720
draft: false
---

## Purpose

In order to investigate pod networking, it can be helpful to have a web app in your pods which can display the IP address of the pod and the IP address of the client. Your mission is update your containerized web app to include such information. While doing so, you have the added benefit of learning about **rollout** history, status, and how changing the `image` part of your container spec in your pod spec in your deployment's template, you initiate the replacement of the old replica set your deployment had managed with a new replica set for the updated version of your app.

## Prerequisites

Before you dive deeper with pod networking, you have to be in the deep end of the pool. You should have already:
- [Deploy a Fleet of Pods]({{< ref "021_running_at_scale" >}}) first. There you create a `demo` deployment running pods with the container image `demo:1.0.0`. 
- [Set up a Service]({{< ref "023_services" >}}) to provision a `jumpbox` pod for testing, and of course a `Service` (of `ClusterIP` type) to provide access to the pods in your deployment.
- [Update Your Deployment]({{< ref "appendix/718_update_deployment" >}}) to change your deployment to use a new version of your container image `demo:1.1.0`.

## Use Your Service to Get to Your Pods

{{< step >}}Test your updated deployment with the service you created earlier. You *did* the prerequisites, right?{{< /step >}}

```bash
kubectl exec -it jumpbox -- curl http://demo-service.dev
```

{{< output >}}
Hello from demo-58465f467c-l2f6g at 10.244.1.8 back to 10.244.1.6
{{< /output >}}

Note the first IP address. In the example above it was `10.244.1.8`. That is the address of the pod in your deployment that your service dispatched your `curl` request to. The second address after the words "back to" is the client address, `10.244.1.6` in this case. This is the IP address of the `jumpbox` pod you initiated the request from.

{{< step >}}Run another test with the same command, expecting different results.{{< /step >}}

```bash
kubectl exec -it jumpbox -- curl http://demo-service.dev
```

{{< output >}}
Hello from demo-58465f467c-f29sm at 10.244.2.9 back to 10.244.1.6
{{< /output >}}

Repeat this until you get a different IP address and pod name. For example, on the first attempt above the hostname of the pod was `demo-58465f467c-l2f6g` at IP `10.244.1.8`. For the second attempt, the hostname was `demo-58465f467c-f29sm` at IP `10.244.2.9`. You might need to scale your deployment if you had left it with only one replica.

{{% notice tip %}}
Your IP addresses will most likely be different than those depicted here, as will the hostnames of your pods.
{{% /notice %}}

Here is a figure summarizing what you just did.

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph k8s-cluster[Kubernetes cluster]
  jumpbox[jumpbox<br>nginx<br>Pod]
  service((demo-service<br>Service))
  subgraph deploy[demo Deployment]
    demo1([demo-58465f467c-l2f6g<br>Pod<br>IP 10.244.1.8])
    demo2([demo-58465f467c-f29sm<br>Pod<br>IP 10.244.2.9])
  end
end
cloud9 -->|kubectl exec| jumpbox
jumpbox -->|two curl requests| service
service -->|first request| demo1
service -->|second request| demo2
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class jumpbox lavender;
class service cyan;
class deploy orange;
class demo1,demo2 yellow;
{{< /mermaid >}}

## Check Which Pods You Have

{{< step >}}Get a list of the pods in your deployment.{{< /step >}}

```bash
kubectl get pods -n dev
```

{{< output >}}
NAME                    READY   STATUS    RESTARTS   AGE
demo-58465f467c-24bcq   1/1     Running   0          34m
demo-58465f467c-dhws5   1/1     Running   0          34m
demo-58465f467c-f29sm   1/1     Running   0          34m
demo-58465f467c-l2f6g   1/1     Running   0          34m
demo-58465f467c-w4n6j   1/1     Running   0          34m
{{< /output >}}

The default columns for `get pods` does not show the pod IP addresses nor the node names the pods are hosted on.
This is intentional. One of the great benefits of Kubernetes is yielding control to it and letting it orchestrate the use of the infrastructure you have provisioned in your cluster.

However, out of curiosity, for troubleshooting readiness, you may wish to delve deeper into some of these details.

{{< step >}}Use the `-o wide` option this time to show the pod IP and node columns.{{< /step >}}

```bash
kubectl get pods -n dev -o wide
```

{{< output >}}
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
demo-58465f467c-24bcq   1/1     Running   0          34m   10.244.2.7   kind-worker3   <none>           <none>
demo-58465f467c-dhws5   1/1     Running   0          34m   10.244.2.8   kind-worker3   <none>           <none>
demo-58465f467c-f29sm   1/1     Running   0          34m   10.244.2.9   kind-worker3   <none>           <none>
demo-58465f467c-l2f6g   1/1     Running   0          34m   10.244.1.8   kind-worker2   <none>           <none>
demo-58465f467c-w4n6j   1/1     Running   0          34m   10.244.3.9   kind-worker    <none>           <none>
{{< /output >}}

{{% notice note %}}
Each **pod** has a unique IP address.
There may be zero, one, or more pods from your deployment in any given node.
{{% /notice %}}

{{< step >}}Dispatch a number of requests to your service to see the distribution across your pods.{{< /step >}}

```bash
for i in 1 2 3 4 5 6 7 8 9 10; do kubectl exec -it jumpbox -- curl http://demo-service.dev; done
```

{{< output >}}
Hello from demo-58465f467c-dhws5 at 10.244.2.8 back to 10.244.1.6
Hello from demo-58465f467c-24bcq at 10.244.2.7 back to 10.244.1.6
Hello from demo-58465f467c-w4n6j at 10.244.3.9 back to 10.244.1.6
Hello from demo-58465f467c-l2f6g at 10.244.1.8 back to 10.244.1.6
Hello from demo-58465f467c-f29sm at 10.244.2.9 back to 10.244.1.6
Hello from demo-58465f467c-f29sm at 10.244.2.9 back to 10.244.1.6
Hello from demo-58465f467c-l2f6g at 10.244.1.8 back to 10.244.1.6
Hello from demo-58465f467c-f29sm at 10.244.2.9 back to 10.244.1.6
Hello from demo-58465f467c-w4n6j at 10.244.3.9 back to 10.244.1.6
Hello from demo-58465f467c-w4n6j at 10.244.3.9 back to 10.244.1.6
{{< /output >}}

{{% notice tip %}}
You could do a `while` loop as you had in earlier testing. The `for` construct in the shell was used here to perform a fixed number of requests.
{{% /notice %}}

Your sequence of requests are dispatched by the service to the pods that service **selects**. 
- You could use a `Deployment` to provision those pods behind the servce, as was done here.
- Alternatively, you could use a `DaemonSet` behind a service when each node needs one instance of the pod.
- Subsequent requests from the same client were not delivered to the same pod, as no affinity was used in this service.

Questions for now:
- How distributed are the requests?
- What will happen when you scale your deployment?

Questions to think about and answer later:
- Is your app stateless?
- How does your app maintain context between subsequent web requests (e.g. API calls)?

## Success

In this lesson, you have:
- Confirmed that a service dispatches requests to pods it matches.
- Identified that those pods each have unique IP addresses.
- Noted that pod IP addresses are different than the service IP.
- Noted that pod IP addresses are distinct from the node IPs.
