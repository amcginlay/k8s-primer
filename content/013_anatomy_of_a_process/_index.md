---
title: "The Anatomy Of A Process"
chapter: false
weight: 013
draft: false
---

## Purpose
Imagine you are cycling up a hill in a high gear and the bicycle starts to slow down.
You have two choices.
You can either carry on in the hope your momentum and strength will take you safely over the hill or you can drop down a gear and pedal faster.
The latter option will (almost) always enable you to progress to the next hill.
The point is, it is not always desirable or efficient to slow down but it is sometimes necessary choice to progress.

So, as experienced technicians you know what a process is, correct?
Of course you do.

You likely know enough to get by, but do you know the detail required to progress to the next hill without stalling?
In order to learn Kubernetes with confidence you will likely benefit from revisiting the humble **process** to gain a renewed appreciation how they interact with each other and the "quirky tricks" they are capable of.

## Common misconceptions about processes
There are **two** common, yet significant, misconceptions about processes that need to be dispelled.
They are as follows.

### 1) Processes are always siblings (not true)

Is this screen familiar to you?

![task-manager](/images/process/task-manager.png)

Tools like the [Task Manager](https://en.wikipedia.org/wiki/Task_Manager_(Windows)), the [Activity Monitor](https://en.wikipedia.org/wiki/List_of_macOS_components#Activity_Monitor) or even the [top](https://en.wikipedia.org/wiki/Top_(software)) utility may be partially to blame for this.
Running processes are **not** always in perfect (sibling) alignment with each other.
Much like a file system of directories and files, running processes form a tree (or hierarchy) of parents, children, grandchildren and so on.
Any running process can spawn children and those processes will "inherit" some of the DNA of its parent (see point 2).

As you will know, deleting a directory from a file system will also destroy its descendants
The same is true of processes.
When you terminate any process it will cause that entire branch of the process tree to vanish from existence.

{{% notice note %}}
The author is aware that later incarnations of the Windows Task Manager would provide a more accurate representation of the process tree.
{{% /notice %}}

### 2) All processes can access the same environment variables (not true)

Is this screen familiar to you?

![win-env-var](/images/process/win-env-var.png)

If you need to add or edit a Windows [environment variable](https://en.wikipedia.org/wiki/Environment_variable) you just make the change in this dialog and it is immediately available from within your [Windows CMD sessions]((https://en.wikipedia.org/wiki/Cmd.exe)), correct?
Not quite.
You may recall that you would also need to **restart** your CMD sessions for those changes to be visible from inside the session.
This is because the settings you make via the Environment Variables dialog belong to a top-level process (**explorer.exe**) which represents your desktop environment.
Whenever you create a CMD session, your desktop environment spawns a **child process** which takes a **point-in-time** copy of these environment variables from this top-level process and this is why existing sessions require a restart.
At this point, if you restart just **some** CMD sessions but not others then the `env` command will yield inconsistent results.
Enough said.

This behavior is by design and consistent across all popular operating systems, including Linux.
So lets switch our attention to our Cloud9 environment and see some of this in action.

## Hands on with processes

Return to your existing Cloud9 environment and, if necessary, start a new command line session using `Window -> New Terminal`.
Each shell instance is a process like any other and each process has a [process identifier](https://en.wikipedia.org/wiki/Process_identifier) (PID) which is, ostensibly, unique across any single virtual machine instance.
You can identify the PID of the current shell as follows.
```bash
echo $$
```

{{< output >}}
5831
{{< /output >}}

{{% notice note %}}
Linux provides a virtual filesystem at [/proc](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html) which presents an entry for every active PID plus some generic system info.
You can inspect information pertaining the current shell by running `ls -l /proc/$$/`.
You will revisit the /proc virtual filesystem later.
{{% /notice %}}

The numerical response to `echo $$` is the PID for the current shell.
Upon repeat invocations its response is consistent but will differ between any `New Terminal` sessions you create.
Now check the collection of currently running processes under the current user ID (UID) `ec2-user`.
```bash
ps -f
```

Which produces something like this.
{{< output >}}
UID          PID    PPID  C STIME TTY          TIME CMD
ec2-user    5830    5829  0 14:26 pts/1    00:00:00 bash -c export ISOUTPUTPANE=0;bash -l
ec2-user   >5831<   5830  2 14:26 pts/1    00:00:00 bash -l
ec2-user    6443    5831  0 14:26 pts/1    00:00:00 ps -f
{{< /output >}}

Noteworthy points from the response are as follows.
- The current shell PID (`$$`) is included in this list.
- [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) is the default shell in Cloud9 and it runs as a regular process.
- The `-f` switch on `ps` provides a full-format listing which enables you to identify the **parent PIDs** (PPID) of your processes.
- Your call to `ps` is itself (momentarily) a child process of your current shell.
- You will observe three processes and three generations of processes. In the above example, PID `6443` is a child of PID `5831` which is a child of PID `5830`.

{{% notice note %}}
The PPID of the current shell can be discovered using `echo $PPID`.
{{% /notice %}}


Now you can experiment.
Create an environment variable and check it was successfully set.
```bash
K8S_PRIMER=before            # create a local variable
export K8S_PRIMER            # "promote" as an environment variable
env | grep K8S_PRIMER
```

{{% notice note %}}
Be mindful of the syntax used.
Local variables and environment variables look and behave the same but **only** environment variables (referred to as **variables** from this point on) are "inherited" by child processes.
This difference is subtle but important.
{{% /notice %}}

Next, create a child process which sticks around a little longer than `ps`.
The easiest way to achieve this is to ask your shell (Bash) to spawn another shell.
```bash
bash
```

The new child shell becomes your currently active shell.
Now inspect your current PID and summarize your running processes.
```bash
echo $$
ps -f
```

{{< output >}}
48558
UID          PID    PPID  C STIME TTY          TIME CMD
UID          PID    PPID  C STIME TTY          TIME CMD
ec2-user    5830    5829  0 14:26 pts/1    00:00:00 bash -c export ISOUTPUTPANE=0;bash -l
ec2-user    5831    5830  0 14:26 pts/1    00:00:00 bash -l
ec2-user  >48558<   5831  0 15:54 pts/1    00:00:00 bash
ec2-user   48591   48558  0 15:54 pts/1    00:00:00 ps -f
{{< /output >}}

{{% notice note %}}
Observe that the PPID of your current shell matches the PID of your original shell.
{{% /notice %}}

Now check that the variable you initialized in the parent process is visible from the child process.
```bash
echo ${K8S_PRIMER}
```

{{< output >}}
before
{{< /output >}}

So the variable is visible and set appropriately, but what about making modifications to variables from within the child process?
```bash
K8S_PRIMER=after
echo ${K8S_PRIMER}
```

{{< output >}}
after
{{< /output >}}

Also good.
So this modification from `true` to `false` must be visible from the parent process too, correct?
The original shell is still there, in a passive state, patiently waiting for the child to pass control back to it.
Use `exit` to terminate the child process so the parent regains input from the keyboard and you can verify this assertion.
```bash
exit                         # head back in the originating shell (PPID)
echo $$                      # back in the original shell
echo ${K8S_PRIMER}
```

{{< output >}}
5831
before
{{< /output >}}

So the modification made in the child is **not** visible here in the originating parent process.
This is because it **never** was the exact same variable, it just happened to have the same name.
Recall from our earlier discussion (re. Windows CMD sessions) that when a child process is spawned it takes a point-in-time **copy** of the variables.
We have just observed the same behavior in a different operating system.

To be clear, the "copied state" behavior that exists for variables between parent and child processes is observable at **any** depth in the tree and it transpires that this **pattern** is not merely confined to variables.
This allows you to do some pretty amazing things as you will find out in the next section.

## Linux namespaces
It is often useful to step back in time in order to see how we arrived to where we are today.

![pangaea](/images/process/pangaea.gif)

[Pangaea](https://en.wikipedia.org/wiki/Pangaea) was the single supercontinent that evolved to become continents of the earth we know today.
Primitive mammals shared the single landmass but, as time passed, the continents formed and these inhabitants would become separated by the seas.
The one constant over time is the earth.

<!-- Closer to modern times continents would be divided into countries and some these countries would be divided further into states, counties, etc. -->

In this analogy.
- The earth represents a single Linux virtual machine (VM)
- The land-dwelling inhabitants represent processes
- The splintering landmass of Pangaea represents [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces)

{{% notice note %}}
The term "namespace" is used in a variety of contexts.
For example, **Kubernetes** namespaces are a different concept to **Linux** namespaces.
To avoid confusion their use will always be qualified.
{{% /notice %}}

As we will learn later, there are small handful of distinct Linux namespace types and each process must reside in **exactly** one instance of each namespace type at any single point in time.
Some types are more interesting and/or simple to demonstrate than others.
In the interests of practicality we will start by acknowledging just the [UTS](https://en.wikipedia.org/wiki/Linux_namespaces#UTS) Linux namespace type.

### The Linux UTS namespace

<!-- Assuming [superuser](https://en.wikipedia.org/wiki/Superuser) privileges are in place, the VM functions just as you would expect. -->
Because each Linux VM starts with a single UTS namespace **shared** across all running processes you will likely remain oblivious to its existence.
As an example, consider the [hostname](https://en.wikipedia.org/wiki/Hostname) command.
It consistently responds with a single (global) value across all shells and changes are observable from all shells.

Try this out for yourself by creating a pair of terminal windows (**shell one** and **shell two**) inside Cloud9.
Your screen layout may look something like this.

![shell-one-two](/images/process/shell-one-two.png)

Now execute the following in **both** shells.
```bash
hostname
```

The response you see from both windows will look something like the following.
{{< output >}}
ip-172-31-62-220.us-west-2.compute.internal
{{< /output >}}


Now, as [superuser](https://en.wikipedia.org/wiki/Superuser), try **updating** the name of the host from within **shell one**.
```bash
sudo hostname original-linux-uts-namespace
```

Then repeat the original command in **both** shells.
```bash
hostname
```

The response you see will be the **updated** name in both cases.
{{< output >}}
original-linux-uts-namespace
{{< /output >}}

Furthermore, we can confirm the use of shared Linux UTS namespace by inspecting details in the aforementioned virtual filesystem at `/proc`.
Enter the following command in **both** shells.
```bash
ls -l /proc/$$/ns/uts        # -l provides a long listing format, $$ is current shell PID
```

Output will look something like the following.
{{< output >}}
lrwxrwxrwx 1 ec2-user ec2-user 0 Jan 24 12:30 /proc/59266/ns/uts -> uts:[4026531838]
{{< /output >}}

The `uts:[XXXXXXXXXX]` identifier from **both** shells will be the same.
This consistency exists only because the shell processes have "inherited" the **one and only** Linux UTS namespace from their parent process.
Do you sense a pattern emerging?

You are ready now to introduce a **bespoke** instance of the Linux UTS namespace.
To experiment with Linux namespaces in the shell you will use a pair of commands which are just thin wrappers around a specific set of [Linux kernel syscalls](https://en.wikipedia.org/wiki/System_call).
- `unshare` - This command creates a child process as a member of a "bespoke" Linux namespace. It is saying, "Take my offspring to an uninhabited island"
- `nsenter` - This command moves the current process into an existing Linux namespace. It is saying, "I wish to be alone with my family"

From **shell one**, use `unshare` to spawn a child shell inside a bespoke instance of the Linux UTS namespace, inspecting the `uts:[XXXXXXXXXX]` identifier before and after to verify the transition.
```bash
ls -l /proc/$$/ns/uts        # original namespace
sudo unshare --uts bash
ls -l /proc/$$/ns/uts        # new child shell has a different uts:[XXXXXXXXXX] identifier
```

From **shell one**, confirm that the new child shell "inherited" the name of the host and can change it.
```bash
hostname                     # before ...
sudo hostname bespoke-linux-uts-namespace
hostname                     # ... after
```

{{< output >}}
original-linux-uts-namespace
bespoke-linux-uts-namespace
{{< /output >}}

From **shell two**, confirm that the name of the host is unchanged.
```bash
hostname
```

{{< output >}}
original-linux-uts-namespace
{{< /output >}}

A process could also choose to coexist within the Linux UTS namespace of another running process.
To migrate, a source process provides a PID which is a member of the destination Linux UTS namespace.
As a process can only be a member of one Linux UTS namespace at a time it will be evicted from its current one during migration.
From **shell one** grab the PID
```bash
echo $$
```

Replacing `<TARGET_PID_FROM_SHELL_ONE>` as appropriate, use `nsenter` in **shell two** to reunite the shells in your bespoke Linux UTS namespace.
```bash
PID=<TARGET_PID_FROM_SHELL_ONE>
sudo nsenter --target ${PID} --uts bash
```

Execute the following from **shell two** to verify that the shells are reunited.
```bash
hostname
```

{{< output >}}
bespoke-linux-uts-namespace
{{< /output >}}

We can conclude from these experiments that the name of the host does not belonging merely to VMs.
Instead, it is a setting that belongs to instances of the Linux UTS namespace.
This means we can fabricate **multiple hosts per VM**, a feat that was not possible before the Linux kernel exposed UTS namespaces.

To ensure a smooth transition to the next section, close **both** terminal windows at this point.

### The Linux PID namespace (optional)

As noted previously, there are different types of Linux namespaces that can be combined to deliver varying characteristics of process "isolation" which can help improve security.
For example, a process inhabiting one bespoke [Linux PID namespaces](https://en.wikipedia.org/wiki/Linux_namespaces#Process_ID_.28pid.29) can only observe another process if that process also inhabits the same bespoke Linux PID namespace.

Try out Linux PID namespaces for yourself by creating a pair of terminal windows (**shell one** and **shell two**) inside Cloud9.

Execute the following from **both** shells to verify that a long list of processes running on the VM can be viewed.
```bash
sudo ps -e
```

Execute the following in **both** shells to build bespoke Linux PID namespaces, **one in each shell**.
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


Execute the following from **shell one** to start a long running process in the background.
```bash
sleep 1001 &
```

Do something similar from **shell two**.
```bash
sleep 1002 &
```

Now execute the `ps` from **both** shells.
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
/home/ec2-user/environment $ 
{{< /output >}}

Noteworthy points from the response are as follows.
- Inside your bespoke Linux PID namespaces, the current PID `$$` is always initialized to 1.
- The list of member processes is reassuringly short.
- All processes appear to have been run as the `root` user.

## Summary of Linux namespace types

Here's the summary of the main Linux namespace types and word or two on their use.
- `UTS` - independent host names
- `PID` - independent sets of process IDs
- `Mount` - independent mount points for file/data access
- `Network` - independent (virtualized) IP addresses and ports
- ... those are the main ones.

Namespaces are key enabler for building containers on Linux.
However, we must not forget that containers can be further resource-constrained with the use of [control groups](https://en.wikipedia.org/wiki/Cgroups) ... but you have covered enough to move on for now.

## Success

In this exercise you did the following:
- Discussed some common misconceptions
- Put environment variables through their paces to reveal their true nature
- Built and used instances of the Linux UTS namespace
- Did the same for the Linux PID namespace
- Summarized the main Linux namespaces in use today