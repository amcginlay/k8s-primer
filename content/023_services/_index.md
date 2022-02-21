---
title: "Services"
chapter: false
weight: 23
draft: false
---

## Purpose

If I take a photo of a Christmas tree decorated with flashing lights, the image I see may depict five red lights and four green ones.
The second photo I take may show an entirely different collection of red and green lights.
To be considered [Cloud Native](https://en.wikipedia.org/wiki/Cloud_native_computing) we must anticipate that our pods will behave much like the lights on the tree.
The only constant is change.

If pods were *more* static we could configure one pod to know the IP address of its target pod and everything would just work.
However, this is not the case.
The moment the target pod goes offline that IP address is of no further use.
Worse still, that target IP address *could* be recycled by a new, unknown pod ... and who knows where that story ends.

If only there were some way in Kubernetes to specify a means of accessing the **current** collection of pods. There is! It is a *separate* Kubernetes kind of object known as a [`Service`](https://kubernetes.io/docs/concepts/services-networking/service/).
As with a pod, a service is usually allocated an IP addresses.
However, unlike pods, service IPs are discoverable via a private DNS server.
So now, for one pod to communicate to another, it only needs to know the [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) of the associated service.
The service will then evenly distribute requests to the currently active pods which match the specified criteria.

{{< mermaid >}}
graph TB
client
service((Service))
demo1([app-abc-rst<br>Pod])
demo2([app-abc-uvw<br>Pod])
demo3([app-abc-xyz<br>Pod])
client -->|service-fqdn| service
service -->|distribute to| demo1 & demo2 & demo3
{{< /mermaid >}}

<!-- inheriting jumpbox and some narrative from - https://github.com/amcginlay/eks-demos/blob/main/doc/12-clusterip-services/README.md -->

## Create a "jumpbox" (nginx)

Services are intended to facilitate dynamic communication channels **between** pods inside your cluster.
To see this in action and troubleshoot problems it often helps to gain peer-level access to your workloads, just as you might do with a regular [jumpbox](https://en.wikipedia.org/wiki/Jump_server) (or bastion host) in the Amazon EC2 world.

{{< step >}}Use the `kubectl create deployment` command to quickly deploy [nginx](https://www.nginx.com) as a singleton pod in the **default** namespace.
This will serve as your "jumpbox".{{< /step >}}

```bash
kubectl create deployment jumpbox --image nginx
sleep 10 && kubectl exec -it deployment/jumpbox -- curl http://localhost:80
```

{{< output >}}
deployment.apps/jumpbox created
...ten seconds later...
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

...yada yada style yada yada...

<body>
<h1>Welcome to nginx!</h1>
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
{{< /output >}}

{{< mermaid >}}
graph LR
cloud9[Cloud 9 in VPC A]
subgraph k8s-cluster[k8s in VPC B]
  jumpbox[jumpbox<br>nginx<br>Pod]
end
cloud9 -->|kubectl exec| jumpbox
jumpbox -->|curl| jumpbox
{{< /mermaid >}}

You have proven that:
- you can connect into your `jumpbox` from Cloud9
- your `jumpbox` can talk to itself

Now for the real fun.

## Built in services

{{< step >}}The built-in DNS deployment we viewed in the previous chapter has an associated service object.
You can inspect it, and its associated pods, as follows.{{< /step >}}

```bash
kubectl -n kube-system get services,pods -l k8s-app=kube-dns -o wide
```

Example output:
{{< output >}}
NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE    SELECTOR
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3h4m   k8s-app=kube-dns

NAME                           READY   STATUS    RESTARTS   AGE    IP           NODE                 NOMINATED NODE   READINESS GATES
pod/coredns-558bd4d5db-6xhk2   1/1     Running   0          3h4m   10.244.0.2   kind-control-plane   <none>           <none>
pod/coredns-558bd4d5db-pr28z   1/1     Running   0          3h4m   10.244.0.4   kind-control-plane   <none>           <none>
{{< /output >}}

The above command reveals two active pod instances belonging to the `coredns`/`kube-dns` deployment and a **ClusterIP** service which can be used to discover them.
Note the `SELECTOR` attribute which governs which targets will be considered for request forwarding.
See [labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) for more details.

{{% notice note %}}
Almost all services you encounter will be of type **ClusterIP** or some sub-type directly descended from it.
Understanding the nature of **ClusterIP** services is crucial to learning Kubernetes.
{{% /notice %}}

{{< step >}}Now pause for a moment to inspect the DNS configuration ([/etc/resolv.conf](https://en.wikipedia.org/wiki/Resolv.conf)) from inside your cluster{{< /step >}}

```bash
kubectl exec deployment/jumpbox -it -- cat /etc/resolv.conf | grep nameserver
```

Example output:
{{< output >}}
nameserver 10.96.0.10
{{< /output >}}

You will observe that the `CLUSTER-IP` of the `kube-dns` service matches the `nameserver` setting from `/etc/resolv.conf` located on your `jumpbox`.
It would be the same result if tried from **any** pod in your cluster.
This tells you that `coredns`/`kube-dns` pods will be acting as the primary DNS server for workloads in your cluster.

{{< step >}}The FQDNs for almost all Kubernetes services will be of the form `<service>.<namespace>.svc.cluster.local`.
Follow this pattern to discover the IP address of the `kube-dns` service in the `kube-system` namespace using the `getent` command as follows.{{< /step >}}

```bash
kubectl exec deployment/jumpbox -it -- getent hosts kube-dns.kube-system.svc.cluster.local
```

Example output:
{{< output >}}
10.96.0.10      kube-dns.kube-system.svc.cluster.local
{{< /output >}}

This reveals how the FQDN of a service gets mapped and resolved to its IP address.

Try repeating the above `getent hosts` command using the **partially** qualified name of `kube-dns.kube-system`.
What do you discover?
Do it once more using just `kube-dns`.
It may not behave as you expect.
Why do you think this is?

{{% notice note %}}
Lightweight container filesystems regularly omit popular DNS tools like `nslookup`, `host` and `dig`. `getent` is less well known but more readily available.
{{% /notice %}}

So now you are familiar with:
- the basic structure of services.
- how communication between ephemeral pods can avoid the use of IP addresses.
- the FQDNs used in Kubernetes which follow a very simple naming convention.

Your own application pods do not currently have a `Service` associated with them.
You will address this in the next section.

## A service for your pods

{{< step >}}Provision a simple **ClusterIP** `Service` for the pods in your existing `demo` deployment.{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/006-demo-service.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: Service
metadata:
  name: demo                 # a name for your service
spec:
  type: ClusterIP            # the default type - included for clarity
  selector:
    app: demo                # selection criteria for your targeted pods
  ports:
  - port: 80                 # simple pass-thru
EOF
```

{{< step >}}Inspect the properties of your `Service`.{{< /step >}}

```bash
kubectl -n dev get services -o wide
```

Example output:
{{< output >}}
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
demo         ClusterIP      10.96.201.228   <none>        80/TCP    14m     app=demo
{{< /output >}}

{{% notice note %}}
As you saw previously, most `Services` have a `CLUSTER-IP` property assigned to it.
You *could* assign it a value in your manifest, however you would normally leave it out and let the Kubernetes service controller automatically assign a value.
Remember, since your services are registered with a private DNS service, you no longer need to care which IP addresses are used.
{{% /notice %}}

{{< step >}}Check to see that the DNS entry for the service has been created.{{< /step >}}

```bash
kubectl exec deployment/jumpbox -it -- getent hosts demo.dev.svc.cluster.local
```

Example output:
{{< output >}}
10.96.201.228   demo.dev.svc.cluster.local
{{< /output >}}

{{< step >}}Check to see you get a response from one of the pods.{{< /step >}}

```bash
kubectl exec deployment/jumpbox -it -- curl http://demo.dev.svc.cluster.local
```

Example output:
{{< output >}}
Hi from demo-658bfb548-8hjkx
{{< /output >}}

This pod-to-pod communication reveals that your `demo` service is able to forward requests from your `jumpbox` to one of your pods in the `demo` deployment.

{{< step >}}To move the narrative along, now check to see you get a response from **ALL** of the pods.{{< /step >}}

```bash
kubectl exec deployment/jumpbox -it -- /bin/bash -c "while true; do curl http://demo.dev.svc.cluster.local:80; sleep 0.25; done"
# ctrl+c to quit loop
```

Example output:
{{< output >}}
Hi from demo-658bfb548-nrhjv
Hi from demo-658bfb548-8hjkx
...
Hi from demo-658bfb548-rw7x6
Hi from demo-658bfb548-rw7x6
{{< /output >}}

Look at the command you sent to your `jumpbox` pod and satisfy yourself that you understand, at a high level, what you asked it to do.

Here is a graphical depiction of your deployment behind your service.

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph k8s-cluster[Kubernetes cluster]
  jumpbox[jumpbox<br>nginx<br>Pod]
  service((demo-service<br>Service))
  subgraph deploy[demo Deployment]
    demo1([demo-abc-rst<br>Pod])
    demo2([demo-abc-uvw<br>Pod])
    demo3([demo-abc-xyz<br>Pod])
  end
end
cloud9 -->|kubectl exec| jumpbox
jumpbox -->|curl| service
service -->|some requests| demo1
service -->|some requests| demo2
service -->|some requests| demo3
{{< /mermaid >}}

## Endpoints

The constancy of the service FQDN (e.g. `demo.dev.svc.cluster.local`) and the `clusterIP` (e.g. `10.96.201.228`) provide the bedrock of predictable communications architectures in many Kubernetes-deployed workloads.
But it is the flexibility and agility of how the `Service` maps to the pods that delivers many beneficial powers.

{{< step >}}Look at the mappings from your `demo` service to the pod `Endpoints` that implement it.{{< /step >}}

```bash
kubectl -n dev get endpoints demo
```

{{< output >}}
NAME   ENDPOINTS                                   AGE
demo   10.244.1.3:80,10.244.2.3:80,10.244.3.3:80   92m
{{< /output >}}

{{< step >}}Quickly scale in your `demo` deployment from 3 replicas to 2 pods using a generator.{{< /step >}}

```bash
kubectl -n dev scale deployment demo --replicas 2 # look mum, no manifest!
```

{{< output >}}
deployment.apps/demo scaled
{{< /output >}}

{{< step >}}Check the results from a compute and networking standpoint.{{< /step >}}

```bash
kubectl -n dev get deployments,endpoints demo
```

{{< output >}}
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo   2/2     2            2           23h

NAME             ENDPOINTS                     AGE
endpoints/demo   10.244.1.3:80,10.244.2.3:80   95m
{{< /output >}}

Now there are only two pods.
Whenever you scale in (or out) your changes will be automatically reflected in the `Endpoints` list.
Just as a `ReplicaSet` manages your pods, a `Service` manages your `Endpoints`.
`Endpoints` are a high-level representation of the underlying proxy tables that handle all the network plumbing between your nodes and your pods.

## Service type summary

You may hear about other types of services that build on `ClusterIP`--`NodePort` and `LoadBalancer`. There is another service type called `ExternalName`. These are beyond the scope of this workshop, however deserve a brief mention. These can be a distraction. They do serve their purposes. However, as you come to better understand `ClusterIP`, you may likely come to believe that it is sufficient for many of your pod-to-pod communications needs.
- `NodePort` builds on `ClusterIP` by mapping any of the node IP addresses along with a specific port number into the underlying `ClusterIP`. This enables inbound communications from outside the cluster.
- `LoadBalancer` builds on `NodePort`--and thus `ClusterIP`--by allocating a load balancer service or mechanism that operates outside the cluster--that maps to the list of node port addresses. This allows external mechanisms to be involved as well in inbound communications from outside the cluster.
- `ExternalName` merely maps a DNS name of a service in the cluster to another DNS name, which could resolve to a "service" (in the general non-Kubernetes sense of the word) outside the cluster.

When you are ready to go beyond the capabilities of Kubernetes services, you may want to consider service meshes, which are quite beyond the scope of this workshop.

Here is a Venn Diagram depiction of these Kubernetes `Service` types.

{{< mermaid >}}
graph TB
subgraph Service
  subgraph ClusterIP
    subgraph NodePort
      subgraph LoadBalancer
      end
    end
  end 
  subgraph ExternalName
  end
end 

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
class ClusterIP yellow;
class NodePort green;
class LoadBalancer blue;
class ExternalName orange;
class Service cyan;
{{< /mermaid >}}

Another way of thinking of it is in terms of class inheritance, with an analogy to Linnaeus taxonomy.

{{< mermaid >}}
graph BT
bird & Service
sparrow & flamingo & ClusterIP & ExternalName
swallow & NodePort 
barnSwallow[barn swallow] & LoadBalancer
barnSwallow -->|is a type of| swallow
swallow -->|is a type of| sparrow
sparrow -->|is a type of| bird
flamingo -->|is a type of| bird
LoadBalancer -->|is a specialized| NodePort
NodePort -->|is a specialized| ClusterIP
ClusterIP -->|is a type of| Service
ExternalName -->|is a type of| Service
{{< /mermaid >}}

## Service Quiz

Please take the following quiz to review your knowledge of `Service` objects.

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## Service definition

---
shuffle_answers: false
---

Which is the best definition of a `Service`?

> Use process of elimination - what can you rule out?

- [ ] a means of maintaining a stable set of pod replicas
- [ ] a means of upgrading pods from one version to the next
- [x] a means of communicating to a set of pods
- [ ] a means of scaling nodes within the cluster

## Pod selection

How do you select the pods behind a `Service`?

> What is the API version of a `Service`?

- [ ] either matchLabels or matchExpressions
- [x] single label key-value match only
- [ ] port number matches
- [ ] provide a list of IP addresses

## Service type

What is the default type of `Service`?

> Which one did you create without specifying a type?

- [ ] `LoadBalancer`
- [ ] `PodMap`
- [x] `ClusterIP`
- [ ] `NodePort`

## Service plumbing

How is the networking for services plumbed out?

> Which components provide mapping to the pods for a service?

- [ ] `kubelet`
- [ ] `containerd`
- [ ] `coredns`
- [x] `kube-proxy`

{{< /quizdown >}}

## Success

In this training adventure, you have:
- Checked out `coredns`/`kube-dns` - an important built-in service.
- Deployed a `Service` for your own pods (`demo`).
- Connected to your service from a `jumpbox` client, using a name in the form `<service>.<namespace>.svc.cluster.local`.
- Looked at the `Endpoints` mapped behind your `Service`.
- Scaled your `Deployment` in and out and noticed that the service's `Endpoints` adjusts accordingly.
