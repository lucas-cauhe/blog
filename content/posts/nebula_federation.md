+++
date = "2026-01-02T12:23:48+01:00"
draft = false 
title = "Cloud Federation"
subtitle = "Infrastructure deployment and policy management"
summary = "This post describes how multiple cloud instances can work together, obey to the defined policies and the requirements to deploy such technology"
description = "This post describes how multiple cloud instances can work together, obey to the defined policies and the requirements to deploy such technology"
author = "Lucas"

tags = ["Cloud Federation", "OpenNebula", "Ceph", "Vagrant", "OPA"]
categories = ["Technical"]

featuredImage = "/images/federation/federation-fi.png"
featuredImagePreview = "/images/federation/federation-fi-preview.png"

author-link = "https://github.com/lucas-cauhe/cloud-fed"
+++

## Summary

Cloud Federation was the subject of the project I worked on for my Bachelor's thesis.
The project had three main goals: achieve data sovereignty in an institutional context, by using an On-Premise cloud provider, share resources accross entities and develop a policy enforcing system, integrated in the cloud provider.
But first you need to bring all the infrastructure up.
Storage and networks definition, cloud provider integrations, automation tools, etc...
This blog post highlights the main aspects of this project, although the complete documentation can be found at the end of it.

It is also important to note that the system described in this post referes to the virtual version of the real one.
Due to monetary costs and location unavailability the system was simulated/virtualized in a enterprise-level server.

## Architecture design

The 3-plane representation for cloud federation from [NIST](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.500-332.pdf) inspired the design for this federation.
The cloud provider was already fixed for the project, which was OpenNebula.

![Federation Architecture](/images/federation/arch.svg "Federation Architecture")

In the trust plane, the OpenNebula instances share the `oneadmin` user to be part of the same federation.
Trust is further developed in the infrastructure with network policies.

The management plane defines users, detailed in the next section, policies and the monitoring service.
It also describes their dependencies and their support for the upper plane.
Another role for this layer is security, making communications secure accross different entities.

Finally, the usage plane relates users to the resource catalog, where they will be able to publish and deploy virtual services offered in it.
Here is where most of the policies are enforced and users obey to permissions according to their original entity.

## Policy management

Let me tell you first about OpenNebula's federation design and user management.
Their concept of federation is, essentially, replication of internal state.
But not all the state, just users, ACLs and marketplace information, as far as it concerns this project.
Since it seems more like a SSO implementation, I extended the default user organization so that there is no global admin, but only local admins in each zone.
Users can be added to federation-wide groups get extended permissions and will always obey to custom policies defined in each zone.

![Policy enforcing system design](/images/federation/policies.svg "Policy enforcing system design")

In the picture above it's described the design that I eventually implemented.
The monitoring system is essential to this design as it acts as the source of truth for what is happening on each entity's infrastructure.
Metrics are collected by custom OpenNebula hooks and processed with the resource's deployment information (XML template).
This is used as the input for the OPA engine that provides a veredict on whether the resource should stay deployed.

It was designed to be a passive system, where non-compliant resources need to be fixed by the user rather than the system itself.

The system was tested against a small set of policies (naming and overload).
Resource creation triggers the hook that follows the workflow described above and removes the resource created if the result from OPA's endpoint is not compliant.

## Resource sharing

One of the requirements for this project was to have a catalog, a virtual space where users from different entities could share their virtual resources, such as virtual disk images and full-featured services.
This concept was implemented using OpenNebula's Marketplace service, and developing a more complex infrastructure underneath.

![Marketplace infrastructure design](/images/federation/marketplace.png "Marketplace infrastructure design")

OpenNebula's Marketplace is fully documented in its own site, check it out for more details.
What I find more interesting here is the infrastructure that guarantees high availability and fault tolerance using multiple Ceph clusters.

Here is a description of the thinking path that got me to the solution:

> > "Where should I store the Marketplace's objets?"
>
> In a Ceph cluster, of course. It provides object storage and a S3-compatible interface.

> > "Which Ceph cluster should I select, among all the entities?"
>
> All of them, if any fails, the others should be able to retrieve the Marketplace's objects.

> > "Is there a way to replicate a pool's objects accross multiple Ceph clusters and having a unified vision of it?"
>
> There is, it is called Ceph Object Gateway's Multi-site configuration.

And that was it, define the RGW resource in Puppet that includes itself in the Multi-site configuration and whether it acts as the main or secondary zone. The specific configuration I chose for this design was the Multi-zone one.

## Infrastructure

The components of the infrastructure were networks, storage (local and cloud support) and the cloud provider deployment.

### Design

The main aspect considered for the design was to be self-contained, meaning that every architecture's component's characteristic would be stored, processed or transported by a component from the infrastructure.
The infrastructure is based in Podman containers, OpenNebula and Ceph were deployed as such.

### Deployment

The deployment of the infrastructure was automated following carefully a series of steps in two separate stages.
During the first stage, I used Puppet Bolt to precisely define the relationships between the infrastructure components, distribute plans accross virtual machines and run them in parallel when possible.

During first stage, as a first step I defined the networks, bridged for the virtual machines and macvlan for containers.
Next came the Ceph clusters, 3 monitors, 2 managers, 5 OSDs, 1 RGW and 4 MDS backing 2 CephFS.
PGs configuration and CRUSH map rules remained untouched.
The remaining steps involved establishing the federation between two highly available OpenNebula instances, using the Ceph datastore.

During the second stage, I defined a bunch of OpenTofu manifests using the OpenNebula provider.
The Marketplace, the backup datastore, users and the OPA engine virtual machines are defined in these manifests.

## What can be improved

There are a couple of things that could be done better, most of them involve the infrastructure deployment.

### OpenNebula native Puppet resources

When I started this project I knew OpenNebula offers the One-Deploy tool for automated deployment and it's flexible enough to fit the needs of this project.
However, I wanted to get more familiar with Puppet and have a deeper knowledge on how OpenNebula works under the hood.

As of January 2026, OpenNebula's Puppet resources are full of (masked) `exec` resources.
The idea is to use OpenNebula's ruby API to contact each instance for provisioning.
This would avoid the overhead of having the `fork` and `exec` syscalls all the time.

### Ruby interface for librados

The same as before goes for Ceph, even [Openstack's](https://github.com/openstack/puppet-ceph) Puppet Ceph resource is built using `exec` resources or system commands.
However, this improvement would be of interest for a broader audience since this isn't something that only affects Puppet.

---

Thanks for reading this far, more details can be found in the [technical whitepaper](https://github.com/lucas-cauhe/cloud-fed/blob/main/docs/CAUHE_VI%C3%91AO_LUCAS_844665_TFG.pdf) (in spanish) and the source code can be found online in [github](https://github.com/lucas-cauhe/cloud-fed).
