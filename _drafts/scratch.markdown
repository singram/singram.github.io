---
layout: post
title: Scratch
---

Source graph -> chrome

Method #2

Copy your configs to /etc/kubernetes

kubectl describe po kube-dns-v19-kq1gq --namespace=kube-system
kubectl get --namespace=kube-system po

kubectl create -f cluster/saltbase/salt/kube-addons/kube-addon-manager.yaml

kubectl get --namespace=kube-system po
kubectl get deployment --all-namespaces=true


#Prometheus

https://coreos.com/blog/monitoring-kubernetes-with-prometheus.html

kubectl get pods -l app=prometheus -o name | \
sed 's/^.*\///' | \
xargs -I{} kubectl port-forward {} 9090:9090


Adding Highlight.js to Ghost
```language-html
<link rel="stylesheet" href="//cdnjs.cloudflare.com/ajax/libs/highlight.js/8.9.1/styles/github.min.css">  
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/8.9.1/highlight.min.js"></script>  
<script>hljs.initHighlightingOnLoad();</script>  
<style>  
  pre {
    word-wrap: normal;
    -moz-hyphens: none;
    -ms-hyphens: none;
    -webkit-hyphens: none;
    hyphens: none;
    font-size: 0.7em;
    line-height: 1.3em;
  }
    pre code, pre tt {
    white-space: pre;
  }
</style>  
```