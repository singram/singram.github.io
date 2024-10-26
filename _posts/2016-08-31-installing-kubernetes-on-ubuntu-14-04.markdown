---
layout: post
title: Installing Kubernetes on Ubuntu 14.04
date: '2016-08-31 13:24:29'
tags:
- docker
- kubernetes
- ubuntu
---

I typically run my linux environment via VirtualBox on a Windows host for mainly corporate reasons.  [MiniKube](https://github.com/kubernetes/minikube) is the new recommended way to get up and running with Kubernetes for local development, however this requires a host system capable of running a vm and at this time VirtualBox does not support 64bit nested VM's.  With that in mind here are the steps I took to install kubernetes locally, mostly taken from [this](https://github.com/kubernetes/kubernetes/blob/release-1.3/docs/devel/running-locally.md) guide.

**Install Docker**
```language-bash
apt-get install apparmor lxc cgroup-lite
wget -qO- https://get.docker.com/ | sh
sudo usermod -aG docker YourUserNameHere
sudo service docker restart
```

**Install OpenSSL**
```language-bash
sudo apt-get install openssl
```

**Install etcd**
```language-bash
curl -L https://github.com/coreos/etcd/releases/download/v3.0.6/etcd-v3.0.6-linux-amd64.tar.gz -o etcd-v3.0.6-linux-amd64.tar.gz
tar xzvf etcd-v3.0.6-linux-amd64.tar.gz && cd etcd-v3.0.6-linux-amd64
sudo mv etcd /usr/local/bin
etcd --version
```
Original install instructions [here](https://github.com/coreos/etcd/releases)

**Install Go 1.6+**

Remember to remove any previous version installed.

```language-bash
wget https://storage.googleapis.com/golang/go1.7.linux-amd64.tar.gz
tar xzf go1.7.linux-amd64.tar.gz
export GOPATH="/home/singram/personal"
export GOROOT="/home/singram/go/"
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

go get -u github.com/jteeuwen/go-bindata/go-bindata
```
Full instructions can be found [here](https://golang.org/doc/install)

**Install Kubernetes**
```language-bash
mkdir -p $GOPATH/src
cd $GOPATH/src
git clone --depth=1 https://github.com/kubernetes/kubernetes.git
```

**Build and Run kubernetes**
```language-bash
hack/local-up-cluster.sh
```
Beware, you will most likely be prompted for your root password towards the end of the build process.  If you let this timeout, your system will have a number of processes running which are somewhat annoying to cleanup.  If this happens, restarting the system proved the simplest method to reset and retry this step.

If successful you should have a running kubernetes system up and running.

**Configure Kubectl**

From the previous step you should see some output similar to the the commands below.  Open up a new shell and execute the following to set up your `~/.kube/config`
```language-bash
export KUBERNETES_PROVIDER=local
cluster/kubectl.sh config set-cluster local --server=http://127.0.0.1:8080 --insecure-skip-tls-verify=true
cluster/kubectl.sh config set-context local --cluster=local
cluster/kubectl.sh config use-context local
cluster/kubectl.sh
```

From this point on you have a working kubernetes system.  You can either use the `cluster/kubectl.sh` or simply install `kubectl` separately as part of your system.  The config file in your home directory is configured and the important part which is what both kubectl versions will key off.

Check out your kubernetes cluster nodes (there'll only be one)
```language-bash
kubectl get no
kubectl describe no 127.0.0.1
```
What about your pods
```language-bash
kubectl get pods
```

And now you should have a fully working locally hosted kubernetes cluster of one.  Superb!