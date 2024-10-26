---
layout: post
title: Packaging a git tag
date: '2013-08-24 22:40:00'
tags:
- git
---

So the other day I was presented with the following requirements. 

From a git repository retrieve a history tag and it's commit history to deliver to a client.  No other branches should be presented to the client, nor work committed after the tag. 

This actually proved to be a little tricky and I'm certain I'm missing some git wizardry but here's what I did. 

Clone the repository (foo) to work on locally 
```language-bash
git clone myname@github:foo 
```
Checkout the tag and create a branch from it. 
```language-bash
cd foo 
git checkout mytag_1.0.0 
git checkout -b mytag_1.0.0_snapshot 
```
Remove all other local branches and cleanup the repository 
```language-bash
git branch -D master 
git gc 
```

At this point you should have a local repository with a single local branch representing the tag you want and a number of references to remote branches.  This can be verified with 
```language-bash
git branch -a 
```
Now clone your local repository again 
```language-bash
cd .. 
git clone foo foo_final 
```

The foo_final repository should contain nothing but the branch representing the tag at this point. 
Zip is up, throw it on a flash drive and deliver as appropriate. 

Now I make no claims that this is the best way to do this.  In fact I'm certain there should be a better way but this is what I ended up doing.