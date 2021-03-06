---
layout: post
author: gsagie
title: Containers Support In OVN
date: 2015-04-26 16:25:06 -0700
categories:
- SDN
- Openstack
- OVS
tags:
- OVS
- OVN
- Openstack
- Docker
- Containers
---

In the previous OVN post i have written about OVN high level architecture, the northbound and southbound DB’s and their main motivations.
You can read this post [here.](http://http://galsagie.github.io/sdn/openstack/ovs/2015/04/20/ovn-1/)
In this post i am going to describe the containers networking support in OVN and how this integrate with Openstack.

Its first important to understand the OVN Integration with Openstack Neutron, if you were reading the previous post i gave a link to a very good post describing that, but if you didnt i will try to fill it up quickly.

Basically the OVN plugin needs to convert neutron virtual networking model to fit into the OVN Northbound DB schema, the plugin uses OVSDB transactions to update the Northbound DB which is also monitored by ovn-northd daemon (which translate this into the Southbound DB).
This translation has its challenges, but we will get deeper into them with future posts, we will see some of the solutions used to support the containers modeling in this post.

Neutron also has to signal when port is up, meaning that OVN finished all its binding tasks and raised the logical port status to up.

The following diagram depicts OVN architecture and the integration with Openstack (Note that L3 design
is not yet formalized in OVN)

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/ovn-neutron.jpg" />

## Containers Modes

There are currently two ways to configure containers networks in OVN, the first way is trivial, it treats a container just like it would treat a VM, The containers management system/plugin creates a port in neutron and attach it to a network , this translate to a logical port connected to a logical switch in OVN (where Neutron returns back the IP and MAC of the container).

However, Isolation between containers is weaker than isolation between VMs, so some environments deploy containers for different tenants in separate VMs as an additional security measure.
This leads to the second way to configure containers that is supported by OVN.

OVN has special support for running containers inside a VM, with this mode you can attach the inside containers to any logical network managed by OVN, this logical network can contain VM’s, containers or physical machines as endpoints.
Since the use case of running containers inside a VM is very common, this approach reduce complexity and improve performance as another overlay is not needed.

## Deploying Containers Lifecycle

In order to better explain this use case, i will describe what i consider a common “lifecycle” of creating containers inside a VM and connecting them to logical networks.

* Openstack tenant creates a VM that suppose to host containers, the VM has a single network interface that is connected to a logical management network ( used by the containers management system/plugin)

* The VM is assigned with an IP, which is used to spawn containers (either manually or through container orchestration systems) inside that VM and to monitor the health of the created containers.

* The vif-id associated with the VM's network interface can be obtained by making a call to Neutron using tenant credentials. (is used later to create container ports by neutron API)

* The container management system/plugin that wants to deploy a container in that VM now needs to assign it a specific VLAN (only unique inside that VM) and create a port to it using Neutron API (which adds a logical port in OVN).
Then just connecting this port to any desired network.
(The VLAN tag is stripped out in the hypervisor by OVN and is only useful as a context for OVN)

* There are special configuration to OVN that needs to be applied in order to do the above scenario, the next section describe them.

## Configuration in Neutron and OVN

In OVN in order to create a container logical port, you must specify the parent id of this port (meaning the vif-id of the hosting VM logical port) and a tag field (VLAN to use for this container)

These two attributes are not currently supported in the Neutron API. As a result, we are initially allowing these attributes to be set in the 'binding:profile' extension for ports. 
In the future changing the Neutron API to support this natively might be introduced.

‘binding:profile’ extension is a dictionary that enables the application running on the specified host to pass and receive vif port-specific information to the plug-in.

With this support the containers management system/plugin or the tenant itself can define a container port using current Neutron API’s
You can see an example of this [here.](https://review.openstack.org/#/c/176491/3/doc/source/containers.rst)

I would like to thank Russell Bryant for proposing and implementing this elegant and simple solution, it enables Openstack users to utilize this use case in OVN.

The following diagram depicts OVN support for containers inside a VM:

<img src="https://raw.githubusercontent.com/GalSagie/GalSagie.github.io/master/public/img/ovn-containers.jpg" />

There is some great progress going on in OVN and with the Openstack integration, in next posts i will describe the DB tables and logic deeper.

