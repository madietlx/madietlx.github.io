---
layout: post
title:  "Making use of Ubuntu's `unattended-upgrades` package to keep servers up-to-date without breaking them (hopefully)"
date:   2023-01-15 18:30:00 +0100
excerpt_separator: <!--more-->
---

During my life, I've mainly seen and used three ways to keep Ubuntu servers 🐧 up-to-date:

1. Manually (not recommended 🛑 time-consuming and error-prone)
2. Cron job via `/etc/crontab` (works, but still not great...)
3. `unattended-upgrades` (the easiest and most robust way!)
<!--more-->

The first option, logging in to each server and running something like `sudo apt update && sudo apt upgrade`, followed by `sudo reboot`, is obviously the one that will not scale at all. The more servers you have, the more likely you will delay updates because it gets time-consuming.

The next best thing I've come across is a regular cron job, running as `root`, to execute something like `apt-get -y check && apt-get -y update && apt-get --with-new-pkgs -y upgrade && apt-get -y autoremove && apt-get -y autoclean; reboot`. This worked for me for quite a while and quite well, but

- the series of commands always seemed unnecessarily complex to me
- and *everyone* has a different opinion on which commands you should and should not run to keep your Ubuntu servers up-to-date without breaking them 😕

The easiest and most robust way, the one that I stuck to, is making use of Ubuntu's `unattended-upgrades` package. I mainly run the following commands to tweak my configs a little bit more so that

```sh
sed -E -i 's/(APT::Periodic::Download-Upgradeable-Packages )"0";/\1"1";/' /etc/apt/apt.conf.d/10periodic
sed -E -i 's/(APT::Periodic::AutocleanInterval )"0";/\1"7";/' /etc/apt/apt.conf.d/10periodic
sed -E -i 's/\/\/(\s*"\$\{distro_id\}:\$\{distro_codename\}-updates";)/\1/' /etc/apt/apt.conf.d/50unattended-upgrades
sed -E -i 's/\/\/(Unattended-Upgrade::Remove-Unused-Dependencies )"false";/\1"true";/' /etc/apt/apt.conf.d/50unattended-upgrades
```

- upgradeable packages are downloaded (daily) ✔️
- the cache gets cleaned (weekly) and does not grow out of control ✔️
- standard `updates` are installed (not just security updates) ✔️
- unused packages are removed after the upgrade ✔️

`$ cat /etc/apt/apt.conf.d/10periodic`

```conf
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
```

`$ cat /etc/apt/apt.conf.d/50unattended-upgrades`

```conf
// Automatically upgrade packages from these (origin:archive) pairs [...]
Unattended-Upgrade::Allowed-Origins {
	"${distro_id}:${distro_codename}";
	"${distro_id}:${distro_codename}-security";
	[...]
	"${distro_id}ESMApps:${distro_codename}-apps-security";
	"${distro_id}ESM:${distro_codename}-infra-security";
	"${distro_id}:${distro_codename}-updates";
//	"${distro_id}:${distro_codename}-proposed";
//	"${distro_id}:${distro_codename}-backports";
};

[...]

// Do automatic removal of unused packages after the upgrade
// (equivalent to apt-get autoremove)
Unattended-Upgrade::Remove-Unused-Dependencies "true";

[...]
```

The only thing that remains in my `/etc/crontab` is a `reboot` command, executed as `root` on the weekend---which might not be a feasible option for you but is perfectly fine for me.

Please also note ⚠️ that I typically install a limited set of packages on my Ubuntu servers. Thus, I never had any problems with automatically upgrading installed packages and kernels for many years. The risk of a system running old and vulnerable software seems way higher than the risk of breaking something due to unattended upgrades.

P.S. In the next blog posts, I will write about a) upgrading Docker 🐳 and b) the conffile prompt.
