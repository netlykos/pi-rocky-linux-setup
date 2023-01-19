# Setup commands for raspberry pi rocky linux

This documents the commands executed after logging into a fresh rocky linux install. Download the Rocky Linux Raspberry PI (aarch64) image from (https://rockylinux.org/alternative-images/)[https://rockylinux.org/alternative-images/]. After flashing the image to a SD card and plugging it into the PI, the below commands can be used to configure the PI as necessary.
## Software system
```sh
sudo rootfs-expand; \
sudo dnf update -y && \
sudo dnf install -y epel-release && \
sudo /usr/bin/crb enable \
sudo dnf install -y fortune-mod mlocate net-tools bind-utils traceroute podman podman-compose podman-docker xauth gvim
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

Follow the fail2ban guide

```sh
sudo dnf install -y fail2ban
```

- [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9)

## Acknowledgements

- [https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt](https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt)
- [https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/](https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/)
- [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9)
- [https://bobcares.com/blog/fail2ban-unban-ip/](https://bobcares.com/blog/fail2ban-unban-ip/)

