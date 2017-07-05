---
layout: post
title:  "Networking Hell with Linux & Lenovo"
categories: blog
tags: linux
published: false
---

I spent most of today in Linux Networking Hell.
I recently bought a refurbished computer so I can have a proper Linux environment to play in.
All was going well, until my wireless network disconnected.

My new computer suffers from a disconnect loop.
After a few hours of use (seemingly at random), it will disconnect from the network.
If the network is set to auto-connect, it will keep trying to establish a connection but always gets disconnected after about a minute.
Eventually, it loses the ability to see networks.

When I realized what was going on, I recalled an old friend's hatred of Network Manager.
He insisted on setting up all wireless networks manually on the command line.
That still seems excessive, but I was frustrated by the lack of error messages.
Why do I keep getting booted from the network?
Why can't I see the networks that my other computer and phone can?

Even though Network Manager tries to be friendly and non-technical, Linux doesn't hide the actual tools to root out the problems.
Debugging this was about 65% frustrating and 35% fun, and I picked up a few Linux tricks along the way.

# Power Management

The first hint I got was people on Ubuntu forums who were also being constantly disconnected.
The proposed solution is to disable power management.
What is happening is the network gets turned off when its on battery and not in use.
This fits with the disconnects I've gotten, which have all been on battery while idling.
