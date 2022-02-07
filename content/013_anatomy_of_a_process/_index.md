---
title: "The Anatomy Of A Process"
chapter: false
weight: 13
draft: false
---

## Purpose
Imagine you are cycling up a hill in a high gear and the bicycle starts to slow down.
You have two choices.
You can either carry on in the hope your momentum and strength will take you safely over the hill or you can drop down a gear and pedal faster.
The latter option will (almost) always enable you to progress to the next hill.
The point is, it is not always desirable or efficient to slow down but it is sometimes the necessary choice to progress.

So, as experienced technicians you know what a process is, correct?
Of course you do.

You likely know enough to survive, but do you know the detail required to progress to the next big hill without stalling?
In order to tackle Kubernetes with confidence you will revisit the humble **process** to gain a renewed appreciation how they relate to one another and the "quirky tricks" they are capable of.

## Common misconceptions about processes
There are **two** common, yet significant, misconceptions about processes that need to be dispelled.
They are as follows.

### 1) Processes are always siblings (not true)

Is this screen familiar to you?

![task-manager](/images/process/task-manager.png)

Tools like the [Task Manager](https://en.wikipedia.org/wiki/Task_Manager_(Windows)), the [Activity Monitor](https://en.wikipedia.org/wiki/List_of_macOS_components#Activity_Monitor) or even the [top](https://en.wikipedia.org/wiki/Top_(software)) utility may be partially to blame for this.
Running processes are often **not** siblings.
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
You may recall that you would also need to **restart** your CMD sessions for those changes to be visible from **within** the sessions.
This is because the settings you make via the Environment Variables dialog belong to a top-level process (**explorer.exe**) which represents your desktop environment.
Whenever you create a CMD session, your desktop environment spawns a **child process** which takes a **point-in-time** copy of these environment variables from the originating process and this is why existing sessions require a restart.
At this point, if you restart just **some** CMD sessions but not others then the `env` command will yield inconsistent results.

This behavior is by design and consistent across all popular operating systems, including Linux.
So lets switch our attention to our Cloud9 environment and see some of this in action.

## Hands on with processes

Return to your existing Cloud9 environment and, if necessary, start a new command line session using `Window -> New Terminal`.
Each shell instance is a process like any other and each possesses a [process identifier](https://en.wikipedia.org/wiki/Process_identifier) (PID) which is, ostensibly, unique across any single virtual machine instance.

{{< step >}}You can identify the PID of the current shell as follows.{{< /step >}}
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

The numerical response to `echo $$` is the PID for the **current** shell.
Upon repeat invocations its response is consistent but will differ between any `New Terminal` sessions you create.

{{< step >}}Now check the collection of currently running processes under the current user ID (UID) `ec2-user`.{{< /step >}}
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
- You will observe three processes which collectively belong to three generations of processes. In the above example, PID `6443` is a child of PID `5831` which is a child of PID `5830`.

{{% notice note %}}
The PPID of the current shell can be discovered using `echo $PPID`.
{{% /notice %}}

This can be depicted graphically as follows:

{{< mermaid >}}
graph TB
proc1((PID 5830<br>bash -c ...))
proc2((PID 5831<br>bash -l))
proc3((PID 6443<br>ps -f))
proc1 --> proc2
proc2 --> proc3

classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
class proc1,proc2,proc3 blue;
{{< /mermaid >}}

## Environmental Inheritance

Now you can experiment.
{{< step >}}Create an environment variable and check it was successfully set.{{< /step >}}
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

{{< step >}}Next, create a child process which sticks around a little longer than `ps`.
The easiest way to achieve this is to ask your shell (Bash) to spawn another shell.{{< /step >}}
```bash
bash
```

The new child shell becomes your currently active shell.

{{< step >}}Now inspect your current PID and summarize your running processes.{{< /step >}}
```bash
echo $$
ps -f
```

{{< output >}}
48558
UID          PID    PPID  C STIME TTY          TIME CMD
ec2-user    5830    5829  0 14:26 pts/1    00:00:00 bash -c export ISOUTPUTPANE=0;bash -l
ec2-user    5831    5830  0 14:26 pts/1    00:00:00 bash -l
ec2-user  >48558<   5831  0 15:54 pts/1    00:00:00 bash
ec2-user   48591   48558  0 15:54 pts/1    00:00:00 ps -f
{{< /output >}}

{{% notice note %}}
Observe that the PPID of your current shell matches the PID of its originating shell.
{{% /notice %}}

{{< step >}}Now check that the variable you initialized in the parent process is visible from the child process.{{< /step >}}
```bash
echo ${K8S_PRIMER}
```

{{< output >}}
before
{{< /output >}}

So the variable is visible and set appropriately, but what about making modifications to variables from within the child process?
{{< step >}}Modify the variable.{{< /step >}}
```bash
K8S_PRIMER=after
echo ${K8S_PRIMER}
```

{{< output >}}
after
{{< /output >}}

Also good.

{{< mermaid >}}
graph TB
proc1((PID 5830<br>bash -c ...))
proc2((PID 5831<br>bash -l))
proc3((PID 48558<br>bash))
proc1 --> proc2
proc2 --> proc3
env2>K8S_PRIMER=before]
env3>K8S_PRIMER=after]
proc2 -.->|has env| env2
proc3 -.->|has env| env3

classDef blue fill:#0cf,stroke:#333,stroke-width:4px;
class proc1,proc2,proc3 blue;
{{< /mermaid >}}

So you **may** expect the variable modification, which occurred in the child process, to be visible from within the parent process.
The original shell is still there, in a passive state, patiently waiting for the child to pass control back to it.
{{< step >}}Use `exit` to terminate the child process so the parent regains input from the keyboard and you can **verify** this assertion.{{< /step >}}
```bash
exit                         # head back in the originating shell (PPID)
echo $$                      # back in the original shell
echo ${K8S_PRIMER}
```

{{< output >}}
5831
before
{{< /output >}}

So the modification made in the child process is **not** visible back in the originating parent process.
This is because it **never** was the exact same variable, it just happened to have the same name.
Recall from our earlier discussion (re. Windows CMD sessions) that when a child process is spawned it takes a point-in-time **copy** of the variables.
We have just observed the same behavior in a different operating system.

To be clear, the "copied state" behavior that exists for variables between parent and child processes is observable at **any** depth in the tree and it transpires that this **pattern** is not merely confined to variables.
This allows you to do some pretty amazing things, as you will discover in the next chapter.

## Success

In this exercise you did the following:
- Discussed some common misconceptions
- Revealed the hierarchy of processes
- Put environment variables through their paces to reveal their true nature
