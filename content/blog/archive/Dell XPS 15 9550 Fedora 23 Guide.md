---
title: Dell XPS 15 9550 Fedora 23 Guide
date: 2016-04-17T19:58:40-04:00
draft: false
---

I'm blessed with a job that's actively looking to step up their hardware game. So after a few nights of googling, I went to our IT guy and I was able to pursuade him to order the new [Dell XPS 15 Non-touch 9550 P56F](http://www.dell.com/us/business/p/xps-15-9550-laptop/pd?oc=cax15w10s1630&model_id=xps-15-9550-laptop). My model has a Intel Core i7-6700HQ with 16GB of DDR4 RAM and a 512gb SSD on a PCI-e bus. Sickeningly awesome system. It hangs with my MBP but I still miss the trackpad. The MBP feels better and is less plastically, but the internals are better on the Dell side, and while it's a little thicker than the MBP, it's hardly noticable and Dell hides it pretty well.

It's a good Linux laptop. Running Fedora 23, it pretty much works entirely out of the box. Battery life *seems* roughly comparable to my MBP, but we'll see how that pans out as I use it.

## My thoughts so far:

- Wireless: The Broadcom Wireless AC works well out of the gate. it connects to my 5GHz network, reconnects after standby, and seems speedy and reliable.
- Bluetooth: Scans, seemingly works
- Battery: Gnome reports 4-6 hours at 100% depending... so far so good
- UEFI: Was able to perform the UEFI install and everything worked, did not try Secure Boot. Was having issues with hibernation in UEFI, might be user error. Regardless currently running in Legacy Mode.
- Suspend: Standby works reliably, wifi reconnects. Hibernation is a little flakier, tried configuring for hybrid-sleep, but couldn't't get it to suspend but it would save RAM to disk and resume from hibernation
- Touchpad: Multitouch works! It's very poorly configured, and I haven't had much luck taming it, but with touchegg I'm able to do two finger right click, four finger swipe to show the activities menu... but I haven't had a lot of luck improving on that, must be a software thing because the touchpad seems well supported by the OS.

i still need to test a few things, and would like to get a better measure on the battery life. I'm interested to get the [Dell Thunderbolt 3 Dock](http://amzn.com/B01C8PHW32) and see how that goes. Like the Surface Dock, it gives you one cable that charges the laptop and provides dual-head monitors, USB, Ethernet and everything.

## Provisioning

You can get this to work with UEFI, but I ended up going Legacy boot since I was having problems getting standby and hibernate to work when booting through EFI. Also, make sure you set up a swap partion at least the size of your memory so hibernation will work. 

## Bumblebee/Nvidia Optimus

The XPS 15 comes with both an Intel and Nvidia graphics controllers. The Intel GPU does the desktop stuff fine, but if you want to do any gaming you'll want to switch over to the Nvidia. This isn't hard to do, but it seems to render everything to a virtual frame buffer and probably isn't as perforamnt as just using the nvidia card. It's easy to setup though: 

* [Bumblebee Indicator Gnome Extension](https://extensions.gnome.org/extension/843/bumblebee-indicator/)

* [Bumblebee Installation Guide for Fedora](https://fedoraproject.org/wiki/Bumblebee)

Basically, install as follows..

After running all updates and restarting to ensure you're running the latest kernel available to your system:

```
$ dnf -y --nogpgcheck install http://install.linux.ncsu.edu/pub/yum/itecs/public/bumblebee/fedora23/noarch/bumblebee-release-1.2-1.noarch.rpm
$ dnf -y --nogpgcheck install http://install.linux.ncsu.edu/pub/yum/itecs/public/bumblebee-nonfree/fedora23/noarch/bumblebee-nonfree-release-1.2-1.noarch.rpm
$ dnf install bumblebee-nvidia bbswitch-dkms VirtualGL.x86_64 VirtualGL.i686 primus.x86_64 primus.i686 kernel-devel
reboot
```

When you want to use the Nvidia GPU, you need to pass the command through optirun, like follows:

```
$ optirun glxspheres64
```

If you're trying to run a game through Steam, be sure to update the launch command as follows:

```
optirun $COMMAND (or whatever)
```

## Bluetooth

Bluetooth did not work out the box for me, but it would scan and discover devices. Ultimately, I had to go through a whole deal to get it working, mostly following [this guide](https://outhereinthefield.wordpress.com/2014/03/01/ubuntu-13-10-and-bluetooth-on-broadcom-bcm43142-wifibt-combo-adapter/)

My struggle is your gain, if you have a Broadcom BCM2045 A0 you should be able to use something like the following to get this working:

```
$ curl https://www.dropbox.com/s/3ajj26zj3n79pcc/BCM-0a5c-6410.hcd?dl=1 /lib/firmware/brcm/BCM-0a5c-6410.hcd
```

You should test this by turning off your PC and turning it back on, as opposed to just restarting. You can determine where exactly to copy the firmware by using *dmesg | grep btusb*.

## Touch/Gestures

This is a biggie for me, and a pain to setup. Luckily, the synaptics touchpad provided with the laptop supports all sorts of gestures in Linux but you'll have to make some tweaks to get them to work. I used touchegg and touchegg-gce to configure three and four button gestures, and synclient to enable two-finger right click. 

Removing xorg-x11-drv-libinput is needed to favor the synaptics module.

```
$ sudo  dnf remove xorg-x11-drv-libinput
$ dnf install xorg-x11-drv-synaptics
```

Let's configure the synaptics driver so we can take advantage of touchegg. Find the section with MatchDriver "synaptics" and add the options in mine:

*/usr/share/X11/xorg.conf.d/50-synaptics.conf*

```
Section "InputClass"
        Identifier "Default clickpad buttons"
        MatchDriver "synaptics"
        Option "SoftButtonAreas" "50% 0 82% 0 0 0 0 0"
        Option "SecondarySoftButtonAreas" "58% 0 0 8% 42% 58% 0 8%"
        Option "TapButton3" "0"
        Option "ClickFinger3" "0"
EndSection
```

Let's configure touchegg with some sane options...

*~/.config/touchegg/touchegg.conf*

```
<touchégg>
        <settings>
                <property name="composed_gestures_time">0</property>
        </settings>
        <application name="All">
                <gesture type="DRAG" fingers="3" direction="RIGHT">
                        <action type="SEND_KEYS">Alt+Left</action>
                </gesture>
                <gesture type="DRAG" fingers="3" direction="LEFT">
                        <action type="SEND_KEYS">Alt+Right</action>
                </gesture>
                <gesture type="DRAG" fingers="3" direction="UP">
                        <action type="SEND_KEYS">Control+Alt+Up</action>
                </gesture>
                <gesture type="DRAG" fingers="4" direction="UP">
                        <action type="SEND_KEYS">Super+w</action>
                </gesture>
                <gesture type="DRAG" fingers="4" direction="DOWN">
                        <action type="SEND_KEYS">Control+w</action>
                </gesture>
                <gesture type="DRAG" fingers="3" direction="DOWN">
                        <action type="SEND_KEYS">Control+Alt+Down</action>
                </gesture>
        </application>
</touchégg>
```

And finally, let's configure touchegg to start automatically at login:

*/etc/X11/xinit/xinitrc.d/touchegg.sh*

```
#!/bin/sh -x
touchegg&
```

This will give you three finger left and right as browser back and forth, three finger up and down to switch spaces, four finger up to enter the activities menu, and four finger down to close a browser tab. This isn't quite the Magic Trackpad, but it's close.

## Standby and Power Management

As mentioned before, make sure you have a swap partition big enough to store the contents of your RAM. You'll also need to tell the login manager, GDM, what to do when you close the lid:

Edit */etc/systemd/logind.conf* and uncomment the line...

```
HandleLidSwitch=suspend
```

You can also set this to *hibernate* and *hybrid-sleep*, but hibernation doesn't really seem to work well.

Optionally, you can install the [tlp package](https://wiki.archlinux.org/index.php/TLP) that supposedly provides better power management and automatic backlight control. To install...

```
$ dnf install tlp tlp-rdw
```

I'll update this guide as I add more. Good luck, let me know how it goes!
