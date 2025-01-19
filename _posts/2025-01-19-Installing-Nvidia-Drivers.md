---
layout: post
title: 'Installing Nvidia drivers on PopOS'
date: 2025-01-19 20:58
---

# Installing Nvidia drivers
The reason behind writing this is because I wanted a way to chronicle how to install it. This is specificall for Ubuntu based distros. But it is applicable to all which are supported.[^1]

Make sure that gcc is installed and collect system information with:

```
uname -m && cat /etc/os-release
```

This information will be used to determine what you need to download.

```
sudo apt-get install linux-headers-$(uname -r)
```

Install the proper headers. After that follow [this](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/#network-repo-installation-for-ubuntu) to install whatever is neccessary. Update and reboot the system.

You're then told to run the persistence daemon.

```
systemctl start persistenced
```

The above command fails. Which is what tripped me up. I did some searching and found out that the service name is actually `nvidia-persistenced.service`. Making the command

```
systemctl status nvidia-persistenced.service
```



[^1]: [A full list of supported distros](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/#ubuntu-installation-local)
