---
title: "WebLogic Kubernetes Operator 2.0 Release Candidate now available"
date: 2018-12-22T08:00:23-05:00
draft: false
---

We have just made our 2.0 Release Candidate 1 of the [Oracle WebLogic Server
Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator)
available for early adopters. 

The 2.0 release adds some significant new features including: 

* Support for burning your WebLogic domain into the Docker image, in addition
  to supporting the domain being on persistent storage,
* The operator is now installed using Helm charts, replacing the earlier scripts, 
* You can override domain configuration using configuration override templates, 
* Load balancers and Ingress can now be independently configured, 
* WebLogic logs can be directed to a persistent volume or the WebLogic server 
  console output (stdout) can be directed to the pod log, 
* Added lifecycle support for servers and significantly more configurability 
  for generated pods. 
  
Most of these features are the result of direct feedback and requests from 
our users.  We have a list of additional requests, and starting with this 
release we are moving to a much shorter release cycle, where we will release
small incremental features and updates frequently, rather than having large
releases that are months apart. 

The final v2.0 release will be initial release from where the operator team intends 
to provide backward compatability as part of all future releases.

The team has been working really hard to get this release done.  We have also 
been testing it on several different flavors of Kubernetes.  This release
candidate is suitable for early adopters and testers.  The final 2.0 release
will be very early in the new year.

We are also making a [public Slack channel](https://weblogic-slack-inviter.herokuapp.com/)
available, so if you want to get in touch with the team, please come to `#operator` and 
say "Hello!" to us on Slack.

Thank you to all of our users for your continued feedback, and for everyone 
who has been testing and reporting issues and making suggestions. We hope
you like what we have built for this release. 
