---
title: "Scratch"
disableToc: true
---

## A place to try out Hugo features ...

```bash
dummy change
```

{{< step >}}
Get your coffee. Always the first step.
{{< /step >}}

Then something else happens.

{{< step >}}
Download the `k8s-primer`.
{{< /step >}}

{{< step >}}
Then launch Hugo.
{{< /step >}}

Now we have to test a longer step.

{{< step >}}
This is the time for all good steps to go in sequence, be properly indented, and maybe even contain a little markdown so we can indicate a `Pod` kind of thing, or a `spec.replicas` sort of notation. No, that was not the only sentence in the paragraph. It just keeps on going.
{{< /step >}}

{{< step >}}And this is yet another step. Remember to load your images. We should build another one besides `demo:1.0.0` to show them how the `Deployment` handles a change of image. That would be a good exercise. I think a separate exercise after the current deployment one.{{< /step >}}

{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}

```bash
PIP=$(kubectl get pods -n dev -o jsonpath={.items[0].status.podIP})
kubectl exec -it jumpbox -- curl http://$(echo $PIP | sed -e 's/\./-/g').dev.pod
```

{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}
{{< step >}}Another step.{{< /step >}}

<!-- but following from the tabs+tab hugo shortcodes, we now have columns+column... for side-by-side display -->
{{< columns >}}
  {{< column title="shell1" >}}
{{< step >}}And this is yet another step. Remember to load your images. We should build another one besides `demo:1.0.0` to show them how the `Deployment` handles a change of image. That would be a good exercise. I think a separate exercise after the current deployment one.{{< /step >}}
{{< output >}}
This is the time for all good steps to go in sequence, be properly indented, and maybe even contain a little markdown so we can indicate a `Pod` kind of thing, or a `spec.replicas` sort of notation. No, that was not the only sentence in the paragraph. It just keeps on going.
{{< /output >}}
  {{< /column >}}
  {{< column title="shell2" >}}
{{< step >}}
This is the time for all good steps to go in sequence, be properly indented, and maybe even contain a little markdown so we can indicate a `Pod` kind of thing, or a `spec.replicas` sort of notation. No, that was not the only sentence in the paragraph. It just keeps on going.
{{< /step >}}
{{< output >}}And this is yet another step. Remember to load your images. We should build another one besides `demo:1.0.0` to show them how the `Deployment` handles a change of image. That would be a good exercise. I think a separate exercise after the current deployment one.{{< /output >}}
  {{< /column >}}
{{< /columns >}}

### The Linux UTS namespace

