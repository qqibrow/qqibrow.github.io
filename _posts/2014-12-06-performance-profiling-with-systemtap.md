#Systemtap Tutorial

SystemTap (stap) is a scripting language and tool for profiling program on Linux kernel-based operation system. It could be used to probe both kernel and userland functions. You can find what it can do in the latest Linux performance tools graph:

![alt text][logo]
[logo]: http://www.brendangregg.com/Perf/linux_observability_tools.png

Stap supports c++ and c. For java, it may support it  from RHEL7 according to [this](http://developerblog.redhat.com/2014/01/10/probing-java-w-systemtap/). The key is the systemtap-runtime-java package, which I don’t see it in RHEL6 repo.

##Install Guide

* Make sure utrace is enabled in your OS. By default, RHEL should has this option open. But it’s better to double check.
   ```
   grep CONFIG_UTRACE /boot/config-`uname -r` 
   ```
* Install required kernel package, including kernel-debuginfo, kernel-debuginfo-common, kernel-devel (**Optional if you only use user-land probe**)
```
   sudo yum update -y --enablerepo="y-extras,y-contrib,y-updates"
   sudo yum update -y --enablerepo="y-extras,y-contrib,y-updates" kernel-debuginfo
   sudo yum install -y --enablerepo="y-extras,y-contrib,y-updates" kernel-debuginfo
   sudo yum install -y --enablerepo="y-extras,y-contrib,y-updates" kernel-devel
```
* Install debug info for program ls, which we will use as a test.
`` debuginfo-install `rpm -qf /bin/ls` ``
* Test whether the installation works and user-land probe is enabled. If succeed, it should print out a line "hello world!"
```
sudo stap -e 'probe process("ls").function("main") { printf("hello world!\n"); }' -c ls
```
##Usage Guide

You don’t need to learn stap language very well to use it. There are many good resources on Github. I mainly use scripts from [agentzh](https://github.com/agentzh), who is pretty active on performance tuning using stap. Blow are his two repositories I use:

###[nginx-systemtap-toolkit](https://github.com/openresty/nginx-systemtap-toolkit)

Although this repo is mainly for nginx performance tuning, there are also many for general purpose usage. Here I use the [sample-bt](https://github.com/openresty/nginx-systemtap-toolkit/blob/master/sample-bt) and [sample-bt-off-cpu](https://github.com/openresty/nginx-systemtap-toolkit/blob/master/sample-bt-off-cpu) to generate the [FlameGraph](http://www.brendangregg.com/flamegraphs.html). 

* **On-CPU Flame Graph**

sample-bt could be used to generate in-CPU graph. This graph shows how your program consume CPU cycles from the perspective of functions. Follow the instruction [here](https://github.com/openresty/nginx-systemtap-toolkit#sample-bt). 

For example, below graph is got from profiling csfeed daemon. From the graph, you can clearly see that it cost most of CPY on the function. `parseCTF`. Click that function, you can follow the stacktrace further to find bottomnecks.

**ATTENTION**: For C++ program, we need to unmangle the .bt file. Please run `cat [bt file] | c++filt -n > [output bt file]`

![alt text][csfeed_oncpu]
[csfeed_oncpu]: http://effectaffect.corp.ne1.yahoo.com/share/csfeed_d1_oncpu.svg
[Click the link to show interactive graph](http://effectaffect.corp.ne1.yahoo.com/share/csfeed_d1_oncpu.svg)


* **Off-CPU Flame Graph**

sample-bt-off-cpu could be used to generate off-CPU graph, which shows all the blocking(or latency) come from. Then running process is exactly the same with sample-bt.


FlameGraph is pretty powerful in performance tuning, since it could give you a overview of the program without any domain-specific knowledge. Need to mention that FlameGraph is independent of stap, you can definitely use other tools to generate flameGraph. Please check following links.

1. [Nodejs in Flames](http://techblog.netflix.com/2014/11/nodejs-in-flames.html)
2. [Java Flame Graphs](http://www.brendangregg.com/blog/2014-06-12/java-flame-graphs.html)


###[stapxx](https://github.com/openresty/stapxx)

stapxx is a macro language built on top of stap, which provide more functionalities and easier to use. For more detail, please review the main page. Here I mainly use the func-latency-distr script. The usage is very clear here.https://github.com/openresty/stapxx#func-latency-distr

For example, below is a comparison I did using func-latency-distr.sxx to show how big the cross-colo latency is:
```
# function call latency result of csfeed1.bf1 connecting to db1.bf1

Start tracing 17531 (/home/y/libexec64/quotefeed/quotefeed)
Please wait for 30 seconds...
Distribution of _ZN7finance9symbology9quotefeed13SymbolMessage7GetYFIDEPSs latencies (in nanoseconds) for 2657 samples
max/avg/min: 20716903/11012404/5844073
   value |-------------------------------------------------- count
 1048576 |                                                      0
 2097152 |                                                      0
 4194304 |                                                     32
 8388608 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  2621
16777216 |                                                      4
33554432 |                                                      0
67108864 |                                                      0

# latency of the same function call of csfeed1.ne1 connecting to db1.bf1

Start tracing 14732 (/home/y/libexec64/quotefeed/quotefeed)
Please wait for 30 seconds...
Distribution of _ZN7finance9symbology9quotefeed13SymbolMessage7GetYFIDEPSs latencies (in nanoseconds) for 27 samples
max/avg/min: 1486334834/1079656591/515740032
     value |-------------------------------------------------- count
  67108864 |                                                    0
 134217728 |                                                    0
 268435456 |@                                                   1
 536870912 |@@@@                                                4
1073741824 |@@@@@@@@@@@@@@@@@@@@@@                             22
2147483648 |                                                    0
4294967296 |    
```
You can see clearly that cross-colo latency is almost 1000 times bigger than inner-colo latency.


For more questions, please contact lniu@yahoo-inc.com. And welcome to add more details. 
