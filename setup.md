# Raspberry PI RockyLinux 9 Setup

<!-- TOC -->

- [Raspberry PI RockyLinux 9 Setup](#raspberry-pi-rockylinux-9-setup)
  - [System setup](#system-setup)
    - [Setup user](#setup-user)
    - [Copy ssh keys](#copy-ssh-keys)
    - [Disable ssh password login](#disable-ssh-password-login)
    - [Setup sudo for user](#setup-sudo-for-user)
    - [Set hostname](#set-hostname)
    - [Setup fail2ban](#setup-fail2ban)
    - [Change selinux](#change-selinux)
    - [Podman setup](#podman-setup)
    - [Host registration with ddclient](#host-registration-with-ddclient)
  - [Software setup](#software-setup)
    - [Apache httpd server](#apache-httpd-server)
      - [Certbot setup](#certbot-setup)
      - [Configure fail2ban](#configure-fail2ban)
    - [Setup Wireguard Server using podman](#setup-wireguard-server-using-podman)
    - [Setup TimeMachine backup](#setup-timemachine-backup)
    - [Homebridge setup](#homebridge-setup)
      - [Homebridge: Simplisafe setup](#homebridge-simplisafe-setup)
    - [WiFi hotspot for VPN connection](#wifi-hotspot-for-vpn-connection)
    - [Scrypted setup](#scrypted-setup)
  - [Acknowledgements](#acknowledgements)

<!-- /TOC -->

## System setup

```sh
sudo rootfs-expand; \
sudo dnf update -y && \
sudo dnf install -y epel-release && \
sudo /usr/bin/crb enable && \
sudo dnf install -y fortune-mod mlocate net-tools bind-utils \
  traceroute rsync podman podman-compose podman-docker xauth \
  gvim rsync bzip2 bunzip2 netcat p7zip
```

### Setup user

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

### Disable ssh password login

```sh
sudo vim /etc/ssh/sshd_config
```

Look for the string ``PasswordAuthentication`` and set the value to ``no``.

### Setup sudo for user

```sh
sudo visudo
```

### Set hostname

```sh
sudo hostnamectl set-hostname xxx.xxx.xxx
```

### Setup fail2ban

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

```text
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

### Change selinux

By default selinux is enforced, this can cause issues when using podman and/or having to play around with namespaces when using the -v option. SELinux can be changed to permissive mode where warnings are emitted instead of failures.

```sh
sudo vim /etc/sysconfig/selinux
```

Change ``SELINUX=enforcing`` to ``SELINUX=permissive``

```sh
reboot
```

### Podman setup

Enable linger for podman processes.

```sh
sudo loginctl enable-linger $(id -un)
```

### Host registration with ddclient

Use the below to setup ddclient as systemd service to manage host registration with [http://pairdomains.com](http://pairdomains.com). Install and configure ddclient on the local machine:

```sh
sudo dnf install -y ddclient
sudo vim /etc/ddclient.conf /usr/lib/systemd/system/ddclient.service
```

Edit the file ``/etc/ddclient.conf`` and replace the file content with the below:

```text
# check every 300 seconds
daemon=300
# log update msgs to syslog
syslog=yes
# record PID in file.
pid=/var/run/ddclient/ddclient.pid
# enable the ssl library
ssl=yes

protocol=dyndns2
use=web,web=https://myip.pairdomains.com/
web-skip='Current IP Address: '
server=dynamic.pairdomains.com, \
login=pairdomains,              \
password=XXXXXXXX               \
XXX.XXX.XXX
```

Edit the file ``/usr/lib/systemd/system/ddclient.service`` and replace the file content with the below:

 ```text
[Unit]
Description=A Perl Client Used To Update Dynamic DNS
After=syslog.target network-online.target nss-lookup.target

[Service]
User=ddclient
Group=ddclient
Type=forking
PIDFile=/run/ddclient/ddclient.pid
EnvironmentFile=-/etc/sysconfig/ddclient
ExecStartPre=/bin/touch /var/cache/ddclient/ddclient.cache
ExecStart=/usr/sbin/ddclient -file /etc/ddclient.conf -verbose $DDCLIENT_OPTIONS
ExecStop=/usr/bin/pkill -SIGKILL -P /var/run/ddclient/ddclient.pid

[Install]
WantedBy=multi-user.target
```

Enable and start the ddclient service

```sh
sudo systemctl daemon-reload
sudo systemctl enable ddclient
sudo systemctl start ddclient
sudo systemctl status ddclient
sudo journalctl -fu ddclient.service 
```

## Software setup

### Apache httpd server

Install apache, mod_ssl and certbot for SSL certificate.

```sh
sudo dnf install -y httpd mod_ssl certbot python3-certbot-apache
```

Create a virtual host entry for the domain that will be hosted on the server using the below content:

```domain.conf
<VirtualHost *:80>
  ServerName xxx.xxx.xxx
  DocumentRoot /var/www/html
  ServerAlias xxx.xxx.xxx
  ErrorLog /var/www/error.log
  CustomLog /var/www/requests.log combined
</VirtualHost>
```

After the service is running, change the default index.html file. After the welcome page has been modified,  modify the firewall rules to accept requests on httpd ports (80, 443).

```sh
sudo systemctl enable httpd
sudo systemctl start httpd
sudo echo "<html><head><title>It's alive...</title></head><body>It's alive...</body></html>" > /var/www/html/index.html
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

#### Certbot setup

```sh
sudo certbot -v --apache -d xxx.xxx.xxx
```

Create a certbot renewal timer based on documentation found [https://stevenwestmoreland.com/2017/11/renewing-certbot-certificates-using-a-systemd-timer.html](https://stevenwestmoreland.com/2017/11/renewing-certbot-certificates-using-a-systemd-timer.html).

Create the file ``/etc/systemd/system/certbot-renewal.service`` with the content below:

```certbot-renewal.service
[Unit]
Description=Certbot Renewal

[Service]
ExecStart=/usr/bin/certbot renew --post-hook "systemctl restart httpd"
```

The above service executes the certbot renew command and restarts the httpd service after the renewal process has completed.

Timer unit files contain information about a timer controlled and supervised by systemd. By default, a service with the same name as the timer is activated.

Create a timer unit file ``/etc/systemd/system/certbot-renewal.timer`` in the same directory as the service file. The configuration below will activate the service weekly, and 300 seconds after boot-up.

```certbot-renewal.timer
[Unit]
Description=Timer for Certbot Renewal

[Timer]
OnBootSec=300
OnUnitActiveSec=1w

[Install]
WantedBy=multi-user.target
```

Start and enable the timer:

```sh
sudo systemctl start certbot-renewal.timer
sudo systemctl enable certbot-renewal.timer
```

To view the status information for the timer:

```sh
systemctl status certbot-renewal.timer
```

To view the journal entries for the timer

```sh
journalctl -u certbot-renewal.service
```

#### Configure fail2ban

Change fail2ban configuration to allow for checking httpd logs

```sh
sudo vim /etc/fail2ban/jail.local
```

Look for expression ``^[httpd`` and add the line ``enabled = true`` to the sections found. After all the changes are made, reload the fail2ban service.

### Setup Wireguard Server using podman

Some of the details to setup wireguard were copied from [https://www.procustodibus.com/blog/2022/10/wireguard-in-podman/](https://www.procustodibus.com/blog/2022/10/wireguard-in-podman/). However the [linuxserver.io](https://github.com/linuxserver/docker-wireguard) wireguard container is being used.

Setup modules to be loaded in ``/etc/modules-load.d/wireguard.conf``.

```/etc/modules-load.d/wireguard.conf
# /etc/modules-load.d/wireguard.conf
# WireGuard module
wireguard
# iptables/nftables modules for basic DNAT/SNAT and masquerading
nft_chain_nat
nft_compat
xt_nat
iptable_nat
xt_MASQUERADE
# nftables modules for wg-quick default route
nft_ct
nft_fib_inet
iptable_mangle
iptable_raw
xt_addrtype
xt_comment
xt_connmark
xt_mark
```

Reload modules and check the logs for any errors.

```sh
sudo systemctl restart systemd-modules-load
journalctl -u systemd-modules-load
```

Open udp port 51820 for external connections (required as run as non-root).

```sh
sudo firewall-cmd --add-port=51820/udp --permanent && sudo firewall-cmd --reload && sudo firewall-cmd --list-all 
```

Create a directory to store the configuration files.

```sh
mkdir -p ~/code/containers/wireguard/volume/config
```

Create a ``launch.sh`` script in the directory ``~/code/containers/wireguard`` with the below content.

```sh
#!/bin/env bash

podman run -d \
    --cap-add NET_ADMIN \
    --cap-add NET_RAW \
    --cap-add SYS_MODULE \
    --name wg-server \
    --publish 51820:51820/udp \
    --rm \
    --sysctl net.ipv4.conf.all.forwarding=1 \
    --sysctl net.ipv4.conf.all.src_valid_mark=1 \
    -e TZ=Etc/UTC \
    -e SERVERURL=routecvt02.netlykos.org \
    -e SERVERPORT=51820 \
    -e PEERS=10 \
    -e PEERDNS=1.0.0.1,8.8.8.8 \
    -e LOG_CONFS=true \
    --volume /home/netlykos/code/containers/wireguard/volume/config:/config:Z \
    lscr.io/linuxserver/wireguard:latest
```

To verify that the container started without any issues and to confirm after connecting, tail the logs.

```sh
podman logs -f wg-server
```

### Setup TimeMachine backup

Install samba

```sh
sudo dnf install -y samba
```

Create the `timemachine` user, ensuring it cannot login but has samba credentials

```sh
sudo adduser timemachine -MN -s /sbin/nologin -u 50000 -g nobody
sudo smbpasswd -a timemachine
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.$(date +%Y-%m-%d)
sudo mkdir -p /mount/xxxx/backups/timemachine
sudo chown timemachine:nobody /mounts/xxxx/backups/timemachine
```

Update the samba configuration file as below (hint: copy/paste and then make the necessary changes)

```smb.conf
[global]
workgroup = hostname
min protocol = SMB2

# security
security = user
passdb backend = tdbsam
map to guest = Bad User

# mac Support
spotlight = yes
vfs objects = acl_xattr catia fruit streams_xattr
fruit:aapl = yes
fruit:time machine = yes

#NetShares 

[volumes]
comment = Time Machine
path = /mounts/xxxx/backups/timemachine
valid users = timemachine
browsable = yes
writable = yes
read only = no
create mask = 0644
directory mask = 0755
```

Test the configration changes are okay.

```sh
sudo testparm
```

Start and enable the samba service, then verify they are working as expected and add the firewall rules for samba.

```sh
sudo systemctl start smb
sudo systemctl enable smb
sudo systemctl start nmb
sudo systemctl enable nmb
sudo systemctl status smb
sudo systemctl status nmb
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

### Homebridge setup

Homebridge can be setup to use podman using the following steps.

1. Create a shell script to launch the container

```sh
#!/bin/sh

docker run -d -rm \
  --net=host \
  --name=homebridge \
  -v /mounts/sda1/containers/volumes/homebridge:/homebridge \
  homebridge/homebridge:latest
```

2. Update the firewall rules to allow connections to the homebridge container from an external system.

*Note*: Both the http port to manage homebridge and the port that HomeKit needs to be connectable with homebridge need to be added to the firewall rules. The HomeKit communication port can change, so check the logs as the application is starting up for the port.

```logs
Homebridge v1.6.1 (HAP v0.11.1) (Homebridge XXXX) is running on port XXXXX.
```

Use the below to add the ports to the firewall.

```sh
sudo firewall-cmd --add-port=8581/tcp --permanent && \
sudo firewall-cmd --add-port=XXXXX/tcp --permanent && \
sudo firewall-cmd --reload && \
sudo firewall-cmd --list-all
```

3. Connect to the homebridge UI and setup the admin account.

#### Homebridge: Simplisafe setup

Connect to the homebridge UI via a browser (It might be best to use FireFox and/or Chrome), or if using Safari make sure that the Developer Console is enabled and http logs are being retained. Use the plugin [homebridge-simplisafe 3](https://github.com/homebridge-simplisafe3/homebridge-simplisafe3). You should find the redirect url in the location attribute in the http headers available via the developer console.

### WiFi hotspot for VPN connection

Use the below to turn the Raspberry PI into a WiFi router serving up VPN connection:

```sh
sudo dnf install elrepo-release epel-release
sudo dnf install -y wireguard-tools dnsmasq hostapd systemd-resolved

```

### Scrypted setup

Scrypted can be setup to use podman using the following steps.

1. Create a shell script to launch the container

```sh
#!/bin/sh

podman run \
  -d \
  --name scrypted \
  --network host \
  --restart unless-stopped \
  -v /mounts/sda1/containers/volumes/scrypted:/server/volume \
  koush/scrypted:latest
```

1. Update the firewall rules to allow connections to the scrypted container from an external system.

```sh
sudo firewall-cmd --add-port=11080/tcp --permanent && \
sudo firewall-cmd --reload && \
sudo firewall-cmd --list-all
```

1. Connect to the scrypted UI and setup the admin account.

## Acknowledgements

- [https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt](https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt)
- [https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/](https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/)
- [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9)
- [https://bobcares.com/blog/fail2ban-unban-ip/](https://bobcares.com/blog/fail2ban-unban-ip/)
- [https://www.how2shout.com/linux/how-to-disable-or-turn-off-selinux-on-rocky-linux-8/](https://www.how2shout.com/linux/how-to-disable-or-turn-off-selinux-on-rocky-linux-8/)
- [https://www.howtoforge.com/how-to-install-wireguard-vpn-on-rocky-linux-9/](https://www.howtoforge.com/how-to-install-wireguard-vpn-on-rocky-linux-9/)
- [https://irvingduran.com/2021/04/how-to-configure-macos-timemachine-and-ubuntu-20-04/](https://irvingduran.com/2021/04/how-to-configure-macos-timemachine-and-ubuntu-20-04/)
- [https://dev.to/ea2305/time-machine-backup-with-your-home-server-1lj6](https://dev.to/ea2305/time-machine-backup-with-your-home-server-1lj6)
- [https://www.tecmint.com/install-samba-rhel-rocky-linux-and-almalinux/](https://www.tecmint.com/install-samba-rhel-rocky-linux-and-almalinux/)
