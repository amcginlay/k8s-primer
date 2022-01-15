---
title: "The Anatomy Of A Web Server"
chapter: false
weight: 012
draft: false
---

## Introduction

In order to learn Kubernetes, we first need to understand containers and what they are used for.
Containers, in their running state, could encapsulate any type of process however in the [API-first](https://en.wikipedia.org/wiki/Web_API) world of [microservices](https://en.wikipedia.org/wiki/Microservices) a large majority of these are web servers.

We potentially interact with many such web servers every time we pick up a modern smartphone.
It is easy to perceive of web servers as [black boxes](https://en.wikipedia.org/wiki/Black_box), obscured behind decades of ongoing effort to address our modern expectations of scale and security.
We believe this obscurity causes unnecessary obstacles to learning and can can dissuade candidates from pursuing a deeper knowledge of modern software development.

We aim to address this by leveraging concepts you already know to reveal simplicity behind the technology we all use every day.

Feel like you know all this already?
Tempted to fast forward?
Stick around for a moment as the simplicity of what follows might still surprise you.

## Everything is a file

Linux afficionados will relish telling you that "everything is a file".
That may indeed be true, but the advice is of limited help when you are just getting started.
So your next question might be "Could we clarify what a file is?"
In the case of a text file, it is a sequence of characters delimited with line breaks and finshed off with an invisible end-of-file (EOF) marker.

Your **terminal screen** can display a sequence of characters, so is it a file?

Your **keyboard** can emit a sequence of characters, so is it a file?

In both cases Linux says "yes" to these questions and refers to said **dev**ices as the **standard output ("/dev/stdout")** and **standard input ("/dev/stdin")** respectively.

Starting from basic principals, you likely know that the [cat](https://en.wikipedia.org/wiki/Cat_(Unix)) command is typically used to send regular file content to the terminal screen.
Using [output redirection](https://en.wikipedia.org/wiki/Redirection_(computing)) we can show that the following two commands behave identically.

{{% notice note %}}
All commands in snippet boxes, like the one below, should be entered into a Cloud9 terminal session.
{{% /notice %}}
```bash
cat /etc/hosts               # <--- implicit redirection, output will default to /dev/stdout
cat /etc/hosts > /dev/stdout # <--- explicit redirection, same result as above
```

{{% notice note %}}
This [/etc/hosts](https://en.wikipedia.org/wiki/Hosts_(file)) file you just read from reveals the mapping between the DNS name `localhost` and the loopback IP address `127.0.0.1` which typically makes the two terms interchangeable.
{{% /notice %}}

But what happens when you do not provide an argument to the `cat` command?
Try it out and see for yourself.
```bash
cat
```

{{% notice note %}}
It may be revealing to know the previous command is actually identical to `cat < /dev/stdin > /dev/stdout`.
{{% /notice %}}

The solitary `cat` command appears to hang when, in fact, it is just waiting for input from the default device, the keyboard.
So type away, hitting return/enter periodically to see how `cat` consumes lines of characters from the keyboard and sends them to the terminal screen.

When done, hit `Ctrl+D` to regain your command prompt.
This keyboard shortcut represents the invisible [end-of-file (EOF)](https://en.wikipedia.org/wiki/End-of-Transmission_character) marker we discussed earlier.

Before moving on you may also like to observe how, with the help of the `<<<` syntax (known as a [here-string](https://en.wikipedia.org/wiki/Here_document#Here_strings)), you can get Linux to treat a string as a single-line file.
```bash
cat <<< "test"
```

{{% notice note %}}
The above use of `cat` mimics the standard behaviour of the familiar `echo` command.
{{% /notice %}}

## Connecting the cats

Until now, you have been limited to interacting with local files, devices and strings using a single `cat` process.
The [TCP protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) provides a natural extension to what you have observed thus far.
TCP allows you to treat processes as devices which can be interacted with as if they were themselves just files.
With TCP, a pair of processes on a network can communicate with each other in a manner similar to what we have just observed.

For this we need a network enabled version of `cat` known as [netcat](https://en.wikipedia.org/wiki/Netcat) (alternatively `ncat`, or simply `nc`) which we can install as follows.
```bash
sudo yum install -y nc
```

{{% notice note %}}
The following steps will utilse a pair of concurrent Cloud9 terminal sessions.
We shall refer to these sessions as the `SERVER` session and the `CLIENT` session respectively.
New sessions can be created from the Cloud9 menus using `Window -> New Terminal`.
{{% /notice %}}

Start by simply writing to a process, as you would a file.
Observe how the `SERVER` will block until a write operation (i.e. **request**) is received on port 8000.
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

Hit Ctrl+C in the `SERVER` session to break the loop and regain your prompt.

{{% notice note %}}
The `CLIENT` sent four identical requests using different syntax. Observe how, in the final request, `nc` used domain name resolution.
{{% /notice %}}

Now you will observe how, with a small adjustment, the `SERVER` can also perform a read operation (i.e. **response**).
```bash
# SERVER (hit Ctrl+C to break)
while true; do nc --listen 8000 <<< "response"; done

# CLIENT
nc localhost 8000 <<< "request"
```

So now you have a bi-directional, text-based communications channel between two processes.
You are, in fact, halfway towards building a functioning web server but that requires us to observe the [HTTP protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol).
The [curl](https://en.wikipedia.org/wiki/CURL) command can be used to form HTTP requests so, with the `SERVER` still running, give it a spin and see what happens.
```bash
# CLIENT
curl http://localhost:8000
```

The `SERVER` side will output the `curl` request it received which will look something like this.
{{< output >}}
request
GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.79.1
Accept: */*
{{< /output >}}

Meanwhile the `CLIENT` is not so happy, as `curl` expected your `SERVER` to observe the HTTP protocol.
But we can fake it.

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
1. Witnessed how the operating system treats devices like the keyboard and the terminal screen as file devices that can be read from or written to respectively
1. Delved into peculiarities of the `cat` command and how it relates to the `echo` command
1. Witnessed how TCP allows you to also treat running processes as file devices.
1. Used `netcat` (or `ncat`, or `nc`) to create a running process which blocks on a specified port until a valid TCP request is received
1. Built a simple web server to respond to HTTP requests with a static HTML page.
