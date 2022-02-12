---
title: "Run Containers"
chapter: false
weight: 16
draft: false
---

## Run your containerized app in Docker

You can now `run` your app as a containerized background process in Docker.
```bash
container_id=$(docker run --detach --rm --publish 8081:80 demo:1.0.0)
```

Here is a quick summary of the options we applied to `docker run`.
- **`--detach`** - This runs your webserver app in the background so you do not lose your command prompt.
- **`--rm`** - Eliminates the need to call `docker rm` after calling `docker stop`.
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

In both cases the response to `gethostname()` inside our app will match the short-form ID of the container we created.

{{< output >}}
Hello from 63ab8c3fb819
{{< /output >}}

{{% notice note %}}
You reference `localhost` from the Cloud9 instance.
You also reference `localhost` from within the `exec`'d container but the context is different.
Why is this?
{{% /notice %}}

## Overriding environment variables

The `GREETING` environment variable was set in the Dockerfile which ensures it will always be available from within your app.
As you launch your app in Docker you have the opportunity to provide override values for these variables.
This technique of separating your code from its config is considered good practice and you will take advantage of it whenever you can.

Run a second instance of your app with an alternative `GREETING`, taking care to target an unused port on the host.
```bash
container2_id=$(docker run --env "GREETING=Hi from" --detach --rm --publish 8082:80 demo:1.0.0)
```

Test it to see the `GREETING` override in effect.
```bash
curl http://localhost:8082
# ... or ...
docker exec -it ${container2_id} curl localhost:80
```

{{% notice note %}}
You previously referenced port `80` from within **${container_id}**.
You just referenced port `80` from within **${container2_id}**.
There appears to be no port collision there yet they are both running on the Cloud9 instance.
How is that possible?
{{% /notice %}}

{{< mermaid >}}
graph TB
container-->Cloud9
container2-->Cloud9 
{{< /mermaid >}}

## Containers are ephemeral

Remind yourself of what is currently running.
```bash
docker ps --filter id=${container_id} --filter id=${container2_id}
```

The output shown below is similar to what you will see.
{{< output >}}
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                   NAMES
63ab8c3fb819   demo:1.0.0   "docker-php-entrypoi…"   5 seconds ago   Up 4 seconds   0.0.0.0:8081->80/tcp, :::8081->80/tcp   great_pare
2b2b0f8ace50   demo:1.0.0   "docker-php-entrypoi…"   2 minutes ago   Up 2 minutes   0.0.0.0:8082->80/tcp, :::8082->80/tcp   nifty_fermat
{{< /output >}}

You will often hear it said that containers are like lightweight virtual machines.
You will also hear it said that virtual machines are ephemeral, meaning that you must expect that they will eventually fail.
So what happens when a container fails in Docker?
Run the following to simulate a failure in the second container then check the running processes.
```bash
docker exec -it ${container2_id} kill 1
```
```bash
docker ps --filter id=${container_id} --filter id=${container2_id}
```

{{% notice note %}}
Running containers have their own process namespaces so PID 1 always represents the root of its own process tree.
{{% /notice %}}

Docker provides container [restart policies](https://docs.docker.com/config/containers/start-containers-automatically/#use-a-restart-policy) but this is **not enabled by default**.
Without a restart policy in place your Docker container is no better protected than a regular process and will not be replaced in the event of a failure.

We will refer back to this theme once we start using Kubernetes.

## Gracefully stop a containerized app

You can request that your remaining container be stopped using `docker stop` as follows.
```bash
docker stop ${container_id}
```

When your app instances are stopped, the `docker` command will respond with their container ids, such as:

{{< output >}}
4c91176464810c9b796516e1ed48e5361fe98f3ec7107081d5cab080b034a065
{{< /output >}}

## Success

- You have successfully `run` a containerized app using `docker run`.
- You checked the container's process status using `docker ps`.
- Then you tested your app using `curl` which relied on port mapping to the running container.
- You used `docker exec` which, a bit like `ssh` on a VM, allows you to invoke commands from directly within a runnning container.
- You simulated container failure by terminating PID 1 from inside the container.
- Finally, you stopped a container with `docker stop` and in this case it was also removed from the container host (Cloud9).

{{% notice note %}}
Note that the container *image* still exists in your Cloud9 environment, but the compute power for the container runtime was deallocated when the container was removed.
{{% /notice %}}

Now that you have some experience **producing** and **consuming** containers using Docker tooling, you are ready to graduate to Kubernetes! 
