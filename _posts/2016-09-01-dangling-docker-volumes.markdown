---
layout: post
title: Dangling docker volumes
date: '2016-09-01 16:45:07'
tags:
- docker
---

As anyone who works with docker knows, images and containers accumulate rapidly.

All containers can be cleared down with
```language-bash
docker rm $(docker ps -a -q)
```
And likewise, all Images with
```language-bash
docker rmi -f $(docker images -q)
```
What I wasn’t aware of was the dangling volume issue.  While I had no images or containers left after the above, I did however have 20Gb taken up in dangling volumes which I was oblivious to until I wondered where all my system space had disappeared to.

You can check for dangling volumes independent of containers with
```language-bash
docker volume ls -f dangling=true
```
And remove them with
```language-bash
docker volume rm $(docker volume ls -qf dangling=true)
```
Or you can remove them with the associated container by adding the `–v` flag (e.g. `docker rm –v container name`) if you remember to put the flag on every time.
I would suggest incorporating these to your team purge scripts/procedures for a better cleanup.

More information can be found here (recommended reading)

- http://serverfault.com/questions/683910/removing-docker-data-volumes
- http://container42.com/2014/11/03/docker-indepth-volumes/

