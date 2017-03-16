---
layout: post
title: LUKS disk encryption and Nvidia drivers
---

Sometimes the combination of two apparently unrelated components can turn out to have surprising results.

In this case I had just re-installed Ubuntu on my laptop together with full disk encryption and everything was fine.

However, after upgrading to the latest proprietary Nvidia drivers (378.13) and restarting, I was not able to enter the disk encryption password anymore - the splash screen appeared frozen:

<img src="/images/posts/2017-3-16/luks_plymouth.jpg">

Entering the password 'blindly' and pressing enter did not work either.

#### Workaround

Luckily, the next time you reboot, it is possible to select recovery mode from Grub menu, which asks for the encryption password in a text-only (and functioning) prompt.

After entering the password, selecting 'resume normal boot' from the recovery menu will take you to the login screen, like nothing ever happened..until next time.

#### A Slightly better workaround

Since doing that every time is slightly annoying and time-consuming, it is more convenient to update /etc/default/grub and remove "splash" from the following line:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

After that, for the changes to take effect:
```
$ sudo update-grub
```

and the next boot you should be greeted by a text prompt right away.

Not great, but works.

[Turns out](https://bugs.launchpad.net/ubuntu/+source/plymouth/+bug/1386005) it is not a new bug at all, but still around.