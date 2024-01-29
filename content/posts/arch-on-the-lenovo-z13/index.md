---
title: "Running Linux on my Thinkpad Z13 Part 2: Arch"
summary: "I distro hopped to Arch because I was curious about the rolling
release experience."
description: "I distro hopped to Arch because I was curious about the rolling
release experience."
date: 2023-12-09T19:17:19-07:00
lastmod: 2024-01-28
draft: false
---

I really enjoyed pop_os! on my Z13, but I was experiencing some issues when
docking the laptop with multiple monitors (frequent crashes, hang-ups). I was
also just curious about using an Arch based distro and an updated version of
Gnome, so I installed Endeavour OS.

I've had to make enough adjustments that I think another post is warranted.

## Battery

While battery life was okay with just `power-profiles-daemon` (included with Gnome
by default), it didn't seem to be quite as great as I thought it could be when
doing low power tasks like basic text editing in the terminal. Because of that,
I uninstalled it and installed tlp like so:

```bash
# Useful guide: https://linrunner.de/tlp/faq/ppd.html
yay -R power-profiles-daemon
yay -S tlp
sudo sed -i "s/#\?PLATFORM_PROFILE_ON_AC=.*/PLATFORM_PROFILE_ON_AC=performance/1" /etc/tlp.conf
sudo sed -i "s/#\?PLATFORM_PROFILE_ON_BAT=.*/PLATFORM_PROFILE_ON_BAT=low-power/1" /etc/tlp.conf
systemctl enable tlp.service
systemctl mask systemd-rfkill.service systemd-rfkill.socket
echo "Run sudo tlp start or restart"
```

If you're on the Z13, then the platform profiles should work fine for you, but
if not you can run `sudo tlp-stat -p` to see what your options are.

Anyways after these changes I'm seeing a drastically improved battery life for
low-powered tasks (5-6 hour estimates became 8-10 hour estimates when around
75-85%). This seems more in line with what I was getting on pop_os!.

## Sleep Issues

The biggest sleep issue I had was that sometimes the laptop just wouldn't wake
up or go to sleep properly. E.g. one time I had an experience that resembled a
common issue found on Windows laptops with Modern Standby. I closed the lid and
put the laptop in its case, only to return and find a hot laptop with an
unresponsive black screen that I had to force shutdown. This was pretty rare and
I didn't notice any obviously actionable logs in journalctl, but it didn't feel
great. It seemed to be exacerbated when docking/undocking the laptop, but I
never figured out a root cause.

Fortunately, the troubleshooting section of the relevant arch wiki page
[^sleepissues] gives downgrading to the LTS kernel as an unofficial solution,
and it seems to be working for me. All I had to do was run `yay -S linux-lts` to
install the LTS latest kernel, and it was automatically added as the top option
of my systemd-boot menu.

[^sleepissues]: There were loads of troubleshooting techniques that were too
advanced for me, so that's why I figured I'd give the LTS kernel a shot first:
https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Suspend/hibernate_does_not_work,_or_does_not_work_consistently

### Wi-Fi Slowdown

The issue where the WiFi speeds dip dramatically after sleep also returned. My fix
this time around is very similar to the last one I posted.

I copied this script:

```bash
#!/bin/sh

PATH=/sbin:/usr/sbin:/bin:/usr/bin

case "$1" in
	# code execution BEFORE sleeping/hibernating/suspending
	pre)
	;;
	# code execution AFTER resuming
	# Idea from the following blog post:
	# https://null.rip/2022/09/linux-on-the-thinkpad-z13-all-amd-almost-there/
	post)
		echo "Toggling wlan twice"
		rfkill toggle wlan
		rfkill toggle wlan
	;;
esac

exit 0
```

to `/usr/lib/systemd/system-sleep/`[^3]. You can name it whatever you like, it just has
to be executable. I figured deleting the module like I posted
before was overkill since the problem is also solved by just toggling the Wi-Fi
on and off.

<!-- The LTS kernel may have improved this. I should watch and see how often it
happens now. -->

Despite adding this script and confirming that it runs on resuming from sleep, I
still notice the internet speed slowing from time to time. Turning the wlan
on and off manually fixes it, it's just annoying!

