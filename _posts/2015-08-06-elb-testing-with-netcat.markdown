---
layout: post
title:  "ELB Testing with netcat"
date:   2015-08-06 21:00:00
summary: Test your ELB TCP health check is working as expected with a netcat server.
tags:
- aws
---

Start a quick and dirty tcp server:

```bash
sudo yum install -y nc
nc -kl <port> &
```

*Where -k is for keep alive, -l is for listen and \<port\> is the port to listen on.*

I used this to check the configuration of an AWS ELB using TCP:5000 to health check an EC2 instance before I installed an application.

But you might just want to use it to check a security group and / or ACL surrounding the instance:

```bash
nc <host> <port>
```

*Where \<host\> is the server and \<port\> is the port to connect to.*

The application is a JVM application that runs a discard server on port 5000 specifically for heath checking purposes.
