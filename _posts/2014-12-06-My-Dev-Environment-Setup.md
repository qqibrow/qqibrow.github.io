---
layout: post
title:  My Dev Environmnet Setup
---

A good dev environment is very important for developers. Here I will share the tools I use every day and the corresponding custom settings, including vim, several vim plugins, git and screen. 

All the setups are done in RHEL6, but it should work for other Linux-based OS as well. 

##Vim

Different people have their own flavors about IDE. Vim is my preferred one. There are no good or bad between different choices, as long as you found it OK and happy to use. Here I pick vim for several reasons:

* Most of the time I program C++. To be honestly, no perfect IDE for C++ in the market now. Eclipse? too slow for big project. Clion? hmm… At least I didn’t see it support make. IDE your company brewed?  sorry I don’t know. 

* Reduce distraction. I find switching between keyboard and my touchpad is quite annoying, although they are already close enough on mac. This is my personal experience. When I use vim, I map ``esc`` to ``jj``(double enter j) for the same reason. Basically, I want to focus both my hands on the main keyboards and keep typing fast.

* Vim provide every function I asked, including file explorer, autocomplete, code highlight, git integration, and code auto formatter. All are open source and free.

Following are the steps for vim setup:

1. Making sure you vim is at least 7.3 . Some of the plugins these days require latest vim version. Please Follow the insturction [here](http://blog.angeloff.name/post/2010/10/15/vim-7-on-red-hat-enterprise-linux-rhel/), to setup latest vim if you don't have. Although it said it’s for RHEL, the idea is the same.

1. The vim I use now is from [vim-spf13](http://vim.spf13.com/), which integrates all state-of-art vim plugins and is super easy to use as well. Please check the usage here. I highly recommanded you go over all the usage of the plugins once and try them one by one.

1. Below are some packages i use most frequently:

    * [NERDtree](https://github.com/scrooloose/nerdtree)(included in spf13)
    > NERDtree is the file explorer of vim. With NERDtree, you don't need to memorize all shortcuts for file operations.

    * [clang-format](https://github.com/rhysd/vim-clang-format)(install by yourself)
    > This package integrate clang-format into vim which makes code formatting just one simple hit.

    * [YouCompleteMe](http://valloric.github.io/YouCompleteMe/)(a.k.a ycm)(install by yourself)
    > Ycm is pretty powerful for code autocomplete, but you have to compile it if you want its c-family language    completion(c/c++/obj-c). I have to say the compilation process is not easy, but worth a try. If you have any problems when compiling it, just leave a message. I am very happy to help.

1. Last but not least, here are my personal vim learning experience. 
    1. vimtutor. It hides in everyone’s terminal.
    2. Play the game [vim advanture](http://vim-adventures.com/)
    3. Print out vim cheat sheet and spend 5min to read it every morning. 
    4. Read the book [Practical Vim](http://www.amazon.com/Practical-Vim-Thought-Pragmatic-Programmers/dp/1934356980) if have spare time and money.
    5. Get start with vim Now!
 
##Screen

[screen](http://www.gnu.org/software/screen/) is super powerful tool for terminal, especially if your daily job involve logging into different servers and check. I always use screen on my dev box. Every time I log into my dev box and reopen the screen session, it refresh my memory and therefore I can pick up my work very quickly.

I use customize screenrc([link](https://github.com/qqibrow/setup/blob/master/.screenrc)), got from Github and modified based on my colleague’s suggestion. Major changes include mapping function key to backtick and make F1 to F9  to hotkeys. Please check that file for more detail.

ATTENTION, if you use spf13-vim with screen, Please follow [this](http://stackoverflow.com/questions/6787734/strange-behavior-of-vim-color-inside-screen-with-256-colors) to let your screen support 256 color. Be careful to change the .profile rather than .bashrc refering to [this](http://superuser.com/a/370042) of you are mac user.

##Bash Prompt

Believe it or not, Two of The most frequently command I use are ```pwd``` and ```echo $?```, which doesn’t make any sense. Use bash prompt to change this situation. Please refer to [this](http://www.maketecheasier.com/8-useful-and-interesting-bash-prompts) and choose your best bash prompt. BTW, I use the 2nd one, which shows current path and change font color according to command return color.

##Git [Please Ignore this if you use git GUI]

Everyone use git everyday. Use [git-autocompletion](http://git-scm.com/book/en/v1/Git-Basics-Tips-and-Tricks) to make it more powerful. 

Please leave a message if you have any questions. Hopefully this is helpful for you daily development! Happy coding!


