---
layout: post
title: Record Replay Live Stream
---

## Motivation

Recording live steam data is useful from several aspects:

1. Debug exceptions when using external api. Most of the time we redo the same call several times to debug. Instead, u could record once and analysis the underlying data.

2. Build an end-to-end functional test. Every time the same dataset is playing, the underlying result should also be consistent. This test will be very helpful within the continuous delivery system.

3. Performance testing. Replaying data way faster than in production and investigate the bottomneck of the system.

## Background

In my work, we subscribed live data through external connections. Testing is a problem because the live test data is different every time. All we can do is to compare real-time processing results between test and production. In this way, we sometimes get false alerts because of the delay of the comparison. Then we work on tools to record data to local file and replay it in the test environment later and we realize the difficulties: 

1. A variety of protocols are used (TCP, UDP, multicast)
2. Authentication has to be faked and it's not as simple as "enter your username and password".
3. data comes from multiple source, which means we might repeat 1,2 multiple times.

We want to accomplish this with the least change of the current system and that's when [socat](http://www.dest-unreach.org/socat/) stands out.

## Socat
socat is short for "socket concatenate" and its function is very similar with [cat](http://unixhelp.ed.ac.uk/CGI/man-cgi?cat). In cat, `cat file1 file2 file3` will concatenate all files and output to STDOUT. `socat` could accomplish similar things with sockets. Here is a simple echo server using socat. For each request, it will echo the complete HTTP request back.

{% highlight bash %}
socat TCP-LISTEN:8080,fork exec:'cat' &
# result shown on browser http://hostname:8080?msg=hello
GET /?msg=hello HTTP/1.1
Host: hostname:8080
Connection: keep-alive
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/43.0.2357.130 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: en-US,en;q=0.8,ja;q=0.6,zh-CN;q=0.4,zh-TW;q=0.2
Cookie: blablabla
{% endhighlight %}

This command will open a listening port on localhost:8080 and wait for connections. Once it got a connection it will execute the `cat` command and pipe what it received through the tcp connection to it. So actually the whole thing looks like `echo blablabla | cat` more or less, but in a more fancy way.

## Demo
More interesting thing comes in following demo. There is a [redis](http://redis.io/) running on localhost:6379. First some background, set and get command:
{% highlight bash %}
./redis-cli -h localhost -p 7777 set Foo "hello"
OK
./redis-cli -h localhost -p 7777 get Foo
"hello"
{% endhighlight %}

Next we do:
{% highlight bash %}
# setup a socat server to record everything
socat TCP-LISTEN:7022 SYSTEM:'tee message_sent | socat - "TCP4:localhost:7777" | tee message_received' &
# redo the request by with port 7022 rather than 7777
./redis-cli -h localhost -p 7022 set Foo "hello"
{% endhighlight %}

The exactly same result "hello" returns and we got two files `message_received` and `message_sent`. As the name suggested, the `message_sent` file is what was sent to redis server and `message_received` is what was received.
It's a long command but the whole idea is the same as before. we use two socat command here so that we could explicitly get the stream in either direction. Then we use `tee` command to pipe both streams to local files.

Open message_sent file:
{% highlight bash %}
cat -vet message_sent
*2^M$
$3^M$
get^M$
$3^M$
Foo^M$
{% endhighlight %}

The message is in [redis protocol format](http://redis.io/topics/protocol), here it means an array with two elements, the first one is a string "get" with length 3, the second one is a string "Foo" with length also 3. Using socat we can do such raw data analysis to find out the underlying truth.

Replaying a stream of commands is also possible. Repeat the same socat setup as before and try `./redis-cli -h localhost -p 7022` this time:
{% highlight bash %}
./redis-cli -h hostname -p 7022
hostname> set test2 100
OK
hostname> set test3 300
OK
hostname> set test4 400
OK
hostname> exit
{% endhighlight %}

As before, the `message_sent` and `message_received` will be generated. check `message_sent`:
{% highlight bash %}
cat -v message_sent
*3^M
$3^M
set^M
$5^M
test2^M
$3^M
100^M
*3^M
$3^M
set^M
$5^M
test3^M
$3^M
300^M
*3^M
$3^M
set^M
$5^M
test4^M
$3^M
400^M
{% endhighlight %}

we open another redis client and run monior `redis_cli -h hostname -p 7777 monitor` and then reply this `message_sent` file, the following message will show up:
{% highlight bash %}
1435785505.756266 [0 127.0.0.1:48091] "set" "test2" "100"
1435785505.756290 [0 127.0.0.1:48091] "set" "test3" "300"
1435785505.756298 [0 127.0.0.1:48091] "set" "test4" "400"
{% endhighlight %}

Therefore, redis did get commands from the replaying and did exactly the same thing as before.

