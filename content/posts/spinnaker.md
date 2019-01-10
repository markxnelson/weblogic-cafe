---
title: "Oracle Cloud Infrastructure provider for Spinnaker"
date: 2019-01-09T10:00:29-05:00
draft: true
tags: [oci,spinnaker]
---
Over the last year or so, we have had a group of developers working on the
[Oracle Cloud Infrastructure provider for Spinnaker](https://www.spinnaker.io/setup/install/providers/oracle).
All of this code has gone directly into the Spinnaker repositories on GitHub 
and it is all open source.  

We are now at the point where the provider is pretty much fully functional 
and we have published documentation on the spinnaker.io site, and Spinnaker
have updated their site to list Oracle Cloud Infrastructure as one of the 
supported providers, so that is a nice milestone for us! 

I wanted to share a little bit of information about what the provider actually
does.  In a future post I will provide more detailed information about how
to set it up and use it. 

The provider supports the two main style of deployments - Kubernetes-based 
and instance-based. 

For the instance-based approach, the provider supports:

* Creation of Compute Images (in the bakery),
* Creation of Compute Instances (Server Groups),
* Creation/configuration of Load Balancers, including backend sets, 
  listeners and certificates, 
* Deploying, resizing and terminating Server Groups.

For the Kubernetes-based approach, all of the standard operations are supported
for OCI Container Engine, including load balancing and security.  We did not 
really do much in this space - it just worked with the standard Kubernetes 
provider.

The provider also supports other OCI features like: 

* [Object Store](https://www.spinnaker.io/setup/install/storage/oracle/)
  as a storage provider for the persitent store and 
  as an artifact repository,
* Registry as a Docker image registry.

The documentation for the OCI provider is available 
[here](https://www.spinnaker.io/reference/providers/oracle/).

Here are a few screenshots from Spinnaker to give you a feel for how the 
provider manifests in the Spinnaker UI:

{{< figure src="/images/spinnaker001.png" caption="Creating a Server Group" >}}

{{< figure src="/images/spinnaker002.png" caption="Baking and deploying images" >}}

{{< figure src="/images/spinnaker003.png" caption="A deploy pipeline with OCI Server Group deployment configuration" >}}

{{< figure src="/images/spinnaker004.png" caption="Detail view of the Server Group" >}}

There is also a [video demonstration](https://www.youtube.com/watch?v=IoLllqbj6ZE)
on YouTube that shows deploying an application to OCI and OKE using Spinnaker.

