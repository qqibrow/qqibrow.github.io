---
layout: post
title: Automate Syntastic By Strace(TODO)
---

[Syntastic](https://github.com/scrooloose/syntastic) is a syntax checking plugin for Vim that detect syntax errors using external tools and return them back in Vim. I use it in in C++ development and it works pretty well. But to get started with syntastic in one project, I have to setup a conf file, which contains all the include path required during compiling. Here is a [excellent answer](http://stackoverflow.com/questions/16622992/including-header-files-recursively-for-syntastic) about this setup. Basically, I have to add a `.syntastic_cpp_config ` file in the root path of the project and contains the include paths, such as `-I/usr/include` or `-I$(project)/include/`.

It's a pretty simple task, but things will get worse if you have a huge project with a tree structure Makefiles/CMakeLists distributed in sub directories. Moreover, since all we need are in these Makefiles, why bother to read these files again? 