Because each Linux VM starts with a single UTS namespace **shared** across all running processes you will likely remain oblivious to its existence.
As an example, consider the [hostname](https://en.wikipedia.org/wiki/Hostname) command.
It consistently responds with a single (global) value across all shells and changes are observable from all shells.

{{% notice note %}}
UTS stands for UNIX Time-Sharing.
It is not important why this name was chosen and most would agree it is not the most descriptive name.
If it helps, you can think of it as the **host namespace**.
{{% /notice %}}

{{< step >}}Try this out for yourself by creating a pair of terminal windows (**shell one** and **shell two**) inside Cloud9.{{< /step >}}
Your screen layout may look something like this.

![shell-one-two](/images/process/shell-one-two.png)

{{< step >}}Now execute the following in **both** shells.{{< /step >}}
```bash
hostname
```

The response you see from both windows will look something like the following.
{{< output >}}
ip-172-31-62-220.us-west-2.compute.internal
{{< /output >}}

{{< step >}}Now, as [superuser](https://en.wikipedia.org/wiki/Superuser), try **updating** the name of the host from within **shell one**.{{< /step >}}
```bash
sudo hostname original-uts-namespace
```

{{< step >}}Then repeat the original command in **both** shells.{{< /step >}}
```bash
hostname
```

The response you see will be the **updated** name in both cases.
{{< output >}}
original-uts-namespace
{{< /output >}}

Furthermore, we can confirm the use of a shared UTS namespace by inspecting details in the aforementioned virtual filesystem at `/proc`.
{{< step >}}Enter the following command in **both** shells.{{< /step >}}
```bash
ls -l /proc/$$/ns/uts        # -l provides a long listing format, $$ is current shell PID
```

Output will look something like the following.
{{< output >}}
lrwxrwxrwx 1 ec2-user ec2-user 0 Jan 24 12:30 /proc/59266/ns/uts -> uts:[4026531838]
{{< /output >}}

The `uts:[XXXXXXXXXX]` identifier from **both** shells will be the same.
This consistency exists only because the shell processes have "inherited" the **one and only** UTS namespace from their parent process.
Based upon our previous experiments with environment variables, do you sense a pattern emerging?
It appears as if child processes, in many respects, begin their lives as clones of their parents.

You are ready now to introduce a **new** instance of the UTS namespace.
To experiment with Linux namespaces in the shell you will use a pair of commands which are just thin wrappers around a specific set of kernel syscalls.
- `unshare` - This command creates a child process as a member of a "new" Linux namespace. It says, "Take my offspring to an uninhabited island"
- `nsenter` - This command moves the current process into an existing Linux namespace. It says, "I wish to be alone with my family"

{{< columns >}}
{{< column title="shell one" >}}
{{< step >}}From **shell one**, use `unshare` to spawn a child shell inside a new instance of the UTS namespace, inspecting the `uts:[XXXXXXXXXX]` identifier before and after to verify the transition.{{< /step >}}
{{% example %}}
```bash
ls -l /proc/$$/ns/uts        # original namespace
sudo unshare --uts bash
ls -l /proc/$$/ns/uts        # new child shell has a different uts:[XXXXXXXXXX] identifier
```
{{% /example %}}

{{< step >}}From **shell one**, confirm that the new child shell "inherited" the name of the host and can change it.{{< /step >}}
{{% example %}}
```bash
hostname                     # before ...
sudo hostname new-linux-uts-namespace
hostname                     # ... after
```
{{% /example %}}

{{< output >}}
original-uts-namespace
new-uts-namespace
{{< /output >}}
{{< /column >}}

{{< column title="shell two" >}}
{{< step >}}From **shell two**, confirm that the name of the host is unchanged.{{< /step >}}
{{% example %}}
```bash
hostname
```
{{% /example %}}

{{< output >}}
original-uts-namespace
{{< /output >}}
{{< /column >}}
{{< /columns >}}

## Alan's attempt ...
{{< columns >}}
{{% column %}}
```bash
# SERVER
nc --listen 8000
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
cat <<< "request" > /dev/tcp/127.0.0.1/8000
```
{{% /column %}}
{{< /columns >}}

A process could also choose to coexist within the UTS namespace of another running process.
To migrate, a source process provides a PID which is a member of the destination UTS namespace.
As a process can only be a member of one UTS namespace at a time it will be evicted from its current one during migration.

{{< columns >}}
{{< column title="shell one" >}}
{{< step >}}From **shell one** grab the PID.{{< /step >}}
{{% example %}}
```bash
echo $$
```
{{% /example %}}
Which should yield a value such as:
{{< output >}}
48558
{{< /output >}}

{{< step >}}
Write *your* PID value down and use it in shell two (or copy and paste between shells).
{{< /step >}}

{{< /column >}}

{{< column title="shell two" >}}
{{< step >}}Replacing `<TARGET_PID_FROM_SHELL_ONE>` as appropriate, use `nsenter` in **shell two** to reunite the shells in your new UTS namespace.{{< /step >}}
{{% example %}}
```bash
PID=<TARGET_PID_FROM_SHELL_ONE>
sudo nsenter --target ${PID} --uts bash
```
{{% /example %}}

{{< step >}}Execute the following from **shell two** to verify that the shells are reunited.{{< /step >}}
{{% example %}}
```bash
hostname
```
{{% /example %}}

{{< output >}}
new-linux-uts-namespace
{{< /output >}}
{{< /column >}}
{{< /columns >}}

We can conclude from these experiments that the name of the host does not belonging merely to VMs.
Instead, it is a setting that belongs to instances of the UTS namespace.
This means we can fabricate **multiple hosts per VM**, a feat that was not possible before the Linux kernel exposed UTS namespaces.

{{< step >}}To ensure a smooth transition to the next section, close **both** terminal windows at this point.{{< /step >}}


{{< mermaid >}}
graph TB
    subgraph original-uts-namespace
         proc1((PID 5830<br>bash<br>shell1))
         host1>hostname: original]
         proc2((PID 5831<br>bash<br>shell2))
    end
    subgraph new-linux-uts-namespace
         proc3((PID 48558<br>bash))
         host2>hostname: new]
         proc4((PID 48559<br>bash))
    end
proc1 -->|unshare --uts| proc3
proc2 -->|nsenter --uts| proc4

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
class proc1,proc2 blue;
class proc3,proc4 green;
{{< /mermaid >}}

## Quizdown

Here is the demo quizdown quiz from https://github.com/bonartm/hugo-quiz/README.md:


{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## The sound of dog

---
shuffle_answers: false
---

What do dogs sound like?

> Some hint

- [ ] yes
- [ ] no
- [ ] `self.sound = "meow"`
- [x] wuff

## Put the [days](https://en.wikipedia.org/wiki/Day) in order!

> Monday is the *first* day of the week.

1. Monday
2. Tuesday
3. Wednesday
4. Friday
5. Saturday  
{{< /quizdown >}}

## First scratch Kubernetes quiz

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## What is Kubernetes?

---
shuffle_answers: false
---

Which of the following best describes Kubernetes?

> What is a name for a musical group with saxophone and violins?

- [ ] container runtime
- [x] container orchestrator
- [ ] container packager
- [ ] container image

## Unit of compute

Which of these is the fundamental unit of compute in Kubernetes?

> What do cetaceans travel in? Edamame? Peas?

- [ ] Container
- [x] Pod
- [ ] ReplicaSet
- [ ] Deployment

## Pod provisioning

Sequence these pieces of software in order of operation from your shell to the creation of containers in a pod.

> Trace the path your pod creation request would go through these Kubernetes components.

1. kubectl
2. API server
3. Scheduler
4. kubelet
5. container runtime (e.g. containerd, dockerd, cri-o)
{{< /quizdown >}}

## More on Mermaid

Removed this in the refactoring of the `Deployment` chapter...
...definitely scratch material

## Scratch

Yes, a scratch section at the end of this lesson. I'll hoist one up to top-level soon.

A test diagram to ensure that removing `./static/mermaid/` and the ancient `./static/mermaid/mermaid.min.js` 
has resolved the issue with that old mermaid version blocking the much more recent one in the hugo and hugo learn theme locations. 

{{< mermaid >}}
graph TD
subgraph some-id[Title]
    A --> B
end

style some-id fill:#F77,stroke:#F00,stroke-width:2px
{{< /mermaid >}}

Hurray! It worked. We can now have separate id and title for subgraph and even style subgraphs with the revised mermaid.

Therefore, we can style subgraphs within the `graph` diagram style.

{{< mermaid >}}
graph TB
subgraph Deployment-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: Deployment)
  subgraph Deployment-metadata[Deployment metadata]
    name(name)
    deploymentLabels[labels]
  end
  subgraph Deployment-spec[Deployment spec]
    replicas(replicas)
    strategy
    selector
    subgraph template
      subgraph Pod-metadata
        podLabels[labels]
      end
      subgraph Pod-spec
        containers
      end
    end
  end
end

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class Deployment-manifest orange;
class Deployment-metadata blue;
class Deployment-spec green;
class template yellow;
{{< /mermaid >}}

And even do the `flowchart` style which has proper direction flow inherited into its subgraphs.
So we can go vertical instead of horizontal with a depiction of the Deployment manifest.

{{< mermaid >}}
flowchart TB
subgraph Deployment-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: Deployment)
  subgraph Deployment-metadata[Deployment metadata]
    name(name)
    deploymentLabels[labels]
  end
  subgraph Deployment-spec[Deployment spec]
    subgraph spec-top-level[ ]
      replicas(replicas)
      strategy
      selector
    end
    subgraph template
      subgraph Pod-metadata
        podLabels[labels]
      end
      subgraph Pod-spec
        containers
      end
    end
  end
end

style spec-top-level fill:#F77,stroke:#F00,stroke-width:0px,color:#000

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
%% class Deployment-metadata orange;
%% class Deployment-spec yellow;
{{< /mermaid >}}

