---
title: "Running Linux on the Thinkpad Z13"
date: 2023-07-24T21:15:19-07:00
draft: true
---

This is basically just a walk-through of my install script for the Lenovo Z13
Thinkpad. I use pop_os!, but the commands should work for any Debian based distro.

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
