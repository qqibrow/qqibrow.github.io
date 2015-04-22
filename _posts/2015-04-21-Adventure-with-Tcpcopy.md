---
layout: post
title: Adventure with Tcpcopy
---

[Tcpcopy](https://github.com/session-replay-tools/tcpcopy) is a request replication tool which could be very useful in A/B testing and performance testing. It is able to replicate live traffic in production with little overhead and direct the traffic to testing modules. In this way, we could easily replicate a live production environment and catch critical problems before launching. 

I am using the [new architecture](https://github.com/session-replay-tools/tcpcopy) (version v1.0.0) introduced in the main page. I borrow the architecture graph here and made some notation by myself

![my-dev-screenshot]({{ site.baseurl }}/images/tcpcopy-anotation.gif)
