---
layout: post
title: Automate Syntastic By Strace(TODO)
---

## Introduction
[Syntastic](https://github.com/scrooloose/syntastic) is a syntax checking plugin for Vim that detect syntax errors using external tools and return them back in Vim. I use it in in C++ development and it works pretty well. But to get started with syntastic in one project, I have to setup a conf file, which contains all the include path required during compiling. Here is a [excellent answer](http://stackoverflow.com/questions/16622992/including-header-files-recursively-for-syntastic) about this setup. Basically, I have to add a `.syntastic_cpp_config ` file in the root path of the project and contains the include paths, such as `-I/usr/include` or `-I$(project)/include/`.

It's a pretty simple task, but things will get worse if you have a huge project with a tree structure Makefiles/CMakeLists distributed in sub directories. Moreover, since all we need are in these Makefiles, why bother to read these files again? 

## Get started
Well, I start thinking of a way to automate this. A straightforward idea is to track all subprocesses and analysis the passed-in parameters if it's `gcc`. Then I tried [pstree](http://linux.die.net/man/1/pstree). It's easy to get executing parameters using `pstree -a`, like below:
{% highlight bash %}
sshd -D
  ├─sshd
  │   └─sshd
  │       └─bash
  │           └─screen -r blog
  └─sshd
{% endhighlight %}
However,  hard to to live tracking the process in this way. Then my Linux guru friend suggested me to give a shot with `strace`. I used strace before to do error debugging and code analysis, but to be honestly, I didn't think about it in this problem. The truth is, `strace` provides a very elegant one-liner solution:
{% highlight bash %}
strace -eexecve -s 200 -o output -f make
{% endhighlight %}
The syntax is just `trace [command args]` but a lot additional options are provided:

Options
--- | --- 
-e | system call to trace. Only things we care about are the parameters in `execve` systemcall.
-s | length of line output. `gcc` command is extremly line and by default it will be trancated.
-f | > Trace child processes as they are created by currently traced processes.
-o | dump result to a file.

With the powerful `strace`, we could have a two-line solution that meet our basic requirements:
{% highlight bash %}

{% endhighlight %}


## Improve

## Conclusion

