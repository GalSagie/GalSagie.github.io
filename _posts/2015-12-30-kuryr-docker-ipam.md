---
layout: post
author: gsagie
title: Kuryr Support for Docker Pluggable IPAM
date: 2015-12-30 16:25:06 -0700
categories:
- SDN
- Openstack
- Docker
- Kuryr
- Neutron
tags:
- OVS
- OpenStack
- Kuryr
- Docker
- Neutron
---

Recently a pluggable IPAM implementation was merged into Docker Libnetwork.
Libnetwork has a default, built-in IPAM driver and allows third party IPAM drivers to be dynamically plugged.
On network creation, the user can specify which IPAM driver libnetwork needs to use for the network's IP address management.

In order to comply with this requirement we added full support for this feature in Kuryr.
Kuryr, register itself to Docker as the IPAM driver and completely implements Docker IPAM API
and map it to Neutron APIs.

In this document i am going to explain the mapping we did in Kuryr in order to support it.

Any Docker IPAM driver must support the following API:

```
type Ipam interface {

    GetDefaultAddressSpaces() (string, string, error)

    RequestPool(addressSpace, pool, subPool string, options map[string]string, v6 bool) (string, *net.IPNet, map[string]string, error)

    ReleasePool(poolID string) error

    RequestAddress(string, net.IP, map[string]string) (*net.IPNet, map[string]string, error)

    ReleaseAddress(string, net.IP) error
}
```

# Docker and Kuryr Implementation of the API

## Flow

### On network creation

* Request the address pool and pass the options along via RequestPool().
* Request the network gateway address if specified. Otherwise request any address from the pool to be used as network gateway. This is done via RequestAddress().
* Request each of the specified auxiliary addresses via RequestAddress().

### On network deletion

* Release the network gateway address via ReleaseAddress()
* Release each of the auxiliary addresses via ReleaseAddress()
* Release the pool via ReleasePool()

### On endpoint creation

* Request an IPv4 address from the IPv4 pool and assign it to the endpoint interface IPv4 address. If successful, stop iterating.
* Request an IPv6 address from the IPv6 pool (if exists) and assign it to the endpoint interface IPv6 address. If successful, stop iterating.

### On endpoint deletion

* Release the endpoint interface IPv4 address
* Release the endpoint interface IPv6 address if present

I will now describe each interface and how we map it to Neutron API in Kuryr

## GetDefaultAddressSpaces()

### Docker

GetDefaultAddressSpaces returns the default local and global address space names for this IPAM.
An address space is a set of non-overlapping address pools isolated from other address spaces' pools.
In other words, same pool can exist on N different address spaces
In libnetwork, the meaning associated to local or global address space is that a local address space
doesn't need to get synchronized across the cluster whereas the global address spaces does.

### Neutron

We map address spaces to [Neutron Address Scopes.](http://specs.openstack.org/openstack/neutron-specs/specs/liberty/address-scopes.html)
An AddressScope can be associated with multiple SubnetPool objects in a one-to-many relationship.
This will allow delegating parts of an address scope to different tenants.
The SubnetPools under an AddressScope must not overlap.
They must also be from the same address family.

## RequestPool / ReleasePool

### Docker

Request pool API takes as parameters the address space (the only mandatory field) and other three
optional parameters, the pool range in CIDR format, a possible subset of subpools in CIDR format, an
options dictionary which is IPAM driver specific and rather or not this is IPv6 pool or not.

Release just removes the pool.

### Neutron

In Neutron we map these pools to Neutron subnet pools.
Subnet pools is a feature that allows Neutron to track and allocate subnets from a larger available address space.
Of course that we allow the user to pre-allocate these subnets pools and re-use an already configured
pools, so we dont re-allocate them.

## RequestAddress / ReleaseAddress

### Docker

The request address API for docker is relatively simple, the request contains
pool id from which to allocate the address, and optionally the requested IP address.
(In addition to an options dictionary which is a IPAM driver specific)

Release is pretty much straightforward as well in this case, just the address and the
pool id.

### Neutron

This step is a little bit trickier in Neutron, in Neutron, IP allocation is happening
when the port is created, thats why we actually need to create the Neutron port at
this stage.

The docker network creation itself already triggered creation of a Neutron
network and corresponding pools and subnets for this network.
Kuryr only connect this information and allocate the port from the correct subnet.

When the end point creation API is called, meaning the container is brought up
we already have the port ready and we just need to bind it, Kuryr knows to not
re-create the port in case it was already created for IPAM.
(Remember that we could have used Docker default IPAM, so there is a chance
the port wasnt created yet by Kuryr)

# Summary

I think that the Neutron API fits very well into Docker IPAM entities and API
and i hope this post helped you see it.
We now have full support for Docker libnetwork in Kuryr and actually also have
seamless integration with Docker Swarm.

Our attention now is headed to Kubernetes and Magnum (nested containers) integrations
but we are also working hard on building a strong testing/gate framework which will be
able to verify networking solutions stability and quality.

I think this goal is very important for users/providers/developers of networking
solutions to containers and allow users the ability to compare different containers networking
solutions and find the best one for their environments.

Kuryr in my eyes has an important role in OpenStack
It tries to bridge the gap between OpenStack abstractions and deployment
and containers world and orchestration engines, we have a lot of work ahead of us and we
would love anyone that wants to join/use/share use cases and hopefully as a
community we can connect the dots and make OpenStack the leading abstraction for containers
networking.

