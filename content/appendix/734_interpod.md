---
title: "Interpod"
chapter: false
weight: 734
draft: false
---

## Purpose

You hopfully have a solid understanding of communications between processes and between containers *within* a pod; those are examples of *intrapod* communications. 

It should also be stated that containers in one of your pods could communicate with other containers in the same by using the **pod IP address** instead of a *loopback address*. It is really up to the developers, or in some cases the operations people if the developers have yielded such control over the configuration of the app components.

However, we need to go further. Now it is time to address the topic of *interpod* networking.

In this lesson, you will:
1. look at scenarios to use pod IP addresses within the pod.
2. expand beyond the one pod environment to communicate between pods.

## Talking Within and Between Pods - Via Pod Addresses

Can use pod IP for communication.

```bash
demopod=$(kubectl get pods -l app=demo -n dev -o jsonpath={.items[0].metadata.name})                     
echo Going to use $demopod
```

{{< output >}}
Going to use demo-58465f467c-24bcq
{{< /output >}}

```bash
kubectl exec $demopod -n dev  -- grep -v ip6 /etc/hosts                                                  
```

{{< output >}}
# Kubernetes-managed hosts file.
127.0.0.1       localhost
10.244.2.7      demo-58465f467c-24bcq
{{< /output >}}

## Success

In this lesson, you:
- Looked at scenarios to use pod IP addresses within the pod.
- Expanded beyond the one pod environment to communicate between pods.

