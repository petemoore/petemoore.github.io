---
layout: page
title: About
permalink: /about/
---

TaskCluster
===========

[Taskcluster][taskcluster] is an open source CI solution that differs from other CI solutions in the following ways:

* Task can be run on arbitrary platforms, against arbitrary system images
* Tasks can be composed into task graphs to create arbitrary dependency topologies
* Task graphs can be generated and extended dynamically, even as part of a task
* Workers can be deployed anywhere, even on a toaster

It also provides the features you would expect from any world-class CI solution:

* Exhaustive API metadata, serving client libraries and documentation
* Authorization and Security via Hawk, HMAC signatures, authorization scopes, SSL transport
* Centralised logging via papertrail
* Metrics collection via influxdb
* Linux containerisation via docker worker, arbitrary platform support via generic worker


Me
==

I work at Mozilla on the [Taskcluster][taskcluster] team.

[taskcluster]: https://taskcluster.net
