---
title: "Running Linux on my Thinkpad Z13"
date: 2023-07-24T21:15:19-07:00
draft: true
---

This is basically just a walk-through of my install script for the Lenovo Z13
Thinkpad. I'm using pop_os!, but the commands/ideas should work for any Debian
based distro.

## Basics

On pop_os, the "Firmware" app automatically had Lenovo Z13 updates ready for me
to install.

Of course, doing a `sudo apt update && sudo apt upgrade` is always a good idea
before doing anything else with apt on a clean install.

Then you'll want to install the Z13 package included on Lenovo's Ubuntu
image[^1]:

[^1]: Learned this while reading Martin's great review
[here](https://wimpysworld.com/posts/why-i-chose-the-thinkpad-z13-as-my-linux-laptop/)

```bash
sudo apt install oem-sutton.newell-abe-meta
```

If you want to be able to adjust the force touchpad settings (you probably do)
download Lenovo's janky-but-functional tool [here](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-z-series-laptops/thinkpad-z13-type-21d2-21d3/downloads/ds561548-elan-haptic-pad-settings-tool-for-linux-thinkpad-z13-gen-1-z16-gen-1).

## Fingerprint Sensor

```bash
sudo apt install fprintd libpam-fprintd
sudo pam-auth-update
```

You'll see this screen:

![pam-auth-update screen](./pam-auth-fingerprint.png)

You want to enable fingerprint authentication. After doing that I was able to
set up my fingerprints in the settings app and use it to auth sudo and
login. 

The only problem I've noticed is that if I use my fingerprint for my first
login after powering on my device, I have to type in my password anyways when I
start my browser to login to my keyring.

One note: the fingerprint sensor on the Z13 is not as fast as the one you might
have on your phone. You'll want to hold your finger to the sensor a bit longer
in order for it to successfully read your fingerprint consistently.

## Helpful Resources

### Audio Problems

Sometimes I found that no sound would come from my Z13's speakers. The f1/2/3
audio buttons also wouldn't work. Basically the system behaved as if it had no
speakers. For this issue, I found
[this](https://support.system76.com/articles/audio/) article by System76 to be
pretty helpful. What I did to improve my issue was:

- I reinstalled/restarted pipewire per the instructions in the article above.
- I disabled hibernation/fast startup/secure boot.
