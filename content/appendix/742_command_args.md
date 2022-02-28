---
title: "Command and Args"
chapter: false
weight: 742
draft: false
---

## Purpose

Kubernetes `command` and `args` declarations are essential to specify what runs in your containers.
When you do not specify these values in a pod spec within your deployments, these values come from your image.
You can bake default values into your container images using `Dockerfile` directives: `ENTRYPOINT` and `CMD`.
These are often misunderstood.
Let's dispell any mysteries therein.

## UNIX 

Most operating systems provide you with a way to run software in a process via:
- graphical user interface (GUI)
- command line interface (CLI)
- background processing
- scheduled jobs

Shells such as `bash` and `sh` allow you to run an executable binary program or a script by providing:
- the **name** of the app, or a more qualified path to it
- **arguments** to the app, which are also called **options** or **parameters**

{{< step >}}Get a list of pods running on your system.{{< /step >}}
```bash
kubectl get pods
```

Example output:
{{< output >}}
NAME         READY   STATUS    RESTARTS   AGE
catscratch   1/1     Running   3          6h3m
kittycat     1/1     Running   0          6h39m
netcat       1/1     Running   4          5h19m
{{< /output >}}

In this case:
- `kubectl` is the name of the command you ran
- `get` and `pods` are the two arguments to that command

Inevitably, whether you're using a shell, Docker, Kubernetes, or other kind of runtime, all apps and services are launched using `fork()` or `clone()` to make a new process, followed by [`execve()` (or its derivatives)](https://man7.org/linux/man-pages/man2/execve.2.html) to run your executable code. Whether you're running a shell, a Python interpreter, a JavaScript runtime, or a binary of your own, `execve()` needs three pieces of information:
- `pathname` - the file system path to your executable or the executble of the interpreter you want to run your code in
- `argv` - a vector (i.e. list) of ***arguments*** to pass as parameters or options to the process
- `envp` - a pointer to a list of ***environment variables*** to pass/inherit into the process

{{< step >}}Check where the `kubectl` executable is.{{< /step >}}
```bash
which kubectl
```

{{< output >}}
/usr/local/bin/kubectl
{{< /output >}}

So, what the shell actually does in the `kubectl get pods` scenario is:
1. `fork()` a new child process.
2. Wire up standard input, output, and error channels. Because you did no redirection or pipes in the `kubectl get pods`, that essentially keeps your shell's terminal session channels the same for the child process.
3. Looks up the path to `kubectl` like you did with `which kubectl`
4. `execve( "/usr/local/bin/kubectl", ["kubectl", "get","pods"], your_current_environment_variables )`

That's it. That's what shells do in a nutshell! 

Now, let's graduate to containers.

## Container Images

