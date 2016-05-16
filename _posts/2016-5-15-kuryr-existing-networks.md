---
layout: post
author: gsagie
title: Kuryr and Neutron Existing Resources
date: 2016-5-15 16:25:06 -0700
categories:
- SDN
- Openstack
- Docker
- Kuryr
- Neutron
tags:
- OpenStack
- Kuryr
- Docker
---

If you don`t know what Kuryr is by now, shame on you!  please check out
our [OpenStack Tokyo Kuryr Introduction talk](https://www.openstack.org/videos/video/tokyo-2474)
or read [my blog post](http://galsagie.github.io/2015/08/24/kuryr-part1/) about it.

# Kuryr Current Status

Mitaka was the first actual release that we fully worked on Kuryr and i wanted
to describe some of the nice features that we support in our current version.

In this post i will describe the ability to attach elements to existing Neutron/OpenStack resources
which is useful for many different use cases, in the next part i will describe our Kubernetes
integration and nested containers/Magnum support which are two really nice features
we have in late progress stages.

The following diagram depicts the current components we have in Kuryr:

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/kuryr_components.jpg" />

As you can see Kuryr already has full integration working with Docker libnetwork and
with Docker Swarm as a by product of the above.
What we essentially do is implementing a libnetwork remote driver and IPAM driver that maps
the Docker calls into Neutron resources and model.

This works well and as user creates new resources like networks in Docker these networks are created
in Neutron on the fly and Kuryr keeps the mapping between the two entities.
However, we have noticed that we can provide much more by allowing users to attach their newly created
Docker network into an existing Neutron network.

# The Use Cases

## Connecting VMs, Containers and Bare Metal

Allowing to attach into existing networks enable this very useful ability which users really
need and want.
The ability to connect VMs, containers and bare metal servers to the same virtual network in
a seamless management experience and have a consistent networking for all the three.

There are many reasons why you want to do that and there is no real good reason why you
should`nt be able to do it, connecting your OpenStack and containers workloads together
and applying a unified security, isolation and policy profiles on them.

This simple feature in Kuryr allow all of the above using Neutron as we should soon see.

## Bulk creations

Another use case that i hear a lot about are users that wants to deploy batches
of containers and want to do it as fast as possible.
Sometimes with many different networks and newly created ports.

What we noticed is that the API calls to Neutron in these cases can be time consuming
and slow down this process, this is a common problem for any networking plugin
implementation.

By pre-allocating networks/ports in Neutron and then only binding these elements
from their pools when the user needs (when the user uses the Docker API or any other
orchestration/management tool that does) we are able to improve this process
quite significantly.

Attaching to existing Neutron resources feature enable us to provide and achieve this.

# The Flow

The following steps demonstrate how to use this feature with Kuryr.
First we create a Docker network using Docker CLI API and specify to Docker to use
Kuryr as the networking driver and IPAM driver.

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/kuryr_tags1.jpg" />

If we look at Neutron, we can see that a new network was created with "kuryr" in its prefix

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/kuryr_tags2.jpg" />

We can see that Kuryr save the mapping between Neutron network id and Docker network id using
a new feature in Neutron called resource tags.
This is a feature that was designed by the Kuryr team and implemented in Neutron exactly for
a use case like this (and many others as described in the spec).

You can read more about [this feature in this document.](http://docs.openstack.org/developer/neutron/devref/tag.html)

What we see above is that the user creates a network in Docker and it is automatically
added to Neutron by Kuryr with the appropriate mapping, any new container that is
added to this Docker network will be attached to the same Neutron network.

But what if there is already a Neutron network that already has few VMs attached to it.
The user can specify the network name or network id as a Docker options.
Kuryr instead of creating a new Neutron network will attach the containers to the existing
network.

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/kuryr_tags3.jpg" />

We can see that Kuryr keeps a special tag on the network to indicate that this is not a network
created by Kuryr.
This is important as we dont want to delete this network when the user delete the Docker network, just
remove the associations (tha mapping tags on the Neutron network).

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/kuryr_tags4.jpg" />

# Future

Currently, Kuryr only support attaching to existing networks from one Docker control plane, for example
only from one environment of Docker Swarm.
However, we see strong use cases that users may want to connect containers to the same network from two
different Docker Swarm instances for example.

Enabling this is rather simple with the above feature, we will just need to manage tags per a Docker
Swarm environment, this is something we are looking to support really soon.

# Summary

I think this post demonstrate the power of Kuryr and how we can help and simplify life for users
that deploy OpenStack workloads mixed with containers workloads in a very easy manner.

In the next posts i am going to describe other exciting features we are working on like Kubernetes
Integration and Nested containers support.

If you want to take part of our journey feel free to reach out to me or join our IRC channel
in freenode (#openstack-kuryr)