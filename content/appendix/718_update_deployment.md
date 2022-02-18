---
title: "Update Your Deployment"
chapter: false
weight: 718
draft: false
---

## Purpose

In order to investigate pod networking, it can be helpful to have a web app in your pods which can display the IP address of the pod and the IP address of the client. Your mission is update your containerized web app to include such information. While doing so, you have the added benefit of learning about **rollout** history, status, and how changing the `image` part of your container spec in your pod spec in your deployment's template, you initiate the replacement of the old replica set your deployment had managed with a new replica set for the updated version of your app.


{{< mermaid >}}
graph TB
update([image change])-->|updates|deployment(Deployment)
deployment-->|replaces|replicaSetA(Old ReplicaSet<br>image: demo:1.0.0)
replicaSetA-->|terminates|podA[Pod A]
replicaSetA-->|terminates|podB[...]
replicaSetA-->|terminates|podM[Pod M]
deployment-->|with rollout|replicaSetB(New ReplicaSet<br>image: demo:1.1.0)
replicaSetB-->|creates|podN[Pod N]
replicaSetB-->|creates|podO[...]
replicaSetB-->|creates|podZ[Pod Z]

  classDef green fill:#9f6,stroke:#333,stroke-width:4px;
  classDef orange fill:#f96,stroke:#333,stroke-width:4px;
  classDef blue fill:#69f,stroke:#333,stroke-width:4px;
  classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
  class deployment orange;
  class replicaSetA blue;
  class replicaSetB green;
  class podA,podB,podM,podN,podO,podZ yellow;
{{< /mermaid >}}

## Prerequisites

Prior to updating your deployment, you should have one. If you don't already have a `demo` deployment running pods with the container image `demo:1.0.0`, please [Deploy a Fleet of Pods]({{< ref "021_running_at_scale" >}}) first.

## Update Your Deployment

You started by building the simplest of simple [PHP](https://www.php.net/) apps back in the [Containerize Your App]({{< ref "015_build_containers" >}}) lesson. Now it is time to revise that app to include a little more detail.
Beyond that one that just returns the value of the `GREETING` environment variable and the hostname of the server,
this version also includes the pod IP address with the container hosting PHP (`SERVER_ADDR`) and the pod IP address of the client--e.g. the jumpbox pod--as seen from the PHP server (`REMOTE_ADDR`).

{{< step >}}Create another PHP file which includes web pod IP address and client IP address. Call this `index2.php` on the development system.{{< /step >}}
```bash
cat <<EOF >~/environment/index2.php
<?php
  echo getenv("GREETING") . " " . gethostname() . " at " . $_SERVER['SERVER_ADDR'] . " back to " . $_SERVER['REMOTE_ADDR'] . "\n";
?>
EOF
```

{{< step >}}Write a new `Dockerfile` that uses this `index2.php` and copies it to `index.php` in the container image.{{< /step >}}

```bash
cat <<EOF >~/environment/Dockerfile-with-ip
FROM php:8.0.1-apache
COPY index2.php /var/www/html/index.php
ENV GREETING="Hello from"
RUN chmod a+rx index.php
EOF
```

{{< step >}}Build a new container image, also with image name `demo`, but this one with a version number 1.1.0 as the `tag`.{{< /step >}}

```bash
docker build -f Dockerfile-with-ip -t demo:1.1.0 .                                                     
```

{{< output >}}
Sending build context to Docker daemon  69.97MB
Step 1/4 : FROM php:8.0.1-apache
 ---> 6ad14718b8c3
Step 2/4 : COPY index2.php /var/www/html/index.php
 ---> 771f39f1d6dc
Step 3/4 : ENV GREETING="Hello from"
 ---> Running in 776cba627610
Removing intermediate container 776cba627610
 ---> 0b06c2e04081
Step 4/4 : RUN chmod a+rx index.php
 ---> Running in 168483bbcad7
Removing intermediate container 168483bbcad7
 ---> 7fe3f86521a6
Successfully built 7fe3f86521a6
Successfully tagged demo:1.1.0
{{< /output >}}

{{< step >}}Load your new container image `demo:1.1.0` into your Kubernetes cluster.{{< /step >}}

```bash
kind load docker-image demo:1.1.0
```

{{< output >}}
Image: "demo:1.1.0" with ID "sha256:7fe3f86521a6ef04101d9198e671f3ca6ca0b1a360387b674d907db4a47419ba" not yet present on node "kind-worker3", loading...
Image: "demo:1.1.0" with ID "sha256:7fe3f86521a6ef04101d9198e671f3ca6ca0b1a360387b674d907db4a47419ba" not yet present on node "kind-worker", loading...
Image: "demo:1.1.0" with ID "sha256:7fe3f86521a6ef04101d9198e671f3ca6ca0b1a360387b674d907db4a47419ba" not yet present on node "kind-worker2", loading...
Image: "demo:1.1.0" with ID "sha256:7fe3f86521a6ef04101d9198e671f3ca6ca0b1a360387b674d907db4a47419ba" not yet present on node "kind-control-plane", loading...
{{< /output >}}

{{< step >}}Check the current version of the image used in your `demo` deployment.{{< /step >}}

```bash
kubectl get deployment/demo -n dev -o yaml | grep image:
```

{{< output >}}
      - image: demo:1.0.0
{{< /output >}}

{{< step >}}Show the rollout history of this deployment.{{< /step >}}

```bash
kubectl rollout history deployment/demo -n dev
```

{{< output >}}
deployment.apps/demo 
REVISION  CHANGE-CAUSE
1         <none>
{{< /output >}}

{{< step >}}Change the image used by your deployment to the new version `1.1.0`.{{< /step >}}

```bash
kubectl set image deployment/demo -n dev demo=demo:1.1.0
```

{{< output >}}
deployment.apps/demo image updated
{{< /output >}}

{{< step >}}Confirm the rollout was successful.{{< /step >}}

```bash
kubectl rollout status deployment/demo -n dev
```

{{< output >}}
deployment "demo" successfully rolled out
{{< /output >}}

{{< step >}}Check the rollout history again.{{< /step >}}

```bash
kubectl rollout history deployment/demo -n dev                                                           
```

{{< output >}}
deployment.apps/demo 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
{{< /output >}}

You should see your new rollout as a **revision** of the deployment.

{{< step >}}Confirm the image name in your deployment by searching the current manifest.{{< /step >}}

```bash
kubectl get deployment/demo -n dev -o yaml | grep image:
```

{{< output >}}
      - image: demo:1.1.0
{{< /output >}}

## Success

You have successfully:
- Updated a container image with a new version.
- Updated a deployment spec to include a new container image.
- Peformed a rollout of the deployment update and monitored its success.
