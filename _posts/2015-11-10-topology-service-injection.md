---
layout: post
author: gsagie
title: Topology Service Injection
date: 2015-11-10 16:25:06 -0700
categories:
- SDN
- NFV
- Openstack
- OVS
- Dragonflow
tags:
- OVS
- SFC
- NFV
- Neutron
- Dragonflow
- Openstack
- Service Chaining
- OVN
---

In this post i am going to describe a concept i call "Topology Service Injection". I am actually
going to give a talk about this next week in [OVS Conference 2015](http://openvswitch.org/support/ovscon2015/) with Liran
Schour from IBM, and describe a proposal to implement this in OVN with some use cases.

We are also working on introducing this concept in Dragonflow roadmap and have some very cool ideas
regarding this, hopefully will be able to expand more in later posts.

## Classic Service Chaining

Before i start describing the concept itself, lets have a quick introduction to what classic service chaining
is all about.

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/top_inj2.jpg" />

Service chaining comes from the physical world where we would have these physical appliances, each has its own
function and capabilities and you would wire them one after the other, so traffic traverses them.

In the virtualized world, where all these appliances are becoming virtualized (NFV, VNF) we have the same
concept where we want traffic to traverse a specific path across these functions.

There are two types of service chains, static chains, which means every packet that enters the chain will
traverse the same route and the same services that are configured in the chain.
And there are dynamic chains, which are able to tag packets with specific headers (NSH, MPLS) and signal
the networking infrastructure to change paths or chains in the middle depending on the function computation result.
This can also be used to signal some information to the next function in the chain.

In SDN implementations, no matter which type of chain you use, the functions them self are treated just like
any other port in the system.
Sure, there might be specific API to configure the chains, but still they are just ports the infrastructure
needs to pass traffic to and from.

## Topology Service Injection

So what is topology service injection?

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/top_inj6.jpg" />

We want to allow external applications to be able to interact with the networking infrastructure forwarding
pipeline and be able to install their logic in specific hooks in that pipeline.

What this means is if we have an OpenFlow pipeline implemented for network connectivity across
the virtual switches in our environment, we want to allow an external application to have specific places where it can
installs its own OpenFlow like flows and have the flexibility to change traffic forwarding decisions.

This means we allow users of networking SDN solutions like OVN, Dragonflow and others to have the ability
to change the static pipeline and have their logic in the packet forwarding stages.

Of course in order to do that we need to consider security.
We basically allow an external application to alter our network infrastructure and we don't
want it to interfere with ALL traffic, just the data that it is designed and deployed for.

In order to address all the above requirements, we came up with the following design:

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/top_inj3.jpg" />

The way we envision Topology Service Injection implementation is for the user/external application to configure and
register themselfs to the network infrastructure with some API and then attach the service to one or a group
of logical elements in that environment.

This means that if for example we have Logical ports connected to Logical Switches connected to Logical Routers, the user
could attach their services to any of these elements (or a group of them), and then only traffic that suppose to pass in these
virtual elements will go through the external application specific logic.

The network infrastructure will then need to compile the correct pipeline and only allow each application to
install its flows in its designated place.

This design allow us to offer topology service injection as a tenant service, meaning that the external applications could be services
deployed by the tenant and he/she will only be able to attach them to its own logical resources.
By enforcing this in the networking infrastructure, we secure one tenant from interfering with traffic of another tenant as
only traffic that goes to the tenant logical elements will also traverse the injected service table logic.

Once the service is attached to a group of logical elements, the infrastructure compile the correct pipeline (in a distributed manner)
and provide a specific API for the external application to be able to install flows in its table.
This API should look like OpenFlow (or be exactly like OpenFlow) to be compatible with SDN applications today
and also must be secure (token based) so that the application could only install flows in its own designated table.

If you come to think about it, this concept is very similar to netfilter/iptables, where you have these pre
define hooks (PRE ROUTE, POST ROUTE, FORWARDING, etc.) and a user can configure its own rules
in each of these locations.

## Use Cases

There are many use cases for topology service injection by its own, but i want to give here an example
that will demonstrate how this feature complement classic service chaining.

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/top_inj4.jpg" />

We have a service chain with an IPS (Intrusion prevention system) virtualized function.
its inspecting traffic coming from a group of tenant VMs in our setup and suddenly identify
that VM1 is infected either with a trojan horse or some other malware and wants to block all of
its traffic.

In the classic service chaining world, the IPS itself would still receive all of the traffic
from VM1 and blocks it when it reaches it.
With topology service injection, we can design it in such a way that the function has its
own table in the pipeline and can install blocking flows that prevents traffic passing from
VM1, basically quarantine VM1 traffic completely.

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/top_inj5.jpg" />

This reduce the load on the function itself as traffic is already blocked on the network infrastructure
it self.
This also fit if we have an IDS system (Intrusion detection system) which is not on the datapath
and receive only mirrored/tapped traffic.
In that case it MUST have this feature in order to block the traffic (as its not in the datapath).

This is a simple use case, but there are many other similar use cases for example:

* DPI system that identify applications and then installs flows to mark them with specific QoS SLA

* A distributed load balancer function that receive the first packets of a connections and then wire the connection
between the client and the server using flows in the pipeline.

There are many more use cases for this, if you are attending OVS conference we can talk more about them
and i plan on writing some more specific description of the other ideas we have
in that area with Dragonflow.


