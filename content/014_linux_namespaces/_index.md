---
title: "Linux Namespaces"
chapter: false
weight: 14
draft: false
---

## Purpose

You could learn a whole lot about why businesses need **containers**.
Endless discussions about operational efficiency, scalability, velocity, cost reduction, time to market, etc, etc.
Perhaps first, as technologists, you really ought to know what they are.

## Introduction

> **“Any sufficiently advanced technology is indistinguishable from magic”** - Arthur C Clarke

Once upon a not-so-long-ago a peculiar bunch of [Linux system calls](https://en.wikipedia.org/wiki/System_call) emerged that appeared to bend the "laws" of processes.
A handful of innovative disruptors seized the moment, recognizing the importance of these kernel extensions.
The world gazed in amazement as early adopters performed acts of "magic" with their workloads, operating at hitherto unimaginable scale and speed.
As time passed, and with a little assistance, we all caught up.
Now everyone has access to shrink-wrapped tools, allowing you to achieve the same feats with your own workloads.
Any process treated to a sprinkle of this kernel "magic" can become elevated to the status of a container.

Building and running containers is not magic, but it is also not exactly obvious.

## Linux namespaces

It is often useful to step back in time in order to see how we arrived to where we are today.

![pangaea](/images/process/pangaea.gif)

[Pangaea](https://en.wikipedia.org/wiki/Pangaea) was the single supercontinent that evolved to become the continents of the earth we know today.
Primitive mammals **shared** the single landmass but, as time passed, the continents formed and these inhabitants would become isolated by the seas.
The one constant throughout these formative years was the earth.

<!-- Closer to modern times continents would be divided into countries and some these countries would be divided further into states, counties, etc. -->

In this analogy.
- The earth represents a single Linux virtual machine (VM)
- The land-dwelling inhabitants represent processes
- The splintering landmass of Pangaea represents [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces)

{{% notice note %}}
The term "namespace" is used in a variety of contexts.
For example, **Kubernetes** namespaces are a different concept to **Linux** namespaces.
For the duration of this chapter when you see the word namespace you must interpret that as **Linux namespace**.
{{% /notice %}}

As we will learn later, there are a small handful of distinct namespace types and each process must reside in **exactly** one instance of each namespace type at any point in time.
Some types are more interesting and/or simple to demonstrate than others.
In the interests of practicality we will start by acknowledging just the [UTS](https://en.wikipedia.org/wiki/Linux_namespaces#UTS) namespace type.

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

{{< step >}}From **shell one**, use `unshare` to spawn a child shell inside a new instance of the UTS namespace, inspecting the `uts:[XXXXXXXXXX]` identifier before and after to verify the transition.{{< /step >}}
```bash
ls -l /proc/$$/ns/uts        # original namespace
sudo unshare --uts bash
ls -l /proc/$$/ns/uts        # new child shell has a different uts:[XXXXXXXXXX] identifier
```

{{< step >}}From **shell one**, confirm that the new child shell "inherited" the name of the host and can change it.{{< /step >}}
```bash
hostname                     # before ...
sudo hostname new-linux-uts-namespace
hostname                     # ... after
```

{{< output >}}
original-uts-namespace
new-uts-namespace
{{< /output >}}

{{< step >}}From **shell two**, confirm that the name of the host is unchanged.{{< /step >}}
```bash
hostname
```

{{< output >}}
original-uts-namespace
{{< /output >}}

A process could also choose to coexist within the UTS namespace of another running process.
To migrate, a source process provides a PID which is a member of the destination UTS namespace.
As a process can only be a member of one UTS namespace at a time it will be evicted from its current one during migration.
{{< step >}}From **shell one** grab the PID.{{< /step >}}
```bash
echo $$
```

{{< step >}}Replacing `<TARGET_PID_FROM_SHELL_ONE>` as appropriate, use `nsenter` in **shell two** to reunite the shells in your new UTS namespace.{{< /step >}}
```bash
PID=<TARGET_PID_FROM_SHELL_ONE>
sudo nsenter --target ${PID} --uts bash
```

{{< step >}}Execute the following from **shell two** to verify that the shells are reunited.{{< /step >}}
```bash
hostname
```

{{< output >}}
new-linux-uts-namespace
{{< /output >}}

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


### The Linux PID namespace (optional)

As noted previously, there are different types of namespaces that can be combined to deliver varying characteristics of process "isolation" which can help improve security.
For example, process "A" can only observe process "B" if it inhabits the same [PID namespaces](https://en.wikipedia.org/wiki/Linux_namespaces#Process_ID_.28pid.29) or another one located further down the process tree.

{{< step >}}Try out PID namespaces for yourself by creating a pair of terminal windows (**shell one** and **shell two**) inside Cloud9.{{< /step >}}

{{< step >}}Execute the following from **both** shells to verify that a long list of processes running on the VM can be viewed.{{< /step >}}
```bash
sudo ps -e
```

{{< step >}}Execute the following in **both** shells to build new PID namespaces, **one in each shell**.{{< /step >}}
```bash
ls -l /proc/$$/ns/pid                  # original namespace
sudo unshare --pid --mount --fork bash # a little more unshare complexity this time
mount -t proc name /proc               # the mountpoint for /proc does not auto-update so re-pin it
ls -l /proc/$$/ns/pid                  # new child shell has a different pid:[XXXXXXXXXX] identifier
```

{{% notice note %}}
In order to reach a viable state to host containers, as you have just seen, some Linux namespaces require a little more coaxing than others.
This is a key reason why this technology required some [democratization](https://en.wikipedia.org/wiki/Democratization_of_technology).
{{% /notice %}}


{{< step >}}Execute the following from **shell one** to start a long running process in the background.{{< /step >}}
```bash
sleep 1001 &
```

{{< step >}}Do something similar from **shell two**.{{< /step >}}
```bash
sleep 1002 &
```

{{< step >}}Now execute the `ps` from **both** shells.{{< /step >}}
```bash
echo $$
sudo ps -ef
```

Example output from **shell one** as follows.
{{< output >}}
1
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 18:18 pts/2    00:00:00 bash
root          15       1  0 18:18 pts/2    00:00:00 sleep 1001
root          16       1  0 18:18 pts/2    00:00:00 ps -ef
{{< /output >}}

Noteworthy points from the response are as follows.
- Inside your new Linux PID namespaces, the current PID `$$` is always initialized to 1.
- The list of member processes is reassuringly short.
- All processes appear to have been run as the `root` user.

{{< mermaid >}}
graph TB
    subgraph pid-namespace-zero
         proc1((PID 5830<br>bash<br>shell1))
         proc2((PID 5831<br>bash<br>shell2))
    end
    subgraph pid-namespace-one
         proc3((PID 1<br>bash))
         proc4((PID 15<br>sleep 1001))
         proc5((PID 16<br>ps -ef))
    end
    subgraph pid-namespace-two
         proc6((PID 1<br>bash))
         proc7((PID 15<br>sleep 1002))
         proc8((PID 16<br>ps -ef))
    end
proc1 -->|unshare --pid| proc3
proc2 -->|unshare --pid| proc6
proc3 --> proc4 & proc5
proc6 --> proc7 & proc8

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
class proc1,proc2 blue;
class proc3,proc4,proc5 green;
class proc6,proc7,proc8 orange;
{{< /mermaid >}}

## Summary of namespace types

Here's the summary of the main namespace types and word or two on their use.
- `UTS` - independent host names
- `PID` - independent sets of process IDs
- `Mount` - independent mount points for file/data access
- `Network` - independent (virtualized) IP addresses and ports
- ... those are the main ones.

Namespaces are key enabler for building containers on Linux.
However, we must not forget that containers can be further resource-constrained with the use of [control groups](https://en.wikipedia.org/wiki/Cgroups) ... but you have covered enough to move on for now.

## Success

In this exercise you did the following:
- Built and used instances of the UTS namespace
- Did the same for the PID namespace
- Summarized the main namespaces in use today