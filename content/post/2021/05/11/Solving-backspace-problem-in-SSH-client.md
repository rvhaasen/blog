---
title: "Solving Backspace Problem in SSH Client"
date: 2021-05-11T15:17:15Z
tags: [Linux]
draft: false
---
When using SSH on a regular basis on all kind of hosts, at some moment you may run into the 'backspace problem', effectively not being able to erase characters entered from the keyboard. 
This is the result of a mismatch between expected and provided capabilities of the terminal.
Until recent, I could solve this problem by just setting the terminal environment-variable. This however did not work
the last time that I encountered this problem, and I had to find another solution. This blog describes how I solved it. 

The expected capabilities of the terminal on the host can be retrieved with command 'stty -a'. Typical, the expected character(-combination) is ^?

What is delivered can be found by pressing CTRL-V, followed by pressing backspace key. It they differ, a solution can be to set the terminal:
```bash
# export TERM=vt100
```
If this works, the variable can be set in .bashrc (asuming bash as terminal)

**If this does not work, next might solve the problem:**

1. First get the terminalmaps. In my case, these were not available and I had to install them first. If this 
is already present then the next step can be skipped. 

My client was running on an Arch-linux host. Termite was used as terminal. The TERMCAPS however were not set. 
Appearently this is not part of the termite installation. This can be added via: 
```bash
$ curl https://raw.gitusercontent.com/thestinger/termite/master/termite.terminfo | tic -x -
```
This will define the termcaps. 

2. Now, infocmp does work and the terminal caps can be copied to a file:
```bash
$ infocmp > $TERM.ti
```
3. Next, copy the file to the SSH server (e.g. using scp) 
After than, use 'tic' on the server in order to install the termcaps:
```bash
 $ tic <capsfile>
```
Now my backspace did work again!

