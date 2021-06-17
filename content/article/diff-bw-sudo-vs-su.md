---
title: "Diff between Sudo vs Su"
date: 2021-06-17T23:42:26+08:00
draft: false

categories: [linux]
tags: [sudo, su, linux]
toc: false
author: "Avradeep"
---
# Difference Between Sudo and Su

`sudo` requires the password of the current user whereas `su` requires the password of the target user.

`su` is an older but much more fully-featured command included in all Linux distributions. It is the traditional wat to switch to root account

For this reason, all Ubuntu-based releases are sudo-only, meaning the root account is not active by default. While installing an Ubuntu OS, you create a user automatically labeled as part of the sudoers group. However, there is no root account setup.

To enable the root user, you need to activate it manually.

This is also the reason you might see that commands like `su` or `su -` fail in Ubuntu and MacOS systems.

To activate the root user run:

```
sudo passwd root
```

Next, the output asks to set the password for the root user. Type and retype a secure password, then hit Enter. The system should notify you the password has been updated successfully.

su can also work as sudo and run a single command

```
su -c [command]
```

Reference:

https://phoenixnap.com/kb/sudo-vs-su-differences