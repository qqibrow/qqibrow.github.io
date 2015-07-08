---
layout: post
title: Single Connection redis performance
---

Recently I did an interesting investigation about redis performance. Given the same load, multiple connections to redis could perform 8x faster than single connection(see below). I am not considering pipelining here, and I already tried many optimization approach in my test. (using taskset to make redis running in single core, using unix domain socket)

```
ubuntu 13.10 x86_64 with kernel version 3.11,
Intel® Core™ i5 CPU M 430 @ 2.27GHz × 4
8GB Memory
```
![enter image description here][1]

The question is **Why multiple connection to redis could perform this faster than single connection?**


The key is to figure out where the extra latency comes from for the single connection case. My first hypophsis is they come from epoll. To find out that, I use [systemtap](http://en.wikipedia.org/wiki/SystemTap) and the [script to determine epoll latency](https://github.com/openresty/stapxx#epoll-loop-blocking-distr). The result (above: 1 connection result, below: 10 connection result. Sorry, the unit should be **nanoseconds** in the pic):
![enter image description here][2]

From the result, you can see that the avg time staying in epoll is almost same, 8,655 nanoseconds vs 10,254 nanoseconds. However, there is a significant difference of total number. For 10 connection we call epoll wait 444,528 times but  we will call it 2,000,054 times in single connection case, which is 4x and that's what lead to this additional time usage.

Next question would be why we call epoll so less time during multiple connection. After exploring a little bit with redis source code, I found the reason. Everytime epoll returns, it will return the number of events it gonna handle. The presudo code is like (hiding most of the details):

```
   fds = epoll_wait(fds_we_are_monitoring);
    for fd in fds:
        handle_event_happening_in(fd);

```
The return value `fds` is a collection of all the events in which IO is happening, for example, read input from socket. For single connection benchmark,  `fds_we_are_monitoring ` is 1, since there are only one connection, then every time the number of fds would be 1. But for 10 connection case, `fds` could be any number less than 10, and then handling the events together in the for loop. The more events we get from epoll return once at a time, the faster we can get. Because the total number of requests is fixed, in this case, 1M set requests.

To verify that, I use systemtap to draw the distribution of return values of the function. [aeProcessEvents](https://github.com/antirez/redis/blob/unstable/src/ae.c#L352), which returns the number of events it handled.

![enter image description here][3]

We can see the avg: 1 in single connection case vs 7 in 10 connection case. Which proves our hypothesis that the more number of events epoll returns once, the faster we can handle the requests, until it become CPU bound.

I think I got the answer for the first question: Why multiple connection to redis could perform this faster than single connection. However, I am still wondering if there is any other way(other than pipeline) to improve performance under single connection. I would appreciate if anyone else could share some thinking about this.

[1]: http://i.stack.imgur.com/JGyMb.png
[2]: http://i.stack.imgur.com/Y6pmj.png
[3]: http://i.stack.imgur.com/lBNGu.png
