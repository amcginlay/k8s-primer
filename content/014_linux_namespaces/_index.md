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

Once upon a not-so-long-ago a peculiar bunch of [Linux system calls](https://en.wikipedia.org/wiki/System_call) emerged that appeared to fundamentally twist the relationship between processes and their hosts.
A handful of innovative disruptors seized the moment, recognizing the importance of these kernel extensions.
The world gazed in amazement as early adopters performed acts of "magic" with their workloads, operating at a scale and speed not previously possible.

Processes treated to a sprinkle of this kernel wizardry become elevated to the status of containers.
With the passage of time the use of containers, somewhat inevitably, became [democratized](https://en.wikipedia.org/wiki/Democratization_of_technology) and the air of mystique that previously surrounded them faded.

You would rarely build a container by hand these days because you tend to have more productive things to do with your time.
That said, if we want to gain a deeper appreciation of the technology that underpins Kubernetes pausing for a moment to do exactly that can provide a powerful accelerant to your learning.

Building and running rudimentary containers by hand is perhaps not for the faint of heart but neither is it magic.

## Linux namespaces

It is often useful to step back in time in order to see how we got here.

![pangaea](/images/process/pangaea.gif)

[Pangaea](https://en.wikipedia.org/wiki/Pangaea) was the single supercontinent that evolved to become the continents of the earth we know today.
Primitive mammals **shared** the land but, as time passed, the continents formed and the inhabitants would become isolated by the seas.

In this analogy.
- The earth represents a single Linux virtual machine (VM)
- The land-dwelling inhabitants represent processes
- The splintering landmass of Pangaea represents [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces)

{{% notice note %}}
The term "namespace" is used in a variety of contexts.
For example, **Kubernetes** namespaces are a different concept to **Linux** namespaces.
For the duration of this chapter when you see the word namespace you must interpret that as **Linux namespace**.
{{% /notice %}}

As we will learn later, there are a small handful of distinct namespace **types**.
Each process **must** reside in a collection of namespaces but **just one** of each type.
Furthermore, any child processes will reside in the same namespaces of their parent process unless they request otherwise, as we will see.

Some types of namespaces are more interesting and/or simple to demonstrate than others.
In the interests of practicality we will start by acknowledging just one of them.

### UTS namespaces

{{% notice note %}}
UTS stands for UNIX Time-Sharing.
It is not important why this name was chosen and most would agree it is not the most descriptive name.
If it helps, you can think of it as the **host namespace**.
{{% /notice %}}

Each Linux VM begins its life with a single instance of the [UTS namespace](https://en.wikipedia.org/wiki/Linux_namespaces#UTS) which is, by default, shared across all its running process.
The ubiquitous nature of root namespaces is the likely reason you may have been oblivious to their existence.
To illustrate its presence, consider the [hostname](https://en.wikipedia.org/wiki/Hostname) command.
By default the "value" of `hostname`, and any modification to it, is observable and consistent across all shell instances (processes).

{{< step >}}Try this out for yourself by creating side-by-side terminal windows (**shell one** and **shell two**) inside Cloud9.
Now execute the following in **both** shells.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
hostname
```
{{% /column %}}
{{% column %}}
```bash
# shell two
hostname
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
ip-172-31-49-172.us-west-2.compute.internal
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
ip-172-31-49-172.us-west-2.compute.internal
{{< /output >}}
{{< /column >}}
{{< /columns >}}

{{< step >}}Now, as [superuser](https://en.wikipedia.org/wiki/Superuser), try **updating** the name of the host from within **shell one**, then **repeat** the original command in **both** shells.{{< /step >}}
{{< columns >}}
{{% column %}}
```bash
# shell one - rename the host
sudo hostname this-is-root-uts
hostname
```
{{% /column %}}
{{% column %}}
```bash
# shell two - no rename required

hostname
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
this-is-root-uts
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
this-is-root-uts
{{< /output >}}
{{< /column >}}
{{< /columns >}}

The response you see will be the **updated** name in both cases.
Furthermore, we can confirm the use of a shared namespaces (including UTS) by inspecting details in the aforementioned virtual filesystem at `/proc`.
{{< step >}}Enter the following command in **both** shells, recalling that `$$` retrieves the `PID` of the current shell. You will use `&&` here to execute two commands consecutively, improving the readability of the output.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
echo "PID=$$" && ls -ld /proc/$$/ns/*
```
{{% /column %}}
{{% column %}}
```bash
# shell two
echo "PID=$$" && ls -ld /proc/$$/ns/*
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
8241
lrwxrwxrwx ... cgroup -> cgroup:[4026531835]
lrwxrwxrwx ... ipc -> ipc:[4026531839]
lrwxrwxrwx ... mnt -> mnt:[4026531840]
lrwxrwxrwx ... net -> net:[4026531993]
lrwxrwxrwx ... pid -> pid:[4026531836]
lrwxrwxrwx ... pid_for_children -> pid:[4026531836]
lrwxrwxrwx ... user -> user:[4026531837]
lrwxrwxrwx ... uts -> uts:[4026531838]
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
8498
lrwxrwxrwx ... cgroup -> cgroup:[4026531835]
lrwxrwxrwx ... ipc -> ipc:[4026531839]
lrwxrwxrwx ... mnt -> mnt:[4026531840]
lrwxrwxrwx ... net -> net:[4026531993]
lrwxrwxrwx ... pid -> pid:[4026531836]
lrwxrwxrwx ... pid_for_children -> pid:[4026531836]
lrwxrwxrwx ... user -> user:[4026531837]
lrwxrwxrwx ... uts -> uts:[4026531838]
{{< /output >}}
{{< /column >}}
{{< /columns >}}

Whilst the PIDs are clearly different, **all** the namespace identifiers (e.g. `uts:[4026531838]`) will match, type for type.
This consistency exists because the shell processes are both inhabiting the same namespaces.
It follows that, if we were to traverse up the "family" tree, these shells would ultimately converge on a **shared** ancestor which also inhabits those same namespaces.

Based upon our previous experiments with environment variables, do you sense a pattern emerging?
It appears as if child processes, in many respects, begin their lives as clones of their parents and this pattern can be tracked right the way back to an initiating process with `PID` 1.

{{% notice note %}}
The process with `PID` 1 is unusual in that it has a `PPID` of 0, yet no such process exists, therefore it has no parent.
{{% /notice %}}

You are ready now to introduce a **new** instance of the UTS namespace.
To experiment with namespaces in the shell you will use a pair of commands which are just thin wrappers around a small collection of kernel syscalls.
- [unshare](https://man7.org/linux/man-pages/man1/unshare.1.html) spawns a child process as a member of a **new** namespace.
- [nsenter](https://man7.org/linux/man-pages/man1/nsenter.1.html) spawns a child process as a member of an **existing** namespace.

{{< step >}}From **shell one**, use `unshare` to spawn a child shell into a **new** instance of the UTS namespace, inspecting the `uts:[XXXXXXXXXX]` identifier before and after to verify the transition.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
echo "PID=$$" && ls -l /proc/$$/ns/uts
sudo unshare --uts bash
echo "PID=$$" && ls -l /proc/$$/ns/uts
hostname
```
{{% /column %}}
{{% column %}}
```bash
# shell two - no action required
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
PID=8241
lrwxrwxrwx ... /proc/8241/ns/uts -> uts:[4026531838]
PID=32609
lrwxrwxrwx ... /proc/32609/ns/uts -> uts:[4026532177]
this-is-root-uts
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
{{< /output >}}
{{< /column >}}
{{< /columns >}}

In the output of the second `ls` command you can observe how the child process which was spawned from the `unshare` command and is now active in **shell one** has a new `uts:[XXXXXXXXXX]` identifier.
Curiously the `hostname` command appears to reveal that this **new** namespace was initialized with a **copy** of the name of the **root** namespace.
This is just the default behavior.

{{< step >}}To prove the name was indeed **copied**, change it in **shell one** and compare it to **shell two**.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one - change hostname
sudo hostname this-is-new-uts
hostname
```
{{% /column %}}
{{% column %}}
```bash
# shell two - leave hostname UNCHANGED

hostname
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
this-is-new-uts
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
this-is-root-uts
{{< /output >}}
{{< /column >}}
{{< /columns >}}

These are different, proving that the active processes of **shell one** and **shell two** reside in independent namespaces.

With the help of `nsenter` a process can choose to spawn a child into the UTS namespace of another running process.
To achieve this, the spawning process needs to target an active PID.
Upon birth, the child process will inhabit the UTS namespace of the target PID.

{{< step >}}Dump the PID of **shell one** to a temp file then, within **shell two**, tell `nsenter` to target that PID's UTS namespace whilst spawning a child process.{{< /step >}}
{{< columns >}}
{{% column %}}
```bash
# shell one
echo "PID=$$" && echo $$ > /tmp/target-pid
```
{{% /column %}}
{{% column %}}
```bash
# shell two
echo "PID=$$" && target=$(cat /tmp/target-pid)
echo "TARGET=${target}"
sudo nsenter --target ${target} --uts bash 
echo "PID=$$"
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
PID=32609
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
PID=8498
TARGET=32609
PID=11587
{{< /output >}}
{{< /column >}}
{{< /columns >}}

From which you can observe that the PID of **shell two** has changed to that of the child process `nsenter` spawned.

{{< step >}}Execute the following from from **both shells** to verify that the shells are reunited in the **new** UTS namespace.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
hostname
```
{{% /column %}}
{{% column %}}
```bash
# shell two
hostname
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
this-is-new-uts
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
this-is-new-uts
{{< /output >}}
{{< /column >}}
{{< /columns >}}

The above experiment demonstrated how it is possible to create the illusion of multiple hosts on a single Linux VM.
Prior to the introduction of UTS namespaces this was not possible.
This was a minimal example of what is possible when we target just one of the namespace types.

The following diagram illustrates the steps you just performed.

{{< mermaid >}}
graph TB
    subgraph Root UTS Namespace
         proc1((PID 8241<br>shell1<br>bash))
         host1>hostname: this-is-root-uts]
         proc2((PID 8498<br>shell2<br>bash))
    end
    subgraph Custom UTS Namespace
         proc3((PID 32609<br>shell1<br>bash))
         host2>hostname: this-is-new-uts]
         proc4((PID 11587<br>shell2<br>bash))
    end
proc1 -->|unshare --uts| proc3
proc2 -->|nsenter --uts| proc4

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
class proc1,proc2 blue;
class proc3,proc4 green;
{{< /mermaid >}}

{{< step >}}To ensure a smooth transition to the next section, close **both** terminal windows at this point.{{< /step >}}

### PID namespaces

As noted previously, there are different types of namespaces that can be combined to deliver varying characteristics of process "isolation" which can help improve security.
As with UTS namespaces, a VM begins life with a **root** [PID namespaces](https://en.wikipedia.org/wiki/Linux_namespaces#Process_ID_.28pid.29) which has visibility over **all** running processes.
As with UTS namespaces, you may spawn child processes into new PID namespaces which can, for example, restrict which subset of processes the inhabitants can see.

{{< step >}}Try out PID namespaces for yourself by creating a pair of terminal windows (**shell one** and **shell two**) inside Cloud9. Then, execute the following from **both** shells to verify that a long list of processes running on the VM can be viewed.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
sudo ps -e
```
{{% /column %}}
{{% column %}}
```bash
# shell two
sudo ps -e
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
  PID TTY          TIME CMD
    1 ?        00:00:05 systemd
    2 ?        00:00:00 kthreadd
    4 ?        00:00:00 kworker/0:0H
...
25489 pts/3    00:00:00 sudo
25490 pts/3    00:00:00 bash
32728 ?        00:00:00 kworker/u4:1
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
  PID TTY          TIME CMD
    1 ?        00:00:05 systemd
    2 ?        00:00:00 kthreadd
    4 ?        00:00:00 kworker/0:0H
...
25489 pts/3    00:00:00 sudo
25490 pts/3    00:00:00 bash
32728 ?        00:00:00 kworker/u4:1
{{< /output >}}
{{< /column >}}
{{< /columns >}}

{{< step >}}Execute the following in **both** shells to build new PID namespaces, **one in each shell**.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
ls -l /proc/$$/ns/pid
sudo unshare --pid --mount --fork bash
mount -t proc name /proc
ls -l /proc/$$/ns/pid
```
{{% /column %}}
{{% column %}}
```bash
# shell two
ls -l /proc/$$/ns/pid
sudo unshare --pid --mount --fork bash
mount -t proc name /proc
ls -l /proc/$$/ns/pid
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
lrwxrwxrwx ... /proc/22803/ns/pid -> pid:[4026531836]
lrwxrwxrwx ... /proc/1/ns/pid -> pid:[4026532179]
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
lrwxrwxrwx ... /proc/22804/ns/pid -> pid:[4026531836]
lrwxrwxrwx ... /proc/1/ns/pid -> pid:[4026532181]
{{< /output >}}
{{< /column >}}
{{< /columns >}}

So both shells inhabit the same PID namespace, until after we call `unshare` and `mount`.
Curiously, you may also observe evidence to suggest that the `PID`s have been "reset" to begin at 1 inside the new namespaces.

{{% notice note %}}
In the above code we call `unshare` followed by `mount`.
This is because the mountpoint at `/proc` is **copied** from the parent but does not auto-update to correspond with its new context so we need to re-pin it
This quirk serves to illustrate how some namespaces require a little more coaxing than others.
It is a key reason why container technology required some [democratization](https://en.wikipedia.org/wiki/Democratization_of_technology).
{{% /notice %}}


{{< step >}}Execute the following from your shells to start identifiable long running processes in the background.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
sleep 1001 &
```
{{% /column %}}
{{% column %}}
```bash
# shell two
sleep 1002 &
```
{{% /column %}}
{{< /columns >}}

{{< step >}}Now execute the `ps` from **both** shells.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
# shell one
echo "PID=$$" && sudo ps -ef
```
{{% /column %}}
{{% column %}}
```bash
# shell two
echo "PID=$$" && sudo ps -ef
```
{{% /column %}}
{{< /columns >}}

Example output:
{{< columns >}}
{{< column >}}
{{< output >}}
PID=1
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 14:27 pts/6    00:00:00 bash
root        20     1  0 14:40 pts/6    00:00:00 sleep 1001
root        21     1  0 14:40 pts/6    00:00:00 sudo ps -ef
root        22    21  0 14:40 pts/6    00:00:00 ps -ef
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
PID=1
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 14:28 pts/7    00:00:00 bash
root        18     1  0 14:40 pts/7    00:00:00 sleep 1002
root        19     1  0 14:40 pts/7    00:00:00 sudo ps -ef
root        20    19  0 14:40 pts/7    00:00:00 ps -ef
{{< /output >}}
{{< /column >}}
{{< /columns >}}

Noteworthy points from the response are as follows.
- Inside your new PID namespaces, the current PID `$$` is always initialized to 1.
- The list of member processes is reassuringly short.
- All processes appear to have been run as the `root` user.

The following diagram illustrates the steps you just performed.

{{< mermaid >}}
graph TB
    subgraph pid-namespace-zero
         proc1((PID 22803<br>shell1<br>bash))
         proc2((PID 22804<br>shell2<br>bash))
    end
    subgraph pid-namespace-two
         proc6((PID 1<br>bash))
         proc7((PID 18<br>sleep 1002))
         proc8((PID 20<br>ps -ef))
    end
    subgraph pid-namespace-one
         proc3((PID 1<br>bash))
         proc4((PID 20<br>sleep 1001))
         proc5((PID 22<br>ps -ef))
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

Even though processes from within different custom namespaces (e.g. `sleep 1001` and `sleep 1002`) cannot see each other, **all** processes remain visible from the root namespace.
How you might go about testing that the previous statement is true?

{{< step >}}To ensure a smooth transition to the next section, close **both** terminal windows at this point.{{< /step >}}

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