---
layout: post
title: Fix MS Visio 2010 Crashes
date: '2012-10-07 23:11:00'
tags:
- visio_2010
- windows_64_bit
---

In the hope that this will help someone else. I've been suffering constant crashes when running MS Visio 2010 rendering it mostly unusable.  Basically what worked for me was to disable the 'Send to Bluetooth' add-in that appears to be enabled by default. 

**Problem environment** 

* Windows 7 64bit Professional
* MS Office Visio 2010

**Solution** 

To fix, you must REMOVE the add-it, not disable it. 

1. Run Visio as Admin (Shift + Right Click the icon and choose Run As Administrator) 
or 
Click 'Start' in the search box type 'visio.exe'.  Right click on 'visio.exe' and select 'Run as Administrator'
2. In Visio.  Under 'File' tab, click 'Options'
3. Select 'Add-Ins'.
4. At the bottom, Click the 'Go' button to manage 'COM Add-ins'.
5. Select the offending add-in and click Remove.
6. Restart and test.