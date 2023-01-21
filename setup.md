# Setup commands for raspberry pi rocky linux

This documents the commands executed after logging into a fresh rocky linux install. Download the Rocky Linux Raspberry PI (aarch64) image from (https://rockylinux.org/alternative-images/)[https://rockylinux.org/alternative-images/]. After flashing the image to a SD card and plugging it into the PI, the below commands can be used to configure the PI as necessary.
## Software system
```sh
sudo rootfs-expand; \
sudo dnf update -y && \
sudo dnf install -y epel-release && \
sudo /usr/bin/crb enable \
sudo dnf install -y fortune-mod mlocate net-tools bind-utils traceroute rsync podman podman-compose podman-docker xauth gvim
```

## Setup user
```sh
sudo useradd -u 10000 -g 100 -G wheel -c "Adi B q=)" -s /bin/bash netlykos
sudo passwd netlykos
sudo su - netlykos
sudo userdel -f -r rocky
```

### Copy ssh keys
```sh
ssh-copy-id netlykos@xxx.xxx.xxx.xxx
```

## Disable ssh password login
```sh
sudo vim /etc/ssh/sshd_config
```

Look for the string ``PasswordAuthentication`` and set the value to ``no``.

## Setup sudo for user
```sh
sudo visudo
```

## Set hostname

```sh
sudo hostnamectl set-hostname xxxxxxx
```

## Setup fail2ban

Run the below to setup fail2ban on system

```sh
sudo dnf install -y fail2ban
cd /etc/fail2ban
sudo cp jail.conf jail.local
sudo vim jail.local
```

In the file ``jail.local``, make the following changes:

- Add the line ``enabled = true`` under the ``[sshd]`` section
- Change the ``[default]`` backend from auto to systemd by changing the line ``backend = auto`` to ``backend = systemd``
- Change the ``[default]`` section to include/set the following values as so:
```
bantime.increment = true
bantime.rndtime = 86400
bantime.multipliers = 1 5 30 60 300 720 1440 2880
bantime  = 60m
findtime  = 60m
maxretry = 2
```

To enable and start the fail2ban service, execute the below:
```sh
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

To verify that the application started as expected (without any errors), execute the below:
```sh
sudo systemctl status fail2ban
```

## Change selinux

By default selinux is enforced, this can cause issues when using podman and/or having to play around with namespaces when using the -v option. SELinux can be changed to permissive mode where warnings are emitted instead of failures. 

```sh
sudo vim /etc/sysconfig/selinux
```
Change ``SELINUX=enforcing`` to ``SELINUX=permissive``

```sh
reboot
```

## Acknowledgements

- [https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt](https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt)
- [https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/](https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/)
- [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9)
- [https://bobcares.com/blog/fail2ban-unban-ip/](https://bobcares.com/blog/fail2ban-unban-ip/)
- [https://www.how2shout.com/linux/how-to-disable-or-turn-off-selinux-on-rocky-linux-8/](https://www.how2shout.com/linux/how-to-disable-or-turn-off-selinux-on-rocky-linux-8/)
