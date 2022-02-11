---
title: "Pipes"
chapter: false
weight: 724
draft: false
---

## Purpose

Sometimes you want to communicate between processes in the same container. Although the [loopback interface]({{< ref "appendix/722_loopback" >}}) could be used to accomplish this, a **pipe** may be more a more appropriate and simpler way to get the job done.

In this lesson, you will:
- explore the nature of **pipes** for communicating between processes.

## Talk to Another Process - Through a Pipe

Another mechanism you can use to communicate between processes is a **pipe**.

{{< step >}}Pipe the output from `man` to be used as the input to `grep`.{{< /step >}}

```bash
man 2 pipe | grep EXAMPLE -A 58
```

{{% notice note %}}
The details of the `pipe` manual page is targeted to *developers*. As this workshop is *not* written for that audience, we have heavily editorialized the output. Feel free to skip ahead to the diagram and narrative after this example output.
{{% /notice %}}

{{< output >}}
EXAMPLE
... description of fork(2), pipe(2), and reader/writer roles ...
... developer "magic" ...
    char* mess = argv[1];    // example in man page just uses argv[1] below
    int pipefd[2];
    pid_t cpid;
    char buf;

    pipe(pipefd);            // request TWO pipe endpoint "file descriptors"
    cpid = fork();           // spawn a child process
    if (cpid == 0) {         // Child reads from pipe
        close(pipefd[1]);    // Close unused write end
        while (read(pipefd[0], &buf, 1) > 0) // this is essentially cat(1)
            write(STDOUT_FILENO, &buf, 1);
        write(STDOUT_FILENO, "\n", 1);
        close(pipefd[0]);
    } else {                 // Parent writes message to pipe
        close(pipefd[0]);    // Close unused read end
        write(pipefd[1], mess, strlen(mess));
        close(pipefd[1]);    // Reader will see EOF
        wait(NULL);          // Wait for child
    }
{{< /output >}}

What does this do? In this case, your shell launches the **parent** process to run this program.
Within this program, that process creates a **pair** of pipe endpoints. 
Then it uses the `fork` system call spawns a child process. 
At the `if`/`else` portion, execution diverges. 
The ***child*** process runs the `if` clause.
The ***parent*** process runs the `else` clause.
In this example, the parent process writes messages to one end of the pipe: the "writer" endpoint.
Similar to a loopback interface, the pipe relays data that is written in one end so it comes out the other.
In this example, the child process reads the message out the other end of the pipe: the "reader" endpoint.

{{< mermaid >}}
graph LR
subgraph cloud9[Cloud 9 dev instance in VPC]
  shell((PID 1582<br>bash))
  parent((PID 1701<br>parent))
  child((PID 1702<br>child))
  pipe1((pipe<br>relays<br>data))
end
shell -.-> parent
parent -.-> child
parent -->|writes| pipe1
pipe1 -->|reads| child
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class shell,parent,child blue2;
class pipe1 cyan;
{{< /mermaid >}}

One of the main jobs of a shell is to do redirection and pipe-fitting. The developer-level details of how shells such as `sh`, `bash`, and `zsh` implement pipes such as discussion of `exec` is beyond the scope of this lesson. However, the conceptual perspective can aid in understanding network communications interconnections. The concepts illustrated with the `parent`/`child` example based on `fork` and `pipe` are sufficient to build on. 

In the actual command line you ran in the previous step: `man 2 pipe | grep EXAMPLE -A 58`, what is the relationship between the shell (assume `bash`), and the processes running `man`, and `grep`? 

{{< mermaid >}}
graph LR
subgraph cloud9[Cloud 9 dev instance in VPC]
  shell((PID 1582<br>bash))
  man((PID 1657<br>man))
  grep((PID 1658<br>grep))
  pipe1((pipe<br>relays<br>data))
end
shell -.-> man
shell -.-> grep
man -->|writes| pipe1
pipe1 -->|reads| grep
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class shell,man,grep blue2;
class pipe1 cyan;
{{< /mermaid >}}

As shown in the diagram, the shell--`bash`--is the parent of both of the other commands. With longer command pipelines composed of three, four, or more commands, the shell is the parent of all the commands in the pipeline. For some number, N, commands, the shell will create one fewer--i.e. N-1--pipes and plumb the pipes together between the sequence of commands.

In the above example, with two commands, `man` and `grep`, N=2. N-1=1, therefore the shell simply creates one pipe as shown in the figure. This is indicated with the dashed lines. Data flow goes from the first command in the pipeline, `man`, into the pipe. The pipe relays the data to the second command, `grep`. This kind of communications has been common in most operating systems for the past several decades. 

The processes within each of your containers can communicate with one another using pipes. You can use other mechanisms for communicating between processes in different containers. Beyond containers, inter-pod, inter-node, and inter-cluster communications would also be based on non-pipe techniques. Nevertheless, similarities between pipes and those other techniques should help your confidence with any communications mechanisms.

A major distinction between the `loopback` and `pipe` mechanisms is that a pipe is unidirectional and dedicated to just two processes. One process writes to the pipe and the other process reads. As network interfaces, loopback interfaces can have many processes communicating back and forth across one loopback interface. Those processes rely on distinct tuples of IP addresses, protocol numbers, and port numbers to uniquely identify their partner endpoints.

## Success

In this lesson, you:
- Explored the nature of **pipes** for communicating between processes.

