---
layout: post
title: Kubernetes local-up-cluster - dns fixes on Ubuntu
date: '2016-09-02 21:47:08'
tags:
- ubuntu
- kubernetes
- dns
---

So as it turns out I didn't get too far beyond the [local kubernetes install](http://stuartingram.com/2016/08/31/installing-kubernetes-on-ubuntu-14-04/) without running into some issues.  The first being the lack of DNS (I wanted to run the amazing [dashboard UI](https://github.com/kubernetes/dashboard)) and then port forwarding to access pod functionality directly.

**Ubuntu prerequisites**

As it turns out there are a number of Ubuntu 14.04 specific hurdles to overcome before kubernetes will work happily.

First of all `dnsmasq` needs to be disabled so comment it out and restart networking services via the following
```language-bash
sudo nano /etc/NetworkManager/NetworkManager.conf
sudo restart network-manager
```
Find out more about `dnsmasq` and ubuntu [here](https://help.ubuntu.com/community/Dnsmasq)
 
Next the tools `socat` and `nsenter` are required for kubernetes port forwarding.
To install `socat` run
```language-bash
sudo apt-get install socat
```
To install `nsenter` is slightly more work due to lack of 14.04 support but not much thanks to the work of [Jérôme Petazzoni](http://jpetazzo.github.io).
```language-bash
docker run --rm jpetazzo/nsenter cat /nsenter > /tmp/nsenter && chmod +x /tmp/nsenter
sudo cp /tmp/nsenter /usr/local/bin
```
Check out the repo [here](https://github.com/jpetazzo/nsenter) or this [gist](https://gist.github.com/mbn18/0d6ff5cb217c36419661) if you want to go step by step

To find out more about `socat` [here](http://www.dest-unreach.org/socat/doc/README) and `nsenter` [here](http://man7.org/linux/man-pages/man1/nsenter.1.html)

**Back to kubernetes**

After these steps it's hopefully smooth sailing.  So lets start kubernetes with DNS on by default by running the following

```language-bash
export KUBERNETES_PROVIDER=local
export API_HOST=`ifconfig docker0 | grep "inet addr" | awk -F'[: ]+' '{ print $4 }'`
export KUBE_ENABLE_CLUSTER_DNS=true
hack/local-up-cluster.sh
```
Instructions for validating your DNS setup can be found [here](https://github.com/kubernetes/kubernetes/blob/master/build/kube-dns/README.md)

Let's add the dashboard
```language-bash
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

This can be accessed via
```language-bash
firefox http://172.17.0.1/ui
```
From this you can view and manage most things you can via the `kubectl` cli.

**Kind of gotcha but not really**

One thing to note is that when you terminate the kubernetes process all the docker containers remain running (see `docker ps`).  This at first caused concern, but then remember kubernetes is designed so that the containers it manages are themselves not dependent on kubernetes to function.  If the scheduler dies, the containers are unaffected, only scheduling.  This is a consistent philosophy throughout the kubernetes system and makes sense that upon shutdown would not remove all running containers.  A few properties to note
1. If kubernetes is subsequently spun up it will reconcile the state of the system with desired state.  As you would expect
2. Other docker containers can be spun up and down locally & independent of those managed by kubernetes.
These properties are possible due to docker labels being applied by kubernetes to the containers it manages.


