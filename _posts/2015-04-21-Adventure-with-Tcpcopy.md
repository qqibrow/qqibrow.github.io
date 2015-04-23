---
layout: post
title: Adventure with Tcpcopy
---

[Tcpcopy](https://github.com/session-replay-tools/tcpcopy) is a request replication tool which could be very useful in A/B testing and performance testing. It is able to replicate live traffic in production with little overhead and direct the traffic to testing modules. In this way, we could easily replicate a live production environment and catch critical problems before launching. 

I am using the [new architecture](https://github.com/session-replay-tools/tcpcopy) (version v1.0.0) introduced in the main page. I borrow the architecture graph here and made some notation by myself.

![my-dev-screenshot]({{ site.baseurl }}/images/tcpcopy_anotation.gif)

* Online server: the application in production.
* Target server: the test server with test application.
* Assistant server: transfer station for response messages from target server.


1. When a request goes to online server, tcpcopy captured the TCP/IP packets in IP layer.
2. Tcpcopy changed the destination of the packets to target server and then send them out.
3. Packets been sent out.
4. Packets arrive target server and reach application.
5. Application send response package out. Here tcpcopy requires a route table set in target server. If that set correctly, the packets will route to assistant server.
6. Packets arrive assistant server
7. Intercept program will extract response header and send them to tcpcopy on online server through a dedicated tcp connection.
8. as 7
9. as 7

## Test Start

Here we play tcpcopy together with redis. The goal is to replicate redis requests from online server to target server. Before get started, both tcpcopy and intercept program have been deployed to online server and assistant server separately. Both are compiled with `./configure && make && make install`

* online server: 10.90.111.142
* target server: 10.90.109.143
* assistant server: 10.90.110.143
* client server: 10.90.114.82

1. Add router setting in target server. Therefore, packets with destination client server will be routed to assistant server.

{% highlight bash %}
sudo route add -host 10.90.114.82 gw 10.90.110.143
# then run route to check result
# route -n
# Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
# 10.90.114.82    10.90.110.143   255.255.255.255 UGH   0      0        0 eth0
{% endhighlight %}

2. Run `intercept` on assistant server
{% highlight bash %}
./intercept -i eth0 -F tcp and src port 6379 -d
{% endhighlight %}

3. Run `tcpcopy` on online server
{% highlight bash %}
# tells tcpcopy to replicate requests from port 6379 and send them to 10.90.109.143:6379, in which the redis application is running on target server.
./tcpcopy -x 6379-10.90.109.143:6379 -s 10.90.110.143 -d
{% endhighlight %}

4. Start redis applications on both online and target servers and also run `redis-cli monitor` so that we could know requests come to redis.

5. Send a request from client server to test server
{% highlight bash %}
./redis-cli -h 10.90.111.142 set date $(date +"%m-%d-%Y")
{% endhighlight %}

## Test results
{% highlight bash %}
#client server
./redis-cli -h 10.90.111.142 set date $(date +”%m-%d-%Y”)
OK
# online server
./redis-cli monitor
OK
1429739006.548335 [0 10.90.114.82:60931] “set” “date” “04-22-2015”
# target server
./redis-cli monitor
OK
1429739009.883228 [0 10.90.114.82:60931] “set” “date” “04-22-2015”
{% endhighlight %}

From the redis log, u can see there is ~3 miliseconds delay of the request to target server and client ip address of both requests is 10.90.114.82.

# A Deep Analysis using tcpdump
We do the same test one more time and use tcpdump to capture packets with port 6379 (redis default port) in every server involved.
{% highlight bash %}
#client server
./redis-cli -h 10.90.111.142 set date $(date +”%m-%d-%Y”)
OK
# online server
05:36:59.427211 IP 10.90.114.82.33800 > 10.90.111.142.6379: tcp 40
        0x0000:  4500 005c 6447 4000 3406 ebc0 0a5a 7252  E..\dG@.4....ZrR
        0x0010:  0a5a 6f8e 8408 18eb bcad 9fcf ada1 862b  .Zo............+
        0x0020:  8018 0073 3098 0000 0101 080a 49ce fbaf  ...s0.......I...
        0x0030:  04b1 2f1b 2a33 0d0a 2433 0d0a 7365 740d  ../.*3..$3..set.
        0x0040:  0a24 340d 0a64 6174 650d 0a24 3130 0d0a  .$4..date..$10..
        0x0050:  3034 2d32 332d 3230 3135 0d0a            04-23-2015..
05:36:59.427353 IP 10.90.114.82.33800 > 10.90.109.143.6379: tcp 40
        0x0000:  4500 005c 6447 4000 3406 edbf 0a5a 7252  E..\dG@.4....ZrR
        0x0010:  0a5a 6d8f 8408 18eb bcad 9fcf 14c7 ee83  .Zm.............
        0x0020:  8018 0073 f6ea 0000 0101 080a 49ce fbaf  ...s........I...
        0x0030:  04b0 9b4a 2a33 0d0a 2433 0d0a 7365 740d  ...J*3..$3..set.
        0x0040:  0a24 340d 0a64 6174 650d 0a24 3130 0d0a  .$4..date..$10..
        0x0050:  3034 2d32 332d 3230 3135 0d0a            04-23-2015..
05:36:59.427517 IP 10.90.111.142.6379 > 10.90.114.82.33800: tcp 5
        0x0000:  4500 0039 a915 4000 4006 9b15 0a5a 6f8e  E..9..@.@....Zo.
        0x0010:  0a5a 7252 18eb 8408 ada1 862b bcad 9ff7  .ZrR.......+....
        0x0020:  8018 0072 f6bf 0000 0101 080a 04b1 2f3f  ...r........../?
        0x0030:  49ce fbaf 2b4f 4b0d 0a                   I...+OK..
# target server
05:37:02.762773 IP 10.90.114.82.33800 > 10.90.109.143.6379: tcp 40
        0x0000:  4500 005c 6447 4000 3406 edbf 0a5a 7252  E..\dG@.4....ZrR
        0x0010:  0a5a 6d8f 8408 18eb bcad 9fcf 14c7 ee83  .Zm.............
        0x0020:  8018 0073 f6ea 0000 0101 080a 49ce fbaf  ...s........I...
        0x0030:  04b0 9b4a 2a33 0d0a 2433 0d0a 7365 740d  ...J*3..$3..set.
        0x0040:  0a24 340d 0a64 6174 650d 0a24 3130 0d0a  .$4..date..$10..
        0x0050:  3034 2d32 332d 3230 3135 0d0a            04-23-2015..
05:37:02.763000 IP 10.90.109.143.6379 > 10.90.114.82.33800: tcp 5
        0x0000:  4500 0039 75f0 4000 4006 d039 0a5a 6d8f  E..9u.@.@..9.Zm.
        0x0010:  0a5a 7252 18eb 8408 14c7 ee83 bcad 9ff7  .ZrR............
        0x0020:  8018 0072 f4c0 0000 0101 080a 04b0 9b6d  ...r...........m
        0x0030:  49ce fbaf 2b4f 4b0d 0a                   I...+OK..
# assistant server
05:37:02.455777 IP 10.90.109.143.6379 > 10.90.114.82.33800: tcp 5
        0x0000:  4500 0039 75f0 4000 4006 d039 0a5a 6d8f  E..9u.@.@..9.Zm.
        0x0010:  0a5a 7252 18eb 8408 14c7 ee83 bcad 9ff7  .ZrR............
        0x0020:  8018 0072 1ecd 0000 0101 080a 04b0 9b6d  ...r...........m
        0x0030:  49ce fbaf 2b4f 4b0d 0a                   I...+OK..
{% endhighlight %}









