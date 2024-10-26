---
layout: post
title: Moving from Lucid Lynx
date: '2012-07-01 13:43:00'
tags:
- ubuntu
- xfce
- xubuntu
---

I've been a dedicated user for the last 4 years using it as my full-time dev environment.  For the last 2 years I have been using 10.0.4 () which has long term support with no complaints.  Stability and support being more important to me that staying at the new shiny end of things. After all, this is the OS I make a living from, all bleeding edge issues, upgrade snafu's and significant UI changes cost someone money at the end of the day.  With less than a year of support left I thought I'd check out where Ubuntu stands and what the options are.  One last thing, I run my development environment in a .  More on the benefits of this in another post. 

Needless to say, like many others out there, from a power user perspective I am not impressed with the mess that is & .  Slow, bloated, difficult to navigate and clearly not designed to make my life easier without me investing a lot of time to make it so.... 

So the alternatives.... 
- Change to a different distro.  Not something I'm ready to entertain right now.  I'm familiar with the Ubuntu architecture, Debian package support and culture.  It's also been solid as a rock for my development needs. 
- .  Heard a lot of good things about recently and the interface looks very slick.  That being said it is fairly heavy weight for my needs and I am certainly not dependent on the eye candy provided by the desktop environment.  Resources are an issue when I want to maximize the allocations made to my development VM, 
- .  Ubuntu based system (just like Kbuntu) with official Canonical support.  Xubuntu runs the desktop environment, a normally lightweight environment.  In the case of Xubuntu this is a little heavier than I'd like due to some default app choices and gnome integration which is still a key architectural piece of the Ubuntu platform.  However it's still significantly more lightweight and responsive than KDE for my particular use cases. 

Xubuntu it is then.12.0.4 to be precise which is the latest currently available with long term support (3 years). 

Being a new environment for me there are a couple of niggles right off the bat. 

- Terminals are not mapped to the normal Gnome standard <CTRL><ATL><t> which I've become accustomed to but <SUPER><t>.  This can easily be fixed via the Keyboard section in the Setting Manager. 
- No .  A fantastic piece of software I use everyday for quickly launching apps.  is an alternative with very similar functionality for Xfce.  I also use on windows based system for identical behavior.  I like this feature bound to <ALT><SPC> in all environments but the Xfce window manager has this already bound to the window menu.  The window menu is not a feature I use on a daily basis and can be unbound via the Settings Manager -> Window Manager -> Keyboard tab. 
- Access to the main panel menu can be achieved by binding xfce4-popup-applicationsmenu which I bound to <SUPER> for a consistent experience in and out of my VM. 

[Launchy](http://www.launchy.net/)