---
layout: post
title: Installing Emacs 24.1 on Ubuntu
date: '2012-06-11 23:38:00'
tags:
- ubuntu
- emacs
---

Emacs 24.1 is finally out. A complete list of changes can be found . 

As of the time of writing there is no official Ubuntu repository for the final 24.1 release that I know of.  There are however a number of community members supporting nightly build packages but I prefer the following method to get up and running until a official package is supported. 

Only the first part is Ubuntu specific.  How you load in dependencies on your own system may vary. 
```language-bash
sudo apt-get install xorg-dev libjpeg-dev libpng-dev libgif-dev libtiff-dev libncurses-dev 

wget ftp://ftp.gnu.org/gnu/emacs/emacs-24.1.tar.gz 
tar xvfz emacs-24.1.tar.gz 
cd emacs-24.1 
./configure 
make 
sudo make install 
```
[here](https://www.gnu.org/software/emacs/NEWS.24.1)