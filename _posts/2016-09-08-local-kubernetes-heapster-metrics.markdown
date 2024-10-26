---
layout: post
title: Kubernetes local-up-cluster - Heapster Metrics
date: '2016-09-08 19:53:51'
tags:
- kubernetes
- install
- docker
---

As it turns out I still didn't quite have the [local kubernetes](http://stuartingram.com/2016/09/02/kubernetes-local-up-cluster-addons-in-ubuntu/) setup right.  The documentation around running some of the standard services with local kubernetes is lacking.  There again, it is primarily geared towards kubernetes development and light weight local testing so getting Heapster up and running is a little outside of the wheelhouse so to speak for the targeted audience.

Assuming you have followed my [previous steps](http://stuartingram.com/2016/09/02/kubernetes-local-up-cluster-addons-in-ubuntu/) to get a local kubernetes cluster up and functional, you can get Heapster and Grafana running out of the box with the following

```language-bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster-controller.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb-grafana-controller.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb-service.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana-service.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster-service.yaml
```

Running `kubectl cluster-info` should yield a url to the Grafana front end which you can open up in a browser and view various stats at the node and pod level.  Pretty nice!

**But it's empty right.  There is no data!**

Finding the Heapster pod (`kubectl get po --all-namespaces=true`) and displaying the logs (`kubectl logs heapster-0sbna --namespace=kube-system`) should yield something like
```language-bash
E0907 18:47:05.041415       1 kubelet.go:230] error while getting containers from Kubelet: failed to get all container stats from Kubelet URL "http://127.0.0.1:10255/stats/container/": Post http://127.0.0.1:10255/stats/container/: dial tcp 127.0.0.1:10255: getsockopt: connection refused
```

If you run `curl http://127.0.0.1:10255/stats/container/` from your local host you should see stats returned just fine.

**So what's going on?**

Well Heapster has got the list of nodes from Kubernetes and is now trying to pull stats from the kublete process on each node (which has a built in cAdvisor collecting stats on the node).  In this case there's only one node and it's known by 127.0.0.1 to kubernetes.  And there's the problem.  The Heapster container is trying to reach the node at 127.0.0.1 which is itself and of course finding no kublete process to interrogate within the Heapster container.

127.0.0.1 is normally the IP address assigned to the "loopback" or local-only interface. This is a "fake" network adapter that can only communicate within the same host. It's often used when you want a network-capable application to only serve clients on the same host.

**So how do we solve this?**

As it turns out two things need to happen.
1. We need to reference the kublete worker node (our host machine running kubernetes) by something else other than the loopback network address of 127.0.0.1
2. The kublete process needs to accept traffic from the new network interface/address 

To change the hostname by which the kublete is referenced is pretty simple.  You can take more elaborate approaches but setting this to your `eth0` ip worked fine for me (`ifconfig eth0`).  The downside is that you need a eth0 interface and this is subject to DHCP so your mileage may vary as to how convenient this is.
`export HOSTNAME_OVERRIDE=10.0.2.15`

To get the kublete process to accept traffic from any network interface is just as simple.
`export KUBELET_HOST=0.0.0.0`

So all together the following will start a local kubernetes instance with DNS and the ability for containers to reach and interact with the kublete process

```language-bash
export KUBERNETES_PROVIDER=local  
export API_HOST=`ifconfig docker0 | grep "inet addr" | awk -F'[: ]+' '{ print $4 }'`  
export KUBE_ENABLE_CLUSTER_DNS=true  
export KUBELET_HOST=0.0.0.0
export HOSTNAME_OVERRIDE=`ifconfig eth0 | grep "inet addr" | awk -F'[: ]+' '{ print $4 }'`  
hack/local-up-cluster.sh  
```

You will of course need to reload all the replica and service definitions for Heapster.  But after doing this and waiting a minute or two you should see data accumulate in the graphs.  Data points are recorded every 60 seconds so give the system time to prove it's working.  You can also check the Heapster pod logs for errors while you wait to verify everything is working.

As an added bonus if you are running the [Kubernetes dashboard](https://github.com/kubernetes/dashboard) (see [here](http://stuartingram.com/2016/09/02/kubernetes-local-up-cluster-addons-in-ubuntu/) for instructions) you will also get statistics from Heapster fed through to that automatically.  

**Awesome sauce!**