[^3]: The [Arch
Wiki](https://wiki.archlinux.org/title/Power_management#Hooks_in_/usr/lib/systemd/system-sleep)
says to use `/usr/lib/systemd/system-sleep/`, but this
[AskUbuntu](https://askubuntu.com/questions/1313479/correct-way-to-execute-a-script-on-resume-from-suspend)
post used `/lib/systemd/system-sleep`. Since I'm using Arch I'm going with the
Arch wiki, but it seems like either would work (and I definitely used the
AskUbuntu way on pop_os!).

## Fingerprint Sensor

As before I started with installing fprintd (`yay -S fprintd`). Just doing this
enabled Gnome's fingerprint support and the lock screen worked well.

The pam files were different on Arch so I had to do a little bit more digging to
get the fingerprint scanner working well for authing other requests. There
wasn't a magic command to run that just figured it out for me (which was awesome
on pop_os! and I wish there had been one in this case), but the Arch wiki[^1]
was extremely helpful as always.

For my first attempt to get fingerprint auth working with sudo, my `/etc/pam.d/sudo`
looked like this:

```bash
#%PAM-1.0
auth	  	sufficient 	pam_fprintd.so # addded line
auth		include		system-auth
account		include		system-auth
session		include		system-auth
```

This didn't work super well with `yay`, as I'd press
<kbd>Ctrl</kbd>+<kbd>C</kbd> to get past the fingerprint auth (when using the
laptop docked, for example), and the whole program would just exit. This
behavior would also trigger pam_faillock, locking me out of authing any sudo
commands for 15 minutes each time. Super annoying.

Reading just a little further down on the page, I removed the above edit and
instead edited `/etc/pam.d/system-auth` like so:

```bash
#%PAM-1.0

# deny=10 because the default of 3 is constantly getting me locked out
auth       required                    pam_faillock.so      preauth deny=10
# Optionally use requisite above if you do not want to prompt for the password
# on locked accounts.
-auth      [success=3 default=ignore]  pam_systemd_home.so
auth       [success=2 default=ignore]  pam_unix.so          try_first_pass nullok likeauth
auth       [success=1 default=bad]     pam_fprintd.so
auth       [default=die]               pam_faillock.so      authfail
auth       optional                    pam_permit.so
auth       required                    pam_env.so
auth       required                    pam_faillock.so      authsucc
# If you drop the above call to pam_faillock.so the lock will be done also
# on non-consecutive authentication failures.

-account   [success=1 default=ignore]  pam_systemd_home.so
account    required                    pam_unix.so
account    optional                    pam_permit.so
account    required                    pam_time.so

-password  [success=1 default=ignore]  pam_systemd_home.so
password   required                    pam_unix.so          try_first_pass nullok shadow
password   optional                    pam_permit.so

-session   optional                    pam_systemd_home.so
session    required                    pam_limits.so
session    required                    pam_unix.so
session    optional                    pam_permit.so
```

As you can see, rather than try to auth with the fingerprint first, we first try
to auth regularly and allow the stack to continue normally if an empty password
is supplied. This means that whenever I want to use my fingerprint, I can just press
<kbd>Enter</kbd> first. I can live with that.

There is one caveat to note: `system-auth` is included in a ton of other places,
so this may not be the most secure option, but I think I'm fine with allowing
most situations to be authorized using my fingerprint when possible.

[^1]: https://wiki.archlinux.org/title/Fprint

### Debugging Tip

One factor that confused my debugging process was the fact that my keyring
password was set to the password I used to encrypt my disk. I'm not sure if I
did that or the installer did, but I initially thought that this behavior was
another symptom of triggering the lockout. I ended up installing seahorse (`yay
-S seahorse`) to change the password on my keyring to what I expected it to be
(the same as my login password set via `passwd`). This works until I restart,
and then it's set back to my LUKS password. Not sure how to change this but I don't
mind it I guess.

## Laggy Touchpad

I find that sometimes my touchpad is extremely laggy, and it seems significantly
worse when I turn on my laptop while connected to power. If I disconnect my AC
adapter and reboot, the touchpad returns to normal.

Skimming the internet, it seems like it could be because I'm using a cheap USB-C
charger I bought in another country. I also noticed the problem in Windows when I went to
check if the problem was due to a missing firmware update (there were some
updates available, but they didn't fix the issue).

I don't really have a ton of insight for this one, I'll just go back to using my
good chargers when I get back home ¯\_(ツ)_/¯

## Misc

### auto-cpufreq

This post[^2] mentioned using auto-cpufreq, but it didn't work for me. For some
reason when I had it enabled the trackpoint would register random right clicks
when the laptop was docked.

### mDNS

To get mDNS working (the feature that enables you to go to a host-name.local
service you may be running on your LAN), I had to:

1. Open the firewall application
   and assign my network to the "home" zone, where mDNS is enabled by default. I
   figured this out while browsing [this Endeavour OS wiki
   page](https://discovery.endeavouros.com/applications/firewalld/).

   Example:

   ![firewall application example](./firewall_screenshot.png)

2. Enable a daemon that provides mDNS. I used Avahi, so I followed the
   instructions [here](https://wiki.archlinux.org/title/Avahi). You should also
   be able to use resolved following the instructions
   [here](https://wiki.archlinux.org/title/Systemd-resolved#mDNS) (it just
   seemed like more work).

[^2]: One of the most helpful posts I looked at:
[https://blog.15cm.net/2022/08/21/my_arch_linux_setup_on_thinkpad_z13_gen_1/](https://blog.15cm.net/2022/08/21/my_arch_linux_setup_on_thinkpad_z13_gen_1/)

### Brave & Wayland

If you're using `brave-bin` from the AUR, then you can add any Brave
flags[^brave_flags] you
want to use permanently to `$XDG_CONFIG_HOME/brave-flags.conf`. Here's what I
use to make Brave use Wayland and allow the two-finger swipe gesture to go back
and forth in history.

```bash
--gtk-version=4
--ozone-platform-hint=auto
--enable-features=TouchpadOverscrollHistoryNavigation
--ozone-platform=wayland
```

The ozone-platform setting makes the title bar weirdly smaller, but that's the
only downside I've noticed. Ultimately, I switched back to Firefox (specifically
Librefox) since it worked well with Wayland out of the box.

[^brave_flags]: The Chromium page of the Arch wiki was extremely helpful here:
https://wiki.archlinux.org/title/Chromium


### Secure Boot

I finally set up secure boot. It was easy to follow the arch wiki.
