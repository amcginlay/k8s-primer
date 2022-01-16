---
title: "The Anatomy Of A Web Server"
chapter: false
weight: 012
draft: false
---

## Introduction

In order to learn Kubernetes, we first need to understand containers and what they are used for.
Containers, in their running state, encapsulate processes serving many purposes however, in the [API-first](https://en.wikipedia.org/wiki/Web_API) world of [microservices](https://en.wikipedia.org/wiki/Microservices), a large majority of these processes are web servers.

We potentially interact with many such web servers every time we pick up a modern smartphone.
It is tempting to view a web server as a metaphorical [black box](https://en.wikipedia.org/wiki/Black_box), obscured behind decades of ongoing effort to address our modern expectations of scale and security.
This obscurity has the potential to dissuade candidates from pursuing a deeper knowledge of modern software development and deployment.

You will address this by leveraging concepts you already know to help peel back the layers and reveal a hidden simplicity behind the technologies we all use every day.

Feel like you know all this already?
Tempted to fast forward?
Stick around for a moment as some of what follows might still surprise you.

## Everything is a file

Linux afficionados will relish telling you that "everything is a file".
That nugget of wisdom is of limited use without a clear explanation.
So a reasonable response might then be to question, "What exactly is a file?"
In the case of a text file, it is a sequence of characters delimited with line breaks and concluded with an invisible end-of-file (EOF) marker.

Your **terminal screen** can display a lines of characters, so is that a file?

Your **keyboard** can emit lines of characters, so is that a file?

In both cases Linux says the answer is "yes" and refers to said **dev**ices as the **standard output (/dev/stdout)** and **standard input (/dev/stdin)** respectively.
Constructs of this type are collectively known as [Device Files](https://en.wikipedia.org/wiki/Device_file).

Starting from basic principals, you likely know that the [cat](https://en.wikipedia.org/wiki/Cat_(Unix)) command is commonly used to send regular file content to the terminal screen.
Using [output redirection](https://en.wikipedia.org/wiki/Redirection_(computing)) we can show that the following two commands behave identically.

{{% notice note %}}
All commands in snippet boxes, like the one below, should be entered into a Cloud9 terminal session.
{{% /notice %}}
```bash
cat /etc/hosts               # <--- without output redirection, file contents will default to /dev/stdout
cat /etc/hosts > /dev/stdout # <--- with output redirection, causing the exact same result
```

{{% notice note %}}
The [/etc/hosts](https://en.wikipedia.org/wiki/Hosts_(file)) file you just read from reveals the mapping between the hostname [`localhost`](https://en.wikipedia.org/wiki/Localhost) and the loopback IP address `127.0.0.1` which makes the two terms interchangeable.
{{% /notice %}}

But what happens when you do not provide an argument to the `cat` command?
Try it out and see for yourself.
```bash
cat
```

{{% notice note %}}
It may be revealing to know the previous command is actually identical to `cat < /dev/stdin > /dev/stdout` in which both the input and output to `cat` are redirected.
{{% /notice %}}

The solitary `cat` command appears to hang when, in fact, it is just blocked, waiting for input from `/dev/stdin`.
So type away, hitting return/enter periodically to see how `cat` consumes lines from the keyboard and sends them to the terminal screen.

When done, hit `Ctrl+D` to regain your command prompt.
This keyboard shortcut represents the invisible [end-of-file (EOF)](https://en.wikipedia.org/wiki/End-of-Transmission_character) marker we discussed earlier.

Before moving on you may also like to observe how, with the help of the `<<<` syntax (known as a [here-string](https://en.wikipedia.org/wiki/Here_document#Here_strings)), you can get Linux to treat a string as an anonymous single-line file.
```bash
cat <<< "test"
```

{{% notice note %}}
Using a here-string in this way causes `cat` to mimics the standard behaviour of the familiar `echo` command.
{{% /notice %}}

## Herding cats

Until now, you have been limited to using a single `cat` process to interact with strings, local files and device files.
The [TCP protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) allows you to use pairs of network-enabled processes to communicate with each other as it they were themselves device files.

To see this in action we need a network enabled version of `cat` known as [netcat](https://en.wikipedia.org/wiki/Netcat) (alternatively `ncat`, or simply `nc`) which you can install on your Cloud9 instance as follows.
```bash
sudo yum install -y nc
```

{{% notice note %}}
The following steps will utilise a pair of concurrent Cloud9 terminal sessions.
We shall refer to these sessions as the `SERVER` session and the `CLIENT` session respectively.
New sessions can be created from the Cloud9 menus using `Window -> New Terminal`.
{{% /notice %}}

TCP-enabled processes are accessible as device files at `/dev/tcp` and utilising them is no more complex than before.
Observe how `nc` sits down on port 8000 and will, much like a solitary `cat` command, block until a single write operation (i.e. **request**) is received.
```bash
# SERVER
nc --listen 8000

# CLIENT
cat <<< "request" > /dev/tcp/127.0.0.1/8000
```

Do that again, this time putting the server into a loop so it can handle subsequent requests.
```bash
# SERVER (hit Ctrl+C to break)
while true; do nc --listen 8000; done

# CLIENT
cat <<< "request 1" > /dev/tcp/127.0.0.1/8000
echo "request 2" > /dev/tcp/127.0.0.1/8000
nc 127.0.0.1 8000 <<< "request 3"
nc localhost 8000 <<< "request 4"
```

{{% notice note %}}
Observe how `nc`, which is used as the listening process in the `SERVER` session, can be repurposed as the transmitter of requests in the `CLIENT` session.
The `CLIENT` sent four identical requests each using a different syntax. 
In the final request, `nc` was able to resolve the IP address for the hostname `localhost`.
{{% /notice %}}

Hit Ctrl+C in the `SERVER` session to break the loop and regain your prompt.

You have just seen how the `CLIENT` can perform write operations against the `SERVER`.
Now observe how, with a small adjustment, the `SERVER` can be configured to write back a response. 
```bash
# SERVER (hit Ctrl+C to break)
while true; do nc --listen 8000 <<< "response"; done

# CLIENT
nc localhost 8000 <<< "request"
```

You now have a bi-directional, text-based communications channel between two processes.
You are halfway towards building a functioning web server but that requires us to observe the [HTTP protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol).
The [curl](https://en.wikipedia.org/wiki/CURL) command can be used to form HTTP requests so, with the `SERVER` still running, give it a spin and see what happens.
```bash
# CLIENT
curl http://localhost:8000
```

The `SERVER` side will output the lines written to it by the `curl` command which will look something like this.
{{< output >}}
request
GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.79.1
Accept: */*
{{< /output >}}

On the `CLIENT` side, the `curl` command expected the `SERVER` to reciprocate using the HTTP protocol.
It did not and `curl` errored as a result, but we can fake it.

Hit Ctrl+C in the `SERVER` session to break the loop and try the following.
```bash
# SERVER (hit Ctrl+C to break)
while true; do nc --listen 8000 <<< "HTTP/1.1 200 OK"; done

# CLIENT
curl http://localhost:8000
```

Great, we have our minimum viable web server.
There is no longer a failure in `curl` but there is also no tangible response.
The HTTP protocol requires the `SERVER` to do a little more work so, building on what we had before, try providing a multi-line response to send back to the `CLIENT`.
```bash
# SERVER (hit Ctrl+C to break)
while true
do nc --listen 8000 << EOF
HTTP/1.1 200 OK

response
EOF
done

# CLIENT
curl http://localhost:8000
```

{{% notice warning %}}
Do not remove the empty line before the word "response".
It is a requirement of the HTTP protocol.
{{% /notice %}}

## Deploy a simple web server

Then, finally, to show how a web server would truly behave in the wild.
```bash
# SERVER (hit Ctrl+C to break)
while true
do nc --listen 8000 << EOF
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Server: nc

<!doctype html>
<html><body><h1>You have just created a web server!</h1></body></html>
EOF
done

# CLIENT
curl http://localhost:8000
```

## Success

In this exercise you did the following:

1. Investigated the structure of a regular text file
1. Witnessed how the operating system can treat physical devices like the keyboard and the terminal screen as device files that can be read from or written to respectively
1. Delved into how the `cat` command can utilise local device files
1. Saw how the TCP protocol allows you to also treat running processes as device files.
1. Used `netcat` (or `ncat`, or `nc`) to create a running process which blocks on a specified port until a valid TCP request is received
1. Built a simple web server for serving a static HTML page.
