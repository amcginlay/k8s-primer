---
title: "Why Kubernetes?"
chapter: false
weight: 700
draft: false
---

## Why Containers?

Compute containers, or simply *containers* as we call them, 
provide several protective isolation mechanisms
to keep your application workloads running smoothly.

Containers provide these isolation features in a lightweight fashion based on 
[Linux namespaces]({{< ref "014_linux_namespaces" >}}).
Containers can also use Linux control groups (cgroups) to 
enforce memory, CPU, and I/O limits as well as provide accounting.

By design, containers require far less overhead compared with virtual machines.
Certainly, containers have all those advantages plus the virtual machine benefits 
when compared with running each application on separate physical servers.

Use of microservices and many small app components further enhances such benefits.

## Why Kubernetes?

Imagine that you have a boat. 
Maybe you fish, harvest kelp, ferry passengers, or carry cargo. 
Perhaps you provide police, military, scientific, or research services.
Or your boat could just be for recreation.
If your boat is small, perhaps you can sail it all by yourself;
if your boat is large, maybe you have help to manage it.

Managing containers on a single Docker container host is like running a single ship.

But what do you do when you have more than one container host server?
For a small group of hosts, you can make do with Docker.
But for larger and larger groups of container hosts, Docker becomes inadequate.

Kubernetes is not like Docker. 
The purpose of Kubernetes is to manage containerized workloads on a *cluster* of *many* container hosts.
You would use Kubernetes if you want to orchestrate a medium to large to massive scale container hosting infrastructure.
Kubernetes is open-source and cloud-native, with a global community, and multi-vendor support.

## Medium to Large to Massive Scale

How many container hosts do you need in your Kubernetes cluster?
The real systems engineering answers require knowledge of your workloads and management strategy.
Instead of delving into that topic, let's stay focused on broad possibilities.
You could use Kubernetes to manage a single container host, or two, three, ten, hundreds, or thousands of container hosts.
It may also be possibly that you choose to have more than one Kubernetes cluster to orchestrate all of your computing workloads.
Remember always, that you are ultimately in control. 
Well, you and your DevOps pipeline automations.

Who is the cast of characters involved with Kubernetes?

1. Each container host server is called a ***node***. 
2. Each autonomous Kubernetes collective is called a ***cluster***.
3. You can manage one or more Kubernetes clusters. Who is ultimately in control? ***You***.

We can describe these three levels of Kubernetes container orchestration through a metaphor.

Let's go back to the boat metaphor.
Kubernetes, like Docker, is steeped in boating, shipping, and other nautical terminology.

1. **Ship** - a single boat. In Kubernetes, each *node* is like a single *boat*.
2. **Flotilla** - a small or medium sized group of boats. A flotilla is also called a **convoy** or **squadron**. In Kubernetes, a *cluster* is like a *flotilla*.
3. **Fleet** - a large group of ships composed of many flotillas, convoys, or squadrons. In Kubernetes, *you* can manage one or many clusters, depending on your needs.

In military or commercial operations, we could refer to the roles and responsibilities for who commands each level of this hierarchy.

1. **Captain** - the commander of a single ship. In Kubernetes, the `kubelet` is the software on each node that manages the node.
2. **Commodore** - the commander of a flotilla. In Kubernetes, the `api-server` is the software in the control plane of the cluster that manages each of the nodes to accomplish the unified mission.
3. **Admiral** - the commander of a fleet. In Kubernetes, you and your devops teams and automations are in control of your Kubernetes clusters to provide for and serve your business operations.

## The Captain of a Ship Metaphor

> "It matters not how great the K8s,  
     How charged with manifests the load,  
   I am the manager of my pods,  
     I am the captain of my node."  
   -- Brad Werner ("kubelet"), with apologies to William Ernest Henley ("Invictus") 
