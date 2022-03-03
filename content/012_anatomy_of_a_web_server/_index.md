---
title: "Anatomy Of A Web Server"
chapter: false
weight: 12
draft: false
---

## Introduction

In order to learn Kubernetes, we first need to understand containers and what they are used for.
In their running state, containers encapsulate processes which may serve a variety of purposes.
In the [API-first](https://en.wikipedia.org/wiki/Web_API) world of [microservices](https://en.wikipedia.org/wiki/Microservices), a large majority of these processes are web servers.

We potentially interact with many such web servers every time we pick up a modern smart device.
It is tempting to view a web server as a metaphorical [black box](https://en.wikipedia.org/wiki/Black_box), obscured behind decades of ongoing effort to address our modern expectations of scale and security.
This obscurity has the potential to dissuade candidates from pursuing a deeper knowledge of modern software development and deployment practices.

You will address this by leveraging concepts you already know.
You will peel back the layers and reveal a hidden simplicity that sits beneath the technologies we all use every day.

Feel like you know all this already?
Tempted to fast forward?
Stick around for a moment as some of what follows might still surprise you.

## Everything is a file

Linux aficionados will relish telling you that "everything is a file" - a nugget of wisdom that is of little use without clear context and/or examples.
A reasonable response might then be to ask, "What exactly is a file?"
In the case of a **text** file, it is a sequence of characters, separated by line breaks and concluded with an invisible end-of-file (EOF) marker.

Your **terminal screen** can display sequences of characters, so is that a file?
The answer is "yes" and Linux refers to this device as the **standard output (/dev/stdout)**.

Your **keyboard** can emit sequences of characters, so is that a file?
The answer is "yes" and Linux refers to this device as the **standard input (/dev/stdin)**.

Constructs of this type are collectively known as [Device Files](https://en.wikipedia.org/wiki/Device_file).

Starting from basic principles, you likely know that the [cat](https://en.wikipedia.org/wiki/Cat_(Unix)) command is commonly used to dump the lines of a text file to the terminal screen.
Using [output redirection](https://en.wikipedia.org/wiki/Redirection_(computing)) we can show that the following two commands behave identically.

{{% notice note %}}
All commands in snippet boxes, like the one below, should be entered into a Cloud9 terminal session.
{{% /notice %}}
```bash
cat /etc/hosts               # <--- without output redirection, file contents will default to /dev/stdout
cat /etc/hosts > /dev/stdout # <--- with output redirection, causing the exact same result
```

{{% notice note %}}
The [/etc/hosts](https://en.wikipedia.org/wiki/Hosts_(file)) file comes as standard in Linux distributions.
It reveals the mapping between the hostname [`localhost`](https://en.wikipedia.org/wiki/Localhost) and the loopback IP address `127.0.0.1` which makes the two terms interchangeable.
{{% /notice %}}

But what happens when you do not provide an argument to the `cat` command?
Try it out and see for yourself.
```bash
cat
```

{{% notice note %}}
It may be revealing to know the previous command is actually identical to `cat < /dev/stdin > /dev/stdout` in which the input <ins> **and** </ins> output to `cat` are **both** redirected.
{{% /notice %}}

The solitary `cat` command appears to hang when, in fact, it is just blocked, waiting for input from `/dev/stdin`.
So type away, hitting return/enter periodically to see how `cat` consumes lines from the keyboard and sends them to the terminal screen.

When done, hit `Ctrl+D` to regain your command prompt.
This keyboard shortcut represents the invisible [end-of-file (EOF)](https://en.wikipedia.org/wiki/End-of-Transmission_character) marker we discussed earlier.

Before moving on you may also like to observe how, with the help of the `<<<` syntax (known as a [here-string](https://en.wikipedia.org/wiki/Here_document#Here_strings)), you can get Linux to treat a string as an anonymous single-line text file.
```bash
cat <<< "test"
```

{{% notice note %}}
Using a here-string causes `cat` to mimic the standard behavior of the familiar `echo` command.
{{% /notice %}}

## Herding cats

Until now, you have been limited to using a single `cat` process to interact with strings, local files and device files.
The [TCP protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) allows you to use pairs of network-enabled processes to communicate with each other as it they were themselves device files.

To see this in action we need a network enabled version of `cat` known as [netcat](https://en.wikipedia.org/wiki/Netcat) (alternatively `ncat`, or simply `nc`) which you can install on your Cloud9 instance as follows.
```bash
sudo yum install -y nc
```

The following steps will utilise a **pair** of concurrent Cloud9 terminal sessions (**shell one** and **shell two**) which can be dragged into a side-by-side orientation as shown in the following screenshot.

![shell-one-two](/images/process/shell-one-two.png)

TCP-enabled processes are accessible as device files at `/dev/tcp` and utilizing them is no more complex than before.
Observe how `nc` sits down on port 8000 and will, much like a solitary `cat` command, block until a single write operation (i.e. **request**) is received.
{{< columns >}}
{{% column %}}
```bash
# SERVER (in shell one)
nc --listen 8000
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT (in shell two)
cat <<< "request" > /dev/tcp/127.0.0.1/8000
```
{{% /column %}}
{{< /columns >}}

Which produces.
{{< columns >}}
{{< column >}}
{{< output >}}
request
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
{{< /output >}}
{{< /column >}}
{{< /columns >}}

{{% notice note %}}
The empty output on the `CLIENT` side is expected.
It means everything worked and nothing untoward happened.
In the world of Linux, **no news is usually good news!**
{{% /notice %}}


Do that again, this time putting the `SERVER` into a loop so it can handle subsequent requests.
{{< columns >}}
{{% column %}}
```bash
# SERVER (hit Ctrl+C to break)
while true; do nc --listen 8000; done
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
cat <<< "request 1" > /dev/tcp/127.0.0.1/8000
echo "request 2" > /dev/tcp/127.0.0.1/8000
nc 127.0.0.1 8000 <<< "request 3"
nc localhost 8000 <<< "request 4"
```
{{% /column %}}
{{< /columns >}}

Which produces.
{{< columns >}}
{{< column >}}
{{< output >}}
request 1
request 2
request 3
request 4
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
{{< /output >}}
{{< /column >}}
{{< /columns >}}

{{% notice note %}}
Observe how `nc`, which is used as the request `--listen`ing process in the `SERVER` session, was repurposed as the request **transmitting** process in the `CLIENT` session.
The `CLIENT` sent **four identical requests** each using a different syntax. 
In the final request, `nc` was able to resolve the IP address for the hostname `localhost`.
{{% /notice %}}

Hit [`Ctrl+C`](https://en.wikipedia.org/wiki/End-of-Text_character) in the `SERVER` session to break the loop and regain your prompt.

You have just seen how the `CLIENT` can perform write operations against the `SERVER`.
Now observe how, with a small adjustment, the `SERVER` can be configured to write back a response. 
{{< columns >}}
{{% column %}}
```bash
# SERVER (... leave this running)
while true; do nc --listen 8000 <<< "response"; done
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
nc localhost 8000 <<< "request"
```
{{% /column %}}
{{< /columns >}}

Which produces.
{{< columns >}}
{{< column >}}
{{< output >}}
request
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
response
{{< /output >}}
{{< /column >}}
{{< /columns >}}

You now have a bi-directional, text-based communications channel between two processes.
You are halfway towards building a functioning web server, but that requires us to observe the [HTTP protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol).
The [curl](https://en.wikipedia.org/wiki/CURL) command can be used to transmit HTTP requests so, with the `SERVER` still running, give it a spin in the `CLIENT` and see what happens.

{{< columns >}}
{{% column %}}
```bash
# SERVER (no action required)
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
curl http://localhost:8000
```
{{% /column %}}
{{< /columns >}}

Which produces.
{{< columns >}}
{{< column >}}
{{< output >}}
GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.79.1
Accept: */*
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
curl: (1) Received HTTP/0.9 when not allowed
{{< /output >}}
{{< /column >}}
{{< /columns >}}

The `SERVER` side dumped the plain-text lines transmitted to it by the `curl` command on the `CLIENT` side.

On the `CLIENT` side, the `curl` command was expecting the `SERVER` to abide by the rules of the HTTP protocol and respond accordingly.
As this was not the case `curl` **errored** but, since the protocol is text-based, we can fabricate a compatible response and try again.

Hit [`Ctrl+C`](https://en.wikipedia.org/wiki/End-of-Text_character) in the `SERVER` session to break the loop and try the following.
{{< columns >}}
{{% column %}}
```bash
# SERVER (hit Ctrl+C to break)
while true; do nc --listen 8000 <<< "HTTP/1.1 200 OK"; done
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
curl http://localhost:8000
```
{{% /column %}}
{{< /columns >}}

Which produces.
{{< columns >}}
{{< column >}}
{{< output >}}
GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.79.1
Accept: */*
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
{{< /output >}}
{{< /column >}}
{{< /columns >}}

It may not be all that exciting but this is our minimum viable web server.
There is no longer a failure in the `curl` command (remember, no news is good news!) but there is also no tangible response **beyond** the successful receipt of a `200` [status code](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes#2xx_success) in the [HTTP header](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields).

{{% notice note %}}
Pass the `--verbose` switch to `curl` if you wish to see proof that the 200 status code was consumed by the `CLIENT`.
{{% /notice %}}

Your `CLIENT` (i.e. curl) expects web servers to provide a response in the form of an [HTTP body](https://en.wikipedia.org/wiki/HTTP_message_body) so try sending back a multi-line response.
{{< columns >}}
{{% column %}}
```bash
# SERVER (hit Ctrl+C to break)
while true
do nc --listen 8000 << EOF
HTTP/1.1 200 OK

multi-line
response
EOF
done
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
curl http://localhost:8000
```
{{% /column %}}
{{< /columns >}}

{{% notice warning %}}
Do not remove the empty line between the lines which read `HTTP/1.1 200 OK` and `multi-line`.
The empty line signifies the separation of header and body, which is a requirement in the HTTP protocol.
{{% /notice %}}

Now the `curl` command on the `CLIENT` will output the body of the response.
{{< columns >}}
{{< column >}}
{{< output >}}
GET / HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.79.1
Accept: */*
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
multi-line
response
{{< /output >}}
{{< /column >}}
{{< /columns >}}

## Deploy a simple web server

Finally, to show how a a web server would would serve up a single static web page in the wild, try the following.
{{< columns >}}
{{% column %}}
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
```
{{% /column %}}
{{% column %}}
```bash
# CLIENT
curl http://localhost:8000/tada.html
```
{{% /column %}}
{{< /columns >}}

Which produces.
{{< columns >}}
{{< column >}}
{{< output >}}
GET /tada.html HTTP/1.1
Host: localhost:8000
User-Agent: curl/7.79.1
Accept: */*
{{< /output >}}
{{< /column >}}
{{< column >}}
{{< output >}}
<!doctype html>
<html><body><h1>You have just created a web server!</h1></body></html>
{{< /output >}}
{{< /column >}}
{{< /columns >}}

## Web Server Quiz

Please take the following quiz to review your knowledge of web servers.

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## Why Web Servers?

---
shuffle_answers: false
---

How are web servers relevant to containers and Kubernetes?

> Choose as many as you think apply.

- [x] some containers host web sites with content based on html, png, css, javascript, etc.
- [x] microservices are most often accessed via a web API based on XML, JSON, or other encoding
- [x] health checks of containers are usually based on web queries
- [x] metrics are often scraped from a container using a web endpoint

## Web Client

Which tool did you use in this lesson for submitting web requests?

> What is the role (client/server) of this tool, and what is a web locator called?

- [ ] `Invoke-WebRequest`
- [ ] `Chrome`
- [x] `curl`
- [ ] `wget`

## Web Server

What software did you use in this lesson to provide a fundamental web server role?

> Even more fundamental than what you were thinking.

- [ ] `tomcat`
- [ ] `apache`
- [x] `nc`
- [ ] `nginx`

## Magic?

Are web servers magical?

> What did you just do in this lesson?

- [ ] Yes, web servers are magical
- [ ] No, but web servers must be extremely complicated
- [x] No, web servers could be either simple or complex
- [ ] I don't know

{{< /quizdown >}}

## Success

In this exercise you did the following:

1. Investigated the structure of a regular text file
1. Witnessed how the operating system can treat physical devices like the keyboard and the terminal screen as device files that can be read from or written to respectively
1. Delved into how the `cat` command can utilize local device files
1. Saw how the TCP protocol allows you to also treat running processes as device files.
1. Used `netcat` (or `ncat`, or `nc`) to create a running process which blocks on a specified port until a valid TCP request is received
1. Built a simple web server for serving a static HTML page.
