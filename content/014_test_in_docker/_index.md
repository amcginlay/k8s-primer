---
title: "Test Your App In Docker"
chapter: false
weight: 014
draft: false
---

## Run your containerized app in Docker

You can now `run` your app as a containerized background process in Docker.
```bash
container_id=$(docker run --detach --rm --publish 8081:80 demo:1.0.0)
```

Here is a quick summary of the options we applied to `docker run`.
- **`--detach`** - This runs your webserver app in the background so you do not lose your command prompt.
- **`--rm`** - The app will run until stopped, then immediately remove any trace of itself.
- **`--publish`** - Containerized apps run in their own [Network namespace](https://en.wikipedia.org/wiki/Linux_namespaces#Network_(net)).
They have their own private IP address and are **internally** free to occupy whatever ports they like.
So whilst port collisions are unlikely inside any individual container it remains a concern on the host which might need to surface many containers from different teams all vying to occupy the same ports.
We solve this issue with [port binding](https://12factor.net/port-binding).
In our case `8081:80` directs the Docker daemon to take all port 8081 requests at the host and forward them to port 80 inside our app. If we wanted to run multiple replicas of our app we could use `8082:80`, `8083:80` and so on.

{{% notice note %}}
The **Docker daemon** is an example of a ***container runtime*** agent; it hosts your containerized app. The `docker` command is an administrative tool which allows you to manage your containers.
{{% /notice %}}

Check that your app is up and running.
```bash
docker ps --filter id=${container_id}
```

The output produced will look something like this which includes information about the port bindings used.
{{< output >}}
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                   NAMES
63ab8c3fb819   demo:1.0.0   "docker-php-entrypoi…"   5 seconds ago   Up 4 seconds   0.0.0.0:8081->80/tcp, :::8081->80/tcp   great_pare
{{< /output >}}

## Test your containerized app

To help galvanize the notion of port bindings we can send test requests to the webserver in two ways.
Firstly, via port 8081 on the the Cloud9 host.
```bash
curl http://localhost:8081
```

And, secondly, via port 80 by `exec`ing onto the container itself.
```bash
docker exec -it ${container_id} curl localhost:80
```

In both cases the response to `gethostname()` inside our app will match the ID of the container we created.

{{< output >}}
63ab8c3fb819
{{< /output >}}

## Stop your containerized app

When you are done running your app, you can stop it using `docker stop` as follows:
```bash
docker stop ${container_id}
```

When the container is stopped, the `docker` command will respond with the container id, such as:

{{< output >}}
4c91176464810c9b796516e1ed48e5361fe98f3ec7107081d5cab080b034a065
{{< /output >}}

{{% notice note %}}
Under normal circumstances, you would typically remove your container using `docker rm ${container_id}` after you stop it. However, because you included the `--rm` option when you *ran* it with `docker run`, the Docker daemon automatically removes the container when it is stopped.
{{% /notice %}}

## Success

- You have successfully `run` a containerized app using `docker run`! 
- You checked the container's process status using `docker ps`.
- Then you tested your app using `curl` which relied on port mapping to the running container.
- Finally, you stopped your container with `docker stop` and in this case it was also removed from the container host (Cloud9).

{{% notice note %}}
Note that the container *image* still exists in your Cloud9 environment, but the compute power for the container runtime was deallocated when the container was removed.
{{% /notice %}}

Now that you have some experience **producing** and **consuming** containers using Docker tooling, you are ready to graduate to Kubernetes! 

<!--

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

minikube start
minikube docker-env > ~/.env
echo "source ~/.env" >> ~/.bashrc
source ~/.env

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

kubectl cluster-info

cat > ~/environment/index.php << EOF
<?php
  echo gethostname() . "\n";
?>
EOF

# test it inside Cloud9

cat > ~/environment/Dockerfile << EOF
FROM php:8.0.1-apache
COPY index.php /var/www/html/
RUN chmod a+rx *.php
EOF

docker build --tag demo:1.0.0 ~/environment/
docker images
container_id=$(docker run --detach --rm demo:1.0.0)
docker ps --filter id=${container_id}

docker exec -it ${container_id} curl localhost:80
docker stop ${container_id}

kubectl run demo --image demo:1.0.0 --image-pull-policy=Never
kubectl exec -it demo -- curl localhost:80
kubectl delete pod demo -->