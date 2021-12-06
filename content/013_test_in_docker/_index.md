---
title: "Test Your App In Docker"
chapter: false
weight: 013
draft: false
---

## Run your containerized app

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

<!-- for i in $(docker ps -q); do docker kill $i; done
docker system prune --all --force

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