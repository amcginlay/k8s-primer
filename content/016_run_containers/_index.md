---
title: "Run Containers"
chapter: false
weight: 16
draft: false
---

As mentioned previously, Docker provides two key features which have brought containers to the masses, the **build tool** which developers use and the **runtime** which your target hosts (e.g. laptops, servers, VMs) need to provide.
Up to this point you have only seen Docker used as a build tool which creates [OCI](https://opencontainers.org/) compliant container images.
These images need a hospitable environment in which to run.
The Docker **runtime**, which runs on your target hosts as a daemon process named `dockerd`, provides that.

## Run your containerized app in Docker

With your container image built you can now `run` your app as a containerized process in the Docker runtime.
{{< step >}}Run your `demo` image as a container.{{< /step >}}
```bash
container_id_one=$(docker run --detach --rm --publish 8081:80 demo:1.0.0)
```

Here is a quick summary of the options we applied to `docker run`.
- **`--detach`** - This runs your webserver app in the background so you do not lose your command prompt.
- **`--rm`** - This mops up resources up after you call `docker stop`, eliminating the need for `docker rm`.
- **`--publish`** - Containerized apps run in their own [Network namespace](https://en.wikipedia.org/wiki/Linux_namespaces#Network_(net)).
They have their own private IP address and are **internally** free to occupy whatever ports they like.
So whilst port collisions are unlikely inside any individual container it remains a concern on the host which might need to provide access to many containers from different teams all vying to occupy the same traditional ports.
We solve this issue with [port binding](https://12factor.net/port-binding).
In our case `8081:80` directs the Docker daemon to take all port 8081 requests at the host and forward them to port 80 inside our app container.
If we wanted to run multiple replicas of our app we could use `8082:80`, `8083:80` and so on.

{{% notice note %}}
The **Docker daemon** is an example of a ***container runtime*** agent; it hosts your containerized app. The `docker` command is an administrative tool which allows you to manage your containers.
{{% /notice %}}

{{< step >}}Check that your app is up and running.{{< /step >}}
```bash
docker ps
```

Example output:
{{< output >}}
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                   NAMES
63ab8c3fb819   demo:1.0.0   "docker-php-entrypoi…"   5 seconds ago   Up 4 seconds   0.0.0.0:8081->80/tcp, :::8081->80/tcp   great_pare
{{< /output >}}

This reveals your running container instance and includes information about the port bindings used.

{{% notice note %}}
Observe the system generated `CONTAINER ID` as this will appear again shortly.
{{% /notice %}}

## Use your containerized app

To help galvanize the notion of port binding, send `curl` requests to your webserver using two approaches.
{{< step >}}Firstly, via port 8081 on the the Cloud9 host.{{< /step >}}
```bash
curl http://localhost:8081
```

{{< step >}}And, secondly, via port 80 by `exec`ing onto the container itself.{{< /step >}}
```bash
docker exec -it ${container_id_one} curl localhost:80
```

Example output:
{{< output >}}
Hello from 63ab8c3fb819
{{< /output >}}

{{< columns >}}
{{< column title="curl http://localhost:8081" >}}
{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph container[Your Container]
  demo((php in<br>demo<br>container))
end
cloud9 -->|curl :8081 -> container :80| demo
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class container yellow;
class demo,curl blue2;
{{< /mermaid >}}
{{< /column >}}
{{< column title="docker exec ... curl localhost:80" >}}
{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph container[Your Container]
  demo((php in<br>demo<br>container))
  curl((curl in<br>demo<br>container))
end
cloud9 -->|docker exec| curl
curl -->|curl :80| demo
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class container yellow;
class demo,curl blue2;
{{< /mermaid >}}
{{< /column >}}
{{< /columns >}}

In both cases the response to `gethostname()` inside your app will match the `CONTAINER ID` we created.
This is the mechanism Docker uses to provide each container instance a unique, isolated identity.
This seems familiar because it is the same technique your used earlier in the course when [learning about the **UTS namespace**]({{< ref "014_linux_namespaces" >}}).

{{% notice note %}}
You referenced `localhost` from the Cloud9 instance.
You also referenced `localhost` from within the `exec`'d container but the context is different.
Why is this?
{{% /notice %}}

{{% expand "Reveal hint" %}}
What does `localhost` resolve to from Cloud9? What does `localhost` resolve to inside your container?
{{% /expand %}}

## Overriding environment variables

The `GREETING` environment variable was set in the Dockerfile which ensures a copy of it will always be available from within your app.
As you launch your app in Docker you have the opportunity to provide override values for these variables.
This technique of separating your code from its config is considered good practice and you will take advantage of it whenever you can.

{{< step >}}Run a second instance of your app with an alternative `GREETING`, taking care to target an unused port on the host.{{< /step >}}
```bash
container_id_two=$(docker run --env "GREETING=Hi from" --detach --rm --publish 8082:80 demo:1.0.0)
```

{{< step >}}Test it, either way, to see the `GREETING` override in effect.{{< /step >}}
```bash
curl http://localhost:8082
docker exec -it ${container_id_two} curl localhost:80
```

{{% notice note %}}
You previously referenced port `80` from within **${container_id_one}**.
You just referenced port `80` from within **${container_id_two}**.
There appears to be no port collision there yet they are both running on the Cloud9 instance.
How is that possible?
{{% /notice %}}

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph container1[Container One]
  demo1((php in<br>demo<br>container))
end
subgraph container2[Container Two]
  demo2((php in<br>demo<br>container))
end
cloud9 -->|curl :8081 -> container :80| demo1
cloud9 -->|curl :8082 -> container :80| demo2
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class container1,container2 yellow;
class demo1,demo2 blue2;
{{< /mermaid >}}

## Containers are ephemeral

{{< step >}}Remind yourself of what is currently running.{{< /step >}}
```bash
docker ps
```

Example output:
{{< output >}}
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                   NAMES
63ab8c3fb819   demo:1.0.0   "docker-php-entrypoi…"   5 seconds ago   Up 4 seconds   0.0.0.0:8081->80/tcp, :::8081->80/tcp   great_pare
2b2b0f8ace50   demo:1.0.0   "docker-php-entrypoi…"   2 minutes ago   Up 2 minutes   0.0.0.0:8082->80/tcp, :::8082->80/tcp   nifty_fermat
{{< /output >}}

You will often hear it said that containers are like lightweight virtual machines.
You will also hear it said that virtual machines are ephemeral, meaning that you must expect that they will eventually fail.
So what happens when a container fails in Docker?
{{< step >}}Run the following to simulate a failure in the second container then check the running processes.{{< /step >}}
```bash
docker exec -it ${container_id_two} kill 1
```
{{< step >}}Check what is running now.{{< /step >}}
```bash
docker ps
```

{{% notice note %}}
As you saw when we introduced **PID namespaces** running containers have process isolation so, when viewed from the perspective of the container, PID 1 always represents the root of its own process tree.
{{% /notice %}}

Docker does provide container [restart policies](https://docs.docker.com/config/containers/start-containers-automatically/#use-a-restart-policy) but this is **not enabled by default**.
Without a restart policy in place your Docker container is no better protected than a regular process and will not be replaced in the event of a failure.

We will refer back to this theme once we start using Kubernetes.

## Within You Without You

> **"We were talking about the space between us all  
And the people who hide themselves behind a wall of illusion  
Never glimpse the truth"** - George Harrison

The isolation afforded to us by containers leads us to speak about processes with some added context.
With regard to a running a container there are two contexts to be aware of.
- **outside** the container - such as when you open a regular shell onto your Linux machine (your root-level host) where the Docker daemon is running
- **inside** the container - such as when you use `docker exec`

As you have seen, containers can achieve isolation from each other but, from the perspective of the root-level host it is just business as usual.
As we learned in an earlier chapter **everything is a file** and, at the root-level host, they are all visible.
You will now test the limits of this visibility.

{{< step >}}First launch a replacement for your second container.{{< /step >}}
```bash
container_id_two=$(docker run --env "GREETING=Hi from" --detach --rm --publish 8082:80 demo:1.0.0)
```

{{< step >}}Jump inside your running containers, and deposit a pair of zero-byte files.{{< /step >}}
```bash
docker exec -it ${container_id_one} touch /TOP_SECRET_INFO_ONE.txt && \
docker exec -it ${container_id_two} touch /TOP_SECRET_INFO_TWO.txt
```

{{< step >}}Now check that `TOP_SECRET_INFO_ONE.txt` exists **inside container one**.{{< /step >}}
```bash
docker exec -it ${container_id_one} ls /
```

Example output:
{{< output >}}
TOP_SECRET_INFO_ONE.txt  boot  etc   lib    media  opt   root  sbin  sys  usr
bin                      dev   home  lib64  mnt    proc  run   srv   tmp  var
{{< /output >}}

{{< step >}}Satisfy yourself that you can do the same for `TOP_SECRET_INFO_TWO.txt` **inside container two**.{{< /step >}}
```bash
^one^two
```

So it appears that your sensitive information is only visible from inside the container it belongs to, and not from inside other containers.
But what about **outside** the containers, at the root-level host?

{{< step >}}Scan the overlay filesystem artifacts within the Cloud9 host.{{< /step >}}
```bash
sudo find /var/lib/docker -type f -name TOP_SECRET_INFO_ONE.txt -o -type f -name TOP_SECRET_INFO_TWO.txt
```

Example output:
{{< output >}}
/var/lib/docker/overlay2/ecc54eeecd42fff...23beb8b88419f2/diff/TOP_SECRET_INFO_ONE.txt
/var/lib/docker/overlay2/ecc54eeecd42fff...23beb8b88419f2/merged/TOP_SECRET_INFO_ONE.txt
/var/lib/docker/overlay2/edd14f116c5a39b...c044a49617d04a/diff/TOP_SECRET_INFO_TWO.txt
/var/lib/docker/overlay2/edd14f116c5a39b...c044a49617d04a/merged/TOP_SECRET_INFO_TWO.txt
{{< /output >}}

Eeeek!
So the contents of `/` inside the container are simply the re-mapped contents of some regular subdirectory under the control of the Docker runtime.
You have just witnessed how Docker exploits **Mount namespaces** to provide the illusion of file-system isolation inside containers.

To close out this section you may also like to `exec` into a container and issue the `mount -t overlay` command to see the `var/lib/docker` references here.

{{% expand "Hint" %}}
`docker exec -it ${container_id_one} mount -t overlay`

`docker exec -it ${container_id_two} mount -t overlay`
{{% /expand %}}
## Gracefully stop a containerized app

{{< step >}}You can request that your remaining container be stopped using `docker stop` as follows.{{< /step >}}
```bash
docker stop ${container_id_one}
docker stop ${container_id_two}
```

When your app instances are stopped, the `docker` command will respond with their container ids, such as: (example output)
{{< output >}}
4c91176464810c9b796516e1ed48e5361fe98f3ec7107081d5cab080b034a065
9ae74dae710ca0f75a72bb50ad318d23dfae847bf733692d46591425b5888cfa
{{< /output >}}


## Success

- You have successfully `run` a containerized app using `docker run`.
- You checked the container's process status using `docker ps`.
- Then you tested your app using `curl` which relied on port mapping to the running container.
- You used `docker exec` which, a bit like `ssh` on a VM, allows you to invoke commands from directly within a runnning container.
- You simulated container failure by terminating PID 1 from inside the container.
- You saw how Docker exploits namespaces to provides the illusion of isolation inside containers.
- Finally, you stopped a container with `docker stop` and in this case it was also removed from the container host (Cloud9).

{{% notice note %}}
Note that the container *image* still exists in your Cloud9 environment, but the compute power for the container runtime was deallocated when each container was removed.
{{% /notice %}}

Now that you have some experience **producing** and **consuming** containers using Docker tooling, you are ready to graduate to Kubernetes! 
