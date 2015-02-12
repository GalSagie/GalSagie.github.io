---
layout: post
author: gsagie
title: Docker Networking
date: 2015-02-12 16:25:06 -0700
categories:
- SDN
- Docker
tags:
- SDN
- Docker
- Socketplane
- OVS
- Containers
---

In this post i am going to introduce docker networking solutions.
I was already experimenting and playing with some new solutions beta’s and open source when i found others were already doing the same thing.
I will try to keep this short with my conclusions and link you to other good content that i saw, dont see a reason to re-write good content.

### What are Containers?

You probably heard about containers, if not here is a quick introduction.
Containers, at least the standard ones (like LXC) have been here for quite some time, the reason why there is a big buzz around them right now is because of the way we deploy applications in the cloud and particularly virtualized environments.

Applications running on these platforms consist of N-tiers (usually the famous web-app-db tiers) each tier is usually a cluster of VM’s and each VM is used for one particular objective.
This approach causes a lot of overhead as each VM also comes with the entire operating system, this overhead ratio increases when each VM is only used to run one application. (common scenario in cloud environments).

As James Bottomley, Parallels‘ CTO of server virtualization said:
“VM hypervisors, such as Hyper-V, KVM, and Xen, all are based on emulating virtual hardware. That means they’re fat in terms of system requirements. Containers, however, use shared operating systems.

 That means they are much more efficient than hypervisors in system resource terms. Instead of virtualizing hardware, containers rest on top of a single Linux instance. This in turn means you can leave behind the useless 99.9% VM junk, leaving you with a small, neat capsule containing your application”

Dockers for instance, also has an efficient way to reuse libraries between containers on the same host, the following diagram illustrate the difference between Virtual Machines and containers:

<img src="http://zdnet3.cbsistatic.com/hub/i/r/2014/10/02/1f130129-49e2-11e4-b6a0-d4ae52e95e57/resize/770x578/3f83f67acfa33fe05865373b2b4b71dd/docker-vm-container.png" />

### What is Docker ?

“Docker is an open platform for developers and sysadmins to build, ship, and run distributed applications.” (taken from Docker site).

You can visit the official [Docker site here.](https://www.docker.com/whatisdocker/)

Docker is one of the more hyped and used “Container platform”.
The starting idea behind it, the way i see it anyway, was to standardize the way containers are used, i think it is becoming much more than that.

You can visit [this blog post](http://blog.scottlowe.org/2014/03/11/a-quick-introduction-to-docker/) to learn more about Docker.

One interesting part is how Docker, and containers in general, fits into the SDN model, i plan to bring some networking solutions that try to tackle this very hot and interesting subject.
In this post i am introducing socketplane.io

### Socketplane.io

Socketplane goal is to provide a native Software Defined Networking solution for Docker.
It leverage OpenVSwitch and uses overlay VXLAN tunnels between containers endpoints (hypervisors or virtual machines hosting containers).

Socketplane uses multicast DNS to allow zero configuration approach, this means that each host or virtual machine residing on the same segment can find Socketplane cluster and join it automatically without configuration.
(I do wonder how this fits with public DNS servers sitting in the data center and how it works for multi data center deployments)

The next step would be to create VXLAN tunnels between all these end points and create the network topology (again without the need to explicity configure the end points).

Socketplane uses Consul open source agent to achieve the above and also to use its distributed key/value store to synchronize networking configuration between the cluster points.

Consul is an open source project i intend to write about in future posts, its a platform for service discovery which also provide distributed key/value store (similar to ETCD which i wrote about).
It uses a gossip protocol agent (Serf) to communicate events between the cluster nodes.
The nice thing about it, is it also has a DNS implementation which is used for the above discovery.
You can read about Consul in the [project website.](https://www.consul.io)

The following diagram depicts SocketPlane architecture quite well:

<img src="https://aucouranton.files.wordpress.com/2015/01/socketplane-arch.png"/>

Socketplane is open source, and they have created a very nice vagrant demo, you can download and play with it [here.](https://github.com/socketplane/socketplane)
And also see [this video](https://www.youtube.com/watch?feature=player_embedded&v=ukITRl58ntg) explaining the demo.

I was experimenting with Socketplane technology and playing with their demo, i personally think its a project we should all keep an eye on.
I was starting to write about my own experiments and then i found this great post which pretty much state everything i wanted to write, you should probably continue with it to get a better feel for Socketplane technology:

[Docker Virtual Networking with Socketplane.io](http://aucouranton.com/2015/01/16/docker-virtual-networking-with-socketplane-io/)

The group at Socketplane are experienced open source veterans and i am waiting to see how this project progress..
In the next post i will be introducing another networking solutions for docker called Weave.

