---
title: "Set Up a Service"
chapter: false
weight: 23
draft: false
---

## Purpose

Deployments, daemonsets, and rag tag collections of bare pods provide a way for you to provision *compute* power into your Kubernetes clusters. For example, a common definition of a `Deployment` is a collection of pods and a means of provisioning them. But how do we reach those pods individually or collectively via the network? Each pod will have its own IP address. You would need to provide a specific IP address to get to any one pod. 

But pods are ephemeral. Node maintenance, container image upgrades, pod configuration updates, and scale out and scale in can evict and create pods as needed. Therefore, you cannot count on the same set of pods being consistent over time. Thus, the set of IP addresses and corresponding DNS names assigned to pods in a deployment, daemonset, or even bare pod will change over time.

If only there were some way in Kubernetes to specify a means of ***accessing*** a collection of pods over the network. There is! It is a *separate* Kubernetes kind of object: [`Service`](https://kubernetes.io/docs/concepts/services-networking/service/).

## Find the API Version for Service

Like the `Pod` kind, `Service` objects are a fundamental resource in Kubernetes.

{{< step >}}Find the API version for the `Service` kind.{{< /step >}}

```bash
kubectl api-resources | head -1; kubectl api-resources | grep svc
```

{{< output >}}
NAME        SHORTNAMES   APIVERSION    NAMESPACED   KIND
services    svc          v1            true         Service
{{< /output >}}

- Use `apiVersion: v1` like for pods, different than daemonsets and deployments.
- Use `kind: Service` when declaring services.

## Select Pods By Label

How do we select the pods that should be included in a `Service`? By matching labels the pods have. See [labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) for details.

However, being a `v1` resource type, `Service` objects use a *simple* style of label matching for their `selector`, like the legacy `ReplicationController`. 

{{% notice note %}}
Note that the `apps/v1` kinds of resources (e.g. `ReplicaSet`, `Deployment`, and `DaemonSet`) can use either `matchLabels` like the simple style or `matchExpressions` for set operations (i.e. members of a list). These more advanced modern choices are referred to as [Label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors).
{{% /notice %}}

{{< step >}}List the services in your cluster.{{< /step >}}

```bash
kubectl get service -A
```

{{< output >}}
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  8h
kube-system   kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   8h
{{< /output >}}

{{< step >}}Look at the selector used in `kube-dns`{{< /step >}}


<!-- skip this shenanigans, just grep for the relevant bit: kubectl get service kube-dns -n kube-system -o yaml >~/environment/112-kube-dns-svc.yaml -->

```bash
kubectl get service kube-dns -n kube-system -o yaml | grep selector -A 1
```

{{< output >}}
  selector:
    k8s-app: kube-dns
{{< /output >}}

The simple selector used in `Service` objects contains a single `key: value` pair to match a label in the pods you want to reach.

## Write a ClusterIP Manifest

{{< step >}}Write a simple `Service` manifest.{{< /step >}}

```bash
cat <<EOF >~/environment/113-demo-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo
  ports:
  - port: 80
EOF
```

{{% notice note %}}
This service requires that you have a `dev` namespace and a `demo` deployment provisioned into it.
{{% /notice %}}

## Provision Your Service

{{< step >}}Provision the `Service` into the `dev` namespace.{{< /step >}}

```bash
kubectl apply -f ~/environment/113-demo-service.yaml -n dev
```

{{< output >}}
service/demo-service created
{{< /output >}}

Congratulations, you have deployed a service to Kubernetes.

## Inspect Your Service

{{< step >}}Get the properties of your `Service`.{{< /step >}}

```bash
kubectl get service demo-service -n dev
```

{{< output >}}
NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
demo-service   ClusterIP   10.96.143.4   <none>        80/TCP    12m
{{< /output >}}

{{< step >}}Show the YAML form of your `Service` manifest.{{< /step >}}

```bash
kubectl get service demo-service -n dev -o yaml | tee ~/environment/114-demo-service-result.yaml
```

{{< output >}}
apiVersion: v1
kind: Service
metadata:
  name: demo-service
  namespace: dev
  ... annotations, creationTimestamp, resourceVersion, uid ...
spec:
  clusterIP: 10.96.143.4
  clusterIPs:
  - 10.96.143.4
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: demo
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
{{< /output >}}

TODO - inherit from - https://github.com/amcginlay/eks-demos/blob/main/doc/12-clusterip-services/README.md


## Test Your Service

## Relationship of ClusterIP to other Service Types

{{< mermaid >}}
graph TB
subgraph Service-manifest[Service manifest]
  apiVersion(apiVersion: v1)
  kind(kind: Service)
  subgraph Service-metadata[Service metadata]
    name(name)
    serviceLabels[labels]
  end
  subgraph Service-spec[Service spec]
    clusterIP
    selector
    subgraph ports
      port1
      port2
      port3
    end
  end
end 

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class Service-manifest orange;
class Service-metadata blue;
class Service-spec green;
class ports yellow;
{{< /mermaid >}}

## Service Quiz

## Success
