---
title: "5. Build Containers"
chapter: false
weight: 15
draft: false
---

## Introduction

When learning Kubernetes it is important to set out ones own viewpoint.
Ask youself, are you a **producer** or a **consumer** of apps?
Arguably, in the modern world of DevOps, you would be equally happy in both roles.
Training material often jumps straight into Kubernetes from a heavily operational stance and learners begin by **consuming** container images (e.g. nginx) rather than **producing** them.

We will take a different approach.
We will first **produce** a simple app before containerizing it.
Only once we are happy with the app in its container form will we **consume** it from Kubernetes.
Hopefully this end-to-end approach can help to close some knowledge gaps that might be holding you back from embracing Kubernetes. 

In this exercise you will do the following:
1. Build a *simple* app and test it *locally* in the Cloud9 environment. 
2. Write a `Dockerfile` with instructions for how to ***containerize*** your simple app.
3. Build a *container image* using your `Dockerfile`.

## Build and test a simple app

Get started by building the simplest of simple [PHP](https://www.php.net/) apps.
This one just returns the value of the `GREETING` environment variable (if present) and the hostname of the server.
{{< step >}}Create your `index.php` microapp by typing or copying-and-pasting this into your Cloud9 terminal.{{< /step >}}
```bash
cat <<EOF >~/environment/index.php 
<?php
  echo getenv("GREETING") . " " . gethostname() . "\n";
?>
EOF
```

Before we tackle containers, keep it simple.
{{< step >}}Set the `hostname` to something appropriate and run this app as a webserver from your Cloud9 environment.{{< /step >}}
```bash
sudo hostname Cloud9
GREETING="Hello from" php -S localhost:8080
```

Example output:
{{< output >}}
PHP 7.2.24 Development Server started at Sat Feb 12 18:24:28 2022
Listening on http://localhost:8080
Document root is /home/ec2-user/environment
Press Ctrl-C to quit.
{{< /output >}}


{{% notice note %}}
The above syntax demonstrates the ability to inject environment variables (e.g. `GREETING`) into child processes at the point of creation.
One may `export` variables to achieve the same effect.
{{% /notice %}}

{{% notice note %}}
Port 8080 is neither reserved or firewalled so this is a popular development alternative to port 80.
{{% /notice %}}

This webserver will tie up this first Cloud9 terminal session until its process is stopped.
{{< step >}}Leave the webserver running and select `Window -> New Terminal` to make a second terminal session available.{{< /step >}}

{{< step >}}In the second terminal session, use the `curl` command to send an HTTP GET request to the webserver as follows.{{< /step >}}
```bash
curl http://localhost:8080
```

Example output:
{{< output >}}
Hello from Cloud9
{{< /output >}}

Your output from `curl` will display the value of the `GREETING` environment variable alongside the hostname which you set earlier.

Now you have tested the app works, you can stop the PHP webserver.
{{< step >}}head back the first terminal session and hit `Ctrl+C` to stop the PHP webserver.{{< /step >}}

Containers exploit the use of [namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) so once the app is packaged as a containerized process its hostname will appear to become independent of the underlying host.
It is this behavior which gives rise to containers often being described as lightweight virtual machines.

## Construct the Dockerfile

As mentioned in the [Setup](../011_setup) section, each Cloud9 instance comes with the [Docker](https://www.docker.com/) build/runtime tooling installed and ready to use.
In this section you will build a container image for your app using Docker.
The Docker build tool uses [Dockerfiles](https://docs.docker.com/engine/reference/builder/) for this.
A `Dockerfile` is a text document stored alongside your app code.
It consists of scripted commands used by the Docker build tool to assemble a container image which packages your app and its dependencies. In this sense, your `Dockerfile` is like a recipe for cooking a **container image** and `index.php` is the main ingredient.

The Dockerfile for your app is very simple and is created as follows. 

{{< step >}}Run this in your Cloud9 terminal.{{< /step >}}
```bash
cat <<EOF >~/environment/Dockerfile 
FROM php:8.0.1-apache
COPY index.php /var/www/html/
ENV GREETING="Hello from"
RUN chmod a+rx index.php
EOF
```

Here is a quick summary of your `Dockerfile`.

- **`FROM php:8.0.1-apache`** Our app depends upon a webserver and PHP libraries to work.
As seen, when Cloud9 is the host, these dependencies are drawn from its own set of general-purpose tools located under `/` (i.e the root).
Historically, all apps running on your host would have no option but to trust that the host-provided dependencies are compatible today, and remain compatible tomorrow.
Incompatibilities which arise from shared dependencies are avoidable with containers via [Mount namespaces](https://en.wikipedia.org/wiki/Linux_namespaces#Mount_(mnt)).
Containers can internally switch their perceived location of `/`, away from the shared location to a mountpoint specifically dedicated to the running container, and safely fill it with just the dependencies it needs.
The `FROM` instruction indicates a version-specific archive of files, known as a base layer, which get extracted into the dedicated mountpoint before being customized by the remaining instructions.
As with any image, the base layer may also inject [`ENV`](https://docs.docker.com/engine/reference/builder/#env) variables,  [`EXPOSE`](https://docs.docker.com/engine/reference/builder/#expose) ports and [`RUN`](https://docs.docker.com/engine/reference/builder/#run) processes, etc in the target habitat defined by its namespaces.
Base layers are themselves just published container images which are sourced from registries such as [Dockerhub](https://hub.docker.com/).
In this example `php` is the name of an image and `8.0.1-apache` is the immutable version of that image.
- **`COPY index.php /var/www/html/`** - The default virtual directory for our webserver is at `/var/www/html/`.
The `COPY` instruction takes your app (`index.php`) from the local file system (on your Cloud9 instance in this case) and lays it down as the homepage for your webserver inside the container image.
- **`RUN chmod a+rx index.php`** - A Linux command that will be familiar to you, `chmod` sets the access permissions of our app to ensure it is executable from any context within the container.
- **`CMD`** - this Dockerfile is slightly unusual in that the commonly seen `CMD` (or `ENTRYPOINT`) instruction **does not** appear.
In your case the base layer initiates the webserver so once your (interpreted) PHP app is deposited and configured correctly everything is just ready to go.

You could now, if you wish, keep your app (`index.php`) and its dependencies (`Dockerfile`) in lockstep under source control.
Each container image begins as an empty collection of files with zero configuration.
When processed, each instruction in the Dockerfile will add one **layer** of files and/or configuration to your container image.
Each layer is used to extend or override the preceding layers.

## Build the container image

With the app ready and the Dockerfile composed you can now build a container image for your app. 
{{< step >}}Run this:{{< /step >}}
```bash
docker build --tag demo:1.0.0 ~/environment/
```

Example output:
{{< output >}}
Sending build context to Docker daemon  15.36kB
Step 1/4 : FROM php:8.0.1-apache
8.0.1-apache: Pulling from library/php
a076a628af6f: Pull complete 
... 
0a115ef70c7a: Pull complete 
Digest: sha256:7fd9e31a9580356adefd6ae2ce20b2b98720cc7bc20bfbfaeb8113281e533408
Status: Downloaded newer image for php:8.0.1-apache
 ---> 6ad14718b8c3
Step 2/4 : COPY index.php /var/www/html/
 ---> 119e834cbbcd
Step 3/4 : ENV GREETING="Hello from"
 ---> Running in 8d714367f5b6
Removing intermediate container 8d714367f5b6
 ---> a12324074429
Step 4/4 : RUN chmod a+rx index.php
 ---> Running in 9cbf62fafc84
Removing intermediate container 9cbf62fafc84
 ---> 0374358a8b8e
Successfully built 0374358a8b8e
Successfully tagged demo:1.0.0
{{< /output >}}

{{% notice note %}}
If you do not tag your images they will get auto-tagged as `latest`, over and over again.
Explicitly tagging your images (i.e. `demo:1.0.0`) is considered good practice and ultimately aids your deployments.
{{% /notice %}}

{{< step >}}Once your build is complete, list the images in the local cache.{{< /step >}}
```bash
docker images
```

Example output:
{{< output >}}
REPOSITORY   TAG            IMAGE ID       CREATED         SIZE
demo         1.0.0          3e88e08548ff   2 minutes ago   417MB
php          8.0.1-apache   6ad14718b8c3   10 months ago   417MB
{{< /output >}}

You will observe two images.

- **`php:8.0.1-apache`** is the versioned base layer you referenced in the Dockerfile `FROM` instruction
- **`demo:1.0.0`** is the newly built and tagged container image for your app which sits down upon the `php` base layer

## Container Image and Build Quiz

Please take the following quiz to review your knowledge of building and maintaining container images.

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## Container image recipe

Which type of file did you use as the recipe to build a container image?

> What was the name of the tool you used to follow this recipe?

- [ ] `Makefile`
- [x] `Dockerfile`
- [ ] CloudFormation template (JSON or YAML)
- [ ] CodeBuild `buildspec.yml`

## Container image relationship

Which is the most accurate relationship between the `demo` container image you built and other images?

> How many images were listed after you build your container image?

- [ ] My container image was built from scratch, not based on another image
- [x] My container image was built with a dependency on another existing image
- [ ] Container images are always built relative to the operating system of the host
- [ ] Container images are always built without dependencies on other images

## Image identity

---
shuffle_answers: false
---

What part of the container image is `demo` or `php`?

> What was the output of the `docker images` command?

- [x] repository
- [ ] tag
- [ ] image ID
- [ ] size

## Namespace Management

What part of the container image is `1.0.0` or `8.0.1-apache`?

> What was the output of the `docker images` command?

- [ ] repository
- [x] tag
- [ ] image ID
- [ ] size

{{< /quizdown >}}

## Success

In this exercise you did the following:
1. Built a *simple* app and tested it *locally* in the Cloud9 environment (like eating raw food).
2. Wrote a `Dockerfile` with instructions for how to ***containerize*** your simple app (like writing a recipe).
3. Built a *container image* using your `Dockerfile` using `docker build` (like cooking according to your recipe).

Congratulations. You are a **producer** of containerized apps! If you are not yet a professional developer, nor ever want to be, you can still produce apps as *container images*.
Now you are ready to play the role of **consumer** and ***run*** that containerized app.
