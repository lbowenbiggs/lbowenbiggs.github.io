---
layout: post
title:  "Networking Hell with Linux & Lenovo"
categories: blog
tags: linux
published: true
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
This fits with the disconnects I've gotten, which have all been on battery while the network was idle.

To check the power management setting, run iwconfig.

```bash
$ iwconfig
wlp3s0    IEEE 802.11bgn  ESSID:off/any  
          Mode:Managed  Access Point: Not-Associated   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr=2347 B   Fragment thr:off
          Power Management:on
```

My wireless network interface is called `wlp3s0`, and the last setting it lists is `Power Management:on`.
To turn it off, run the following command (replace `wlp3s0` with the name of your network interface).

```bash
$ iwconfig wlp3s0 power off
```

This command *looks* like it turns off the device, but it actually disables the power management scheme.
If you did want to disable the interface, run `ifconfig wlp3s0 down`.

The Ubuntu forums typically reported success at this stage, but I ran into more problems.
As soon as the network interface changed state (eg, from disconnected to connecting), the power management would turn back on!
Even worse, all the directions for permanently disabling power management did not work for me.

# Kernel Modules & The RTL8192CE Driver

I started to investigate the actual driver, following advice from the [Ubuntu Troubleshooting Guide](https://help.ubuntu.com/community/WifiDocs/WirelessTroubleShootingGuide/Drivers).
First, I had to check what physical device I had installed.
I ran `lshw -C network` (to list the hardware for networking).
This is the listing for my Wireless Interface:

```bash
[...]
  *-network
    description: Wireless interface
    product: RTL8191SEvB Wireless LAN Controller
    vendor: Realtek Semiconductor Co., Ltd.
    physical id: 0
    bus info: pci@0000:03:00.0
    logical name: wlp3s0
[...]
```

Now I knew I have a Realtek (vendor) card.
To check the driver, I ran `lsmod` (List Modules), searching for the `rtl`, the first few letters in my device:

```bash
$ lsmod | grep rtl
rtl8192se              65536  0
rtl_pci                28672  1 rtl8192se
rtlwifi                77824  2 rtl_pci,rtl8192se
mac80211              737280  3 rtl_pci,rtlwifi,rtl8192se
cfg80211              565248  2 mac80211,rtlwifi
```

The troubleshooting guide suggests blacklisting modules if more than one is used, but that isn't necessary here.
The last column lists the modules that use the one listed.
`rtl8192se`, the actual wireless adapter driver, depends on all the others, so I can't blacklist the other drivers.

It seemed wrong that my device, `rtl8191se`, would use the diver for `rtl8192se`.
With a healthy bit of searching, I found a [page in the Debian Wiki](https://wiki.debian.org/rtl819x) which confirmed this is the correct driver.
It also corroborated my suspicions about power management:

> Devices supported by this driver may suffer from intermittent connection loss. If so, it can be made more reliable by disabling power management

The wiki suggests using `options` to disable the `ipc` and `fwlps` settings, but `options` isn't readily available in Mint.
So I had to dig deeper.
I had to figure out how to pass arguments to the `rtl8192se` module that controls my wireless interface.
And that meant learning more about the `rtl8192se` module in general.

[ArchWiki's guide to kernel modules](https://wiki.archlinux.org/index.php/Kernel_modules) help me start to understand.
In this case, module is synonymous with driver, and the goal is to dynamically load hardware drivers based on what is available.
Before, when I ran `lsmod`, I was checking which modules were loaded.
Specifically, I wanted to know if my wireless adapter driver was running.
Now, I wanted to list information on a specific module, so I ran `modinfo`:

```bash
$ modinfo rtl8192ce
[...]
parm:           swenc:Set to 1 for software crypto (default 0)
 (bool)
parm:           ips:Set to 0 to not use link power save (default 1)
 (bool)
parm:           swlps:Set to 1 to use SW control power save (default 0)
 (bool)
parm:           fwlps:Set to 1 to use FW control power save (default 1)
 (bool)
parm:           debug:Set debug level (0-5) (default 0) (int)
```

The interesting part of this command is the `parm` tags.
The module explains what the `ips` and `fwlps` settings are: different levels of power management.
Now that I know what these settings are, I'm not afraid to change them.
First, remove the module with `modprobe -r rtl8192ce`.
Then start the module with `ips` and `fwlps` disabled (set to 0) and check the results:

```bash
$ modprobe rtl8192se ips=0 fwlps=0

$ systool -vm rtl8192se
[...]
Parameters:
  debug               = "0"
  fwlps               = "N"
  ips                 = "N"
  swenc               = "N"
  swlps               = "Y"
[...]
```

I restarted network manager (`service network-manager restart`), and tried to connect to my network.
And I was disconnected.
I even ran a manual scan to check if the Network Manager GUI was hiding things from me, but got no results:

```bash
$ iwlist scan
wlp3s0    No scan results

lo        Interface doesn't support scanning.

enp0s25   Interface doesn't support scanning.
```

Time to consider if it is a Hardware Problem.

# Whitelisted Wireless Cards

I knew from the Debian Wiki that this driver was known to be buggy in Linux.
As far a I can tell, I'm running the most up-to-date version of the driver.
The other option was to try a different card all together.

I happen to have my old laptop laying around, so I harvested its parts.
It has an Atheros AR5B95 wireless adapter.
Both are Mini PCIe chips, so I simply screwed it in and tried it out.
On boot, my computer made a series of annoying beeps before informing me I had an unsupported PCIe device.
This check caused POST to fail, so I was unable to simply swap out wireless adapters.

I searched for reasons the Atheros chip wouldn't work, and was hit with *many* similar problems from other people with Lenovo computers.
It turns out that Lenovo (along with HP and some other manufacturers) only allow certain hardware to run with their machines.

I couldn't use my old laptop's wireless card, but the disconnct loop was still going.
I started shopping around for a new wireless card, but found that other people had issues with seemingly every wireless card.
I couldn't find information about my model on Lenovo's website, but [ThinkWiki has it](https://www.thinkwiki.org/wiki/Category:T410).
Since shipping with an Intel Centrino is an option, I am betting those devices will be supported.

Lenovo uses a Field Replacement Unit (FRU) number to identify compatible parts.
Most forums recommend matching the FRU exactly when replacing a part.
But since I am upgrading an not replacing, I'm not sure if that rule applies.
All wireless cards I could find that match my FRU were exact copies of the ones I have -- not helpful at all.

I decided to buy a Centrino, even though the FRU is different.
I'm not sure if it'll work, but there's still the whole world of BIOS and chip modding that I could explore....

# Power Management, Redux

Having bought a wireless card, I now have to wait over a week for delivery.
In the meantime, I put my computer back together and tried to convince myself I could live with it.
I still have a perfectly reliable wired card, and I learned new things about Linux today.
I decided to write it all down so I would remember.

I'd already closed all the links, so I had to find the sources I used again.
I ran across a line in an [ArchWiki page about a different Thinkpad](https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_Edge_13):

> If you are using the rtl8192ce module, you may experience some intermittent deauthentication issues with newer kernels (tested on 3.4.4-2-ARCH). The reason is because **the BIOS is turning off the wireless card when the BIOS deems it to be "inactive."**

Of course!
That's why disabling power management at a higher level didn't fix the problem: the underlying BIOS was disabling it.
I booted into BIOS and disabled PCIe power management.
On reboot, I was able to connect to my network with no problems.
I haven't done a full or comprehensive stress test yet, but things are looking up.

# Conclusion

The final solution to constant disconnects on Wifi turned out to be disabling power management at the BIOS.
This worked on my refurbished Lenovo Thinkpad T410 running Linux Mint 4.4 with a RTL8191ce wireless adapter.
As usual, your mileage my vary.
I recommend checking the BIOS solution if the problem persists across operating systems, even if one operating system doesn't seem as effected.
If the BIOS setting isn't available, try setting power management settings at the OS level.
If the problem persists even when plugged in or the network is in use, consider getting a different wireless adapter.
But be careful when shopping, as some manufacturers only allow whitelisted chips.

## References

It's unlikely my story-based approach to debugging helped solve your issue.
So here's a list of the articles, reference sheets, and wiki pages I found most helpful:

* [Ubuntu Wireless Troubleshooting Guide](https://help.ubuntu.com/community/WifiDocs/WirelessTroubleShootingGuide): This guide helps troubleshoot network problems on Ubuntu, ranging from hardware and driver problems to ip configuration. Nothing here is device specific, and is a good starting place for triaging the problem.
* [Guide to Kernel Modules, at the Arch Wiki](https://wiki.archlinux.org/index.php/Kernel_modules): Gives an overview of how to work with kernel modules (drivers) to ensure they are running correctly.
* [The RTL819x Chip Set, at the Debian Wiki](https://wiki.debian.org/rtl819x): This page shows how to install the drivers for certain RealTek adapters on Debian. It also links to the firmware and lists supported devices.
* [Lenovo Thinkpad T410, at ThinkWiki](https://www.thinkwiki.org/wiki/Category:T410): Links to information specific to the hardware on my machine.
