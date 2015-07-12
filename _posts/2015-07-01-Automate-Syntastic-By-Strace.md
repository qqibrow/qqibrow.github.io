---
layout: post
title: Automate Syntastic By Strace
---

## Introduction
[Syntastic](https://github.com/scrooloose/syntastic) is a syntax checking plugin for Vim that detects syntax errors using external tools and return them back in Vim. I use it in in C++ development and it works pretty well. But to get started with syntastic in every project, I have to do some setup at first ([How to setup syntastic in C++ development](http://stackoverflow.com/questions/16622992/including-header-files-recursively-for-syntastic)). Basically, I have to add a `.syntastic_cpp_config ` file in the root path of the project that contains the include paths, such as `-I/usr/include` or `-I$(project)/include/`.

It's a pretty simple task, but things will get worse when it comes to a huge project with a tree structure Makefiles/CMakeLists distributed in sub directories. Moreover, since all we need are in these Makefiles, why bother to read these files again? 

## Get Started
Well, I start thinking of a way to automate this. A straightforward idea is to track all subprocesses and analysis the passed-in parameters if it's `gcc`. To get started, I tried [pstree](http://linux.die.net/man/1/pstree). It's easy to get executing parameters using `pstree -a`, like below:
{% highlight bash %}
sshd -D
  ├─sshd
  │   └─sshd
  │       └─bash
  │           └─screen -r blog
  └─sshd
{% endhighlight %}
However, it's hard to to live tracking the process in this way. Then my Linux guru friend suggested me to give a shot with `strace`. I used strace before to do error debugging and code analysis, but to be honestly, I didn't think about it in this problem. The truth is, `strace` provides a very elegant one-liner solution:
{% highlight bash %}
strace -eexecve -s 200 -o output -f make
{% endhighlight %}
The syntax is just `strace [command args]` but a lot additional options are provided:

Option | Explanation
--- | --- 
-e | System call to trace. Only things we care about are the parameters in `execve` systemcall.
-s | Length of line output. `gcc` command is extremly long and by default it will be trancated.
-f | Trace child processes as they are created by currently traced processes.
-o | Dump result to a file.

To know more about strace, just read the man page and ["strace Wow Much Syscall " by Brendan Gregg](http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html)

With the powerful `strace`, we could have a two-line solution that meet our basic requirements:
{% highlight bash %}
strace -eexecve -s 200 -o output -f make
perl -ne 'while(m/(-I.*?)",/g) {print "$1\n";}' output | sort | uniq > .syntastic_cpp_config
{% endhighlight %}
About the regex, here is a explanation [example](https://regex101.com/r/wG9pQ7/1), the key is to use non-greedy match and lookahead assertion.

## Improve
One problem is there are sometimes relative path in Makefile in subdirectories. To overcome this, we trace `getcwd` systemcall as well and map them to `execve` tracing result based on pid. The final solution is in python and provided [here](https://github.com/qqibrow/autoSyntastic). The usage is pretty simple, instead of running `make`, run `autosyntastic make` and a `.syntastic_cpp_config` will show up in the current directory.

One importatn thing to know is strace will significantly slow down the performance. Therefore, I suggest run autosyntastic only during setup.

## Final words
Tracing is very interesting and a lot of fun. I introduced [perf](http://qqibrow.github.io/CPU-Cache-Effects-and-Linux-Perf/) and [systemtap](http://qqibrow.github.io/performance-profiling-with-systemtap/) before, which are also powerful tracking tools. I feel that with learning of different tracking tools, I could gradually understand performance better, understand programming better and even understand kernel better. With the comming of Linux 4.0, more and more tracing tools and apis will be available. Well, more to learn and long way to the top!
    
    
[![it's a long way to the top by AC DC](http://ak-hdl.buzzfed.com/static/2014-05/enhanced/webdr02/23/1/grid-cell-8638-1400823901-3.jpg)](https://www.youtube.com/watch?v=-sUXMzkh-jI)
