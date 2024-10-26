---
layout: post
title: Bootstrapping RVM on Ubuntu
date: '2012-07-13 11:21:00'
tags:
- ubuntu
- ruby
- rvm
---

Generally speaking the packaged /rubygems for most linux distros lead to more problems than they solve.  In order to gain a stable and consistent ruby platform within development teams across multiple machines & platforms I find that (Ruby Version Manager) is an excellent solution.  In a nutshell it allows a user to cleanly and simply manage both ruby interpreters and isolate gem collections () between projects on a single box. 

Please note these instructions work for my based dev environment and needs.  Full rvm installation instructions can be found .  A shorter bootstrap suggestion can be located at the bottom of that page. 

While I highly recommend RVM for development environments I would not recommend it's use in production due to a rapid release cycle culture, some of which, historically have had issues.  If you don't require multiple ruby environments on your production system why add the overhead and external risk? 
```language-bash
sudo apt-get install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion curl g++ openjdk-6-jre-headless ant openjdk-6-jdk 

curl -L https://get.rvm.io | bash -s stable --ruby 

source /home/singram/.rvm/scripts/rvm 
rvm install ruby-1.9.3-p194 --with-openssl-dir=/usr/local --default 
rvm install jruby 
rvm list known 
```

Useful commands to debug installation issues 
```language-bash
rvm notes 
```
[here](https://rvm.io//rvm/install/)