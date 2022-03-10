---
title: "Multicontainer Pods"
chapter: false
weight: 716
draft: false
---


...

%%%

$ cat 902-demo-duo.yaml 
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demoduo             # a name for your POD
  labels:
    app: demo
    style: duo
    count: "2"
spec:
  containers:                # a pod CAN consist of multiple containers, this one has TWO
  - name: demox              # a name for your first CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
    command:
    - php                    # where to start the container
    args:
    - -S                     # change the server IP + port binding to listen on
    - 127.0.0.1:9081         # use a unique port per container in the pod
  - name: demoy              # second container name
    image: demo:1.0.0        # would usually be different for distinct role
    command:
    - php
    args:
    - -S
    - 127.0.0.1:9082

$ kubectl apply -n dev -f 902-demo-duo.yaml 
pod/demoduo created

$ cat 903-demo-trio.yaml 
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demotrio            # a name for your POD
  labels:
    app: demo
    style: trio
    count: "3"
spec:
  containers:                # a pod CAN consist of multiple containers, this one has THREE
  - name: demox              # a name for your first CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
    command:
    - php                    # where to start the container
    args:
    - -S                     # change the server IP + port binding to listen on
    - 127.0.0.1:9081         # use a unique port per container in the pod
  - name: demoy              # second container name
    image: demo:1.0.0        # would usually be different for distinct role
    command:
    - php
    args:
    - -S
    - 127.0.0.1:9082
  - name: demoz              # third container name
    image: demo:1.0.0        # would usually be different for distinct role
    command:
    - php
    args:
    - -S
    - 127.0.0.1:9083
$ 

$ kubectl apply -n dev -f 903-demo-trio.yaml                                                                           
pod/demotrio created

$ kubectl get pods -n dev
NAME       READY   STATUS    RESTARTS   AGE
demoduo    2/2     Running   0          2m42s
demosolo   1/1     Running   0          4h57m
demotrio   3/3     Running   0          7s

$ kubectl get pods -n dev -o wide
NAME       READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
demoduo    2/2     Running   0          3m34s   10.244.2.2   kind-worker2   <none>           <none>
demosolo   1/1     Running   0          4h58m   10.244.3.2   kind-worker3   <none>           <none>
demotrio   3/3     Running   0          59s     10.244.1.2   kind-worker    <none>           <none>