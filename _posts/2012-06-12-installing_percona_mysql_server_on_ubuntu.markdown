---
layout: post
title: Installing Percona (Mysql) Server on Ubuntu
date: '2012-06-12 22:54:00'
tags:
- mysql
- ubuntu
- percona
---

Full documentation for installing Percona server on Ubuntu can be found on Percona's official site . 

However this is the short form I run through in bash.... 
```language-bash
sudo su - 
export UBUNTU_CODENAME=`lsb_release -c | sed "s|Codename:\s||"` 
gpg --keyserver hkp://keys.gnupg.net --recv-keys 1C4CBDCDCD2EFD2A 
gpg -a --export CD2EFD2A | sudo apt-key add - 
echo "deb http://repo.percona.com/apt $UBUNTU_CODENAME main" >> /etc/apt/sources.list 
echo "deb-src http://repo.percona.com/apt $UBUNTU_CODENAME main" >> /etc/apt/sources.list 
apt-get update 
apt-get install percona-server-server-5.5 percona-server-client-5.5 
```
[here](http://www.percona.com/docs/wiki/repositories:apt)