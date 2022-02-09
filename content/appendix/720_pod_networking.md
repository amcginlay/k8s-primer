---
title: "Pod Networking"
chapter: false
weight: 720
draft: false
---

## Pod Networking

TODO 

lo
veth

kubectl exec -it jumpbox -- curl http://demo-service.dev
Hello from demo-58465f467c-l2f6g at 10.244.1.8 back to 10.244.1.6
DevAccountRole:~/environment $ kubectl exec -it jumpbox -- curl http://demo-service.dev
Hello from demo-58465f467c-f29sm at 10.244.2.9 back to 10.244.1.6

kubectl get pods -n dev
NAME                    READY   STATUS    RESTARTS   AGE
demo-58465f467c-24bcq   1/1     Running   0          34m
demo-58465f467c-dhws5   1/1     Running   0          34m
demo-58465f467c-f29sm   1/1     Running   0          34m
demo-58465f467c-l2f6g   1/1     Running   0          34m
demo-58465f467c-w4n6j   1/1     Running   0          34m
DevAccountRole:~/environment $ kubectl get pods -n dev -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
demo-58465f467c-24bcq   1/1     Running   0          34m   10.244.2.7   kind-worker3   <none>           <none>
demo-58465f467c-dhws5   1/1     Running   0          34m   10.244.2.8   kind-worker3   <none>           <none>
demo-58465f467c-f29sm   1/1     Running   0          34m   10.244.2.9   kind-worker3   <none>           <none>
demo-58465f467c-l2f6g   1/1     Running   0          34m   10.244.1.8   kind-worker2   <none>           <none>
demo-58465f467c-w4n6j   1/1     Running   0          34m   10.244.3.9   kind-worker    <none>           <none>
DevAccountRole:~/environment $ for i in 1 2 3 4 5 6 7 8 9 10; do kubectl exec -it jumpbox -- curl http://demo-service.dev; done
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