You can bake values for the command, arguments, and environment variables into your container images.
Your [`Dockerfile`](https://docs.docker.com/engine/reference/builder/) allows you to specify default values for your containers by providing the following aspects of your container image:
- [`ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#entrypoint) -- lets you indicate the program to run at container startup. You can give `ENTRYPOINT` a path to an executable file within the image, or the name of a command within a shell's path context. In addition, you can provide arguments/parameters after the command in the `ENTRYPOINT` declaration. There are two forms:
    - ***exec*** form: `ENTRYPOINT ["executable", "arg1", "arg2", ...]` *(recommended)*
    - ***shell*** form: `ENTRYPOINT command arg1 arg2` *(in which command is in your shell's `PATH` list)*
- [`CMD`](https://docs.docker.com/engine/reference/builder/#cmd) -- lets you provide default arguments to the command or executable your container uses at startup. However, many people have gotten into the habit of using `CMD` *without* `ENTRYPOINT`, which is why the `Dockerfile` docs say there are three forms. We intentionally list them here in a different order than those docs; this is to stress that the primary intention of `CMD` to to provide arguments, *not* the command itself. The reason this is called `CMD` and many people use this *assuming* there will always be a shell. Though common, this is not a wise assumption.
    - **args** form: `CMD ["arg1", "arg2"]` *(as default args/params to `ENTRYPOINT`)*
    - **exec** form: `CMD ["executable", "arg1", "arg2"]` *(instead of using `ENTRYPOINT`)*
    - **shell** form: `CMD command arg1 arg2`
- [`ENV`](https://docs.docker.com/engine/reference/builder/#env) -- allows you to provide environment variables for the container made from this image. You give key-value pairs with an equal sign between them. You can set multiple variables in each `ENV` instruction. Be careful to follow [quoting and backslash escape rules](https://docs.docker.com/engine/reference/builder/#env). For example:
    - `ENV GREETING="hello k8s" FAREWELL=good\ bye`

## Docker Overrides 

Although our focus is Kubernetes, it can be useful to identify behaviors of `docker run` with respect to the container images being used.
Kubernetes follows those traditions for the most part, as we'll review shortly.

You can provide command line options to `docker run` to override the default values in the image. 
Here are some cherry-picked quotes from the `Dockerfile` reference which describe the nature of the default values and such overrides.
- "An `ENTRYPOINT` allows you to configure a container that will run as an executable." and "Command line arguments to `docker run <image>` will be appended after all elements in an ***exec*** form `ENTRYPOINT`, and will override all elements specified using `CMD`. This allows arguments to be passed to the entry point, i.e., `docker run <image> -d` will pass the `-d` argument to the entry point. You can override the `ENTRYPOINT` instruction using the `docker run --entrypoint` flag." (quotes from [Docker build reference `ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#entrypoint))
- "The main purpose of a `CMD` is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an `ENTRYPOINT` instruction as well." and "If you would like your container to run the same executable every time, then you should consider using `ENTRYPOINT` in combination with `CMD`. See `ENTRYPOINT`. If the user specifies arguments to `docker run` then they will override the default specified in `CMD`." (quote from [Docker build reference `CMD`](https://docs.docker.com/engine/reference/builder/#cmd))
- "The environment variables set using `ENV` will persist when a container is run from the resulting image. You can view the values using `docker inspect`, and change them using `docker run --env <key>=<value>`." (quote from [Docker build reference: `ENV`](https://docs.docker.com/engine/reference/builder/#env))


## Kubernetes Pod and Container Specs

Kubernetes can also either use the `ENTRYPOINT`, `CMD`, and `ENV` values from container images verbatim, or override them.
It's up to you.
As we just reviewed such behaviors of `docker run`, let's first look at `kubectl run` behavior 
and *then* focus on how you can override these with your Kubernetes manifests.

Similar to `docker run`, [`kubectl run`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#run)
also provides you with mechanisms for overriding the default command, argument, and environment variable values.

{{< step >}}Show the `kubectl run` usage information.{{< /step >}}
```bash
kubectl run --help | grep Usage: -A 1
```

{{< output >}}
Usage:
  kubectl run NAME --image=image [--env="key=value"] [--port=port] [--dry-run=server|client] [--overrides=inline-json] [--command] -- [COMMAND] [args...] [options]
{{< /output >}}

Here's a quick summary:
- with `--command` -- you give both a command and arguments after the double-hyphen: e.g. `--command -- ls /bin` (overrides `ENTRYPOINT` and `CMD`)
- without `--command` -- you give only arguments after the double-hyphen: e.g. `-- /bin` (assuming some viable `ENTRYPOINT` in image, overrides `CMD`)
- `--env` -- overrides image-provided environment variable values (or adds), such as: `--env="GREETING=hola --env="FAREWELL=adios"`

{{< step >}}Generate a manifest which shows the usage of the `--command` option to `kubectl run`.{{< /step >}}
```bash
kubectl run mypod --image=myimage --dry-run=client -o yaml --command -- ls /bin
```

{{< output >}}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mypod
  name: mypod
spec:
  containers:
  - command:
    - ls
    - /bin
    image: myimage
    name: mypod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
{{< /output >}}

{{< step >}}Now generate a manifest which shows the usage of `kubectl run` *without* the `--command` option.{{< /step >}}
```bash
kubectl run mypod --image=myimage --dry-run=client -o yaml -- /bin
```

{{< output >}}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mypod
  name: mypod
spec:
  containers:
  - args:
    - /bin
    image: myimage
    name: mypod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
{{< /output >}}


With Kubernetes manifests, you can [Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/) in a devops-oriented fashion.

In your pod spec, you can provide the following for each of your containers:
- `command:` -- overrides image's `ENTRYPOINT`
- `args:` -- overrides image's `CMD`
- `env:` -- adds to image's `ENV` list, wherein the same key will cause override

If you don't include these in your manifests, Kubernetes uses the default values from your image--which you had specified in the `Dockerfile`.

{{% notice note %}}
Kubernetes sends the container the combination of the command and arguments, whether those values come from the image or your manifest.
{{% /notice %}}
