# Setup commands for raspberry pi rocky linux

This documents the commands executed after logging into a fresh rocky linux install. Download the Rocky Linux Raspberry PI (aarch64) image from [https://rockylinux.org/alternative-images/](https://rockylinux.org/alternative-images/). After flashing the image to a SD card and plugging it into the PI, the below commands can be used to configure the PI as necessary.
## Software system
```sh
sudo rootfs-expand; \
sudo dnf update -y && \
sudo dnf install -y epel-release && \
sudo /usr/bin/crb enable \
sudo dnf install -y fortune-mod mlocate net-tools bind-utils traceroute rsync podman podman-compose podman-docker xauth gvim rsync bzip2 bunzip2
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
sudo hostnamectl set-hostname xxx.xxx.xxx
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

## Host registration with ddclient

Use the below to setup ddclient as systemd service to manage host registration with [http://pairdomains.com](http://pairdomains.com). Install and configure ddclient on the local machine:

```sh
sudo dnf install -y ddclient
sudo vim /etc/ddclient.conf /usr/lib/systemd/system/ddclient.service
```

Edit the file ``/etc/ddclient.conf`` and replace the file content with the below:

```
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
 ```
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

## Optional extra's

Additional components to add to the PI that allow the Raspeberry PI to act as:
- A wireguard server
- A WiFi hotspot for a VPN connection

### Wireguard server

TBD - add details to setup a wireguard server


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

Configure certbot:
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

Change fail2ban configuration to allow for checking httpd logs
```sh
sudo vim /etc/fail2ban/jail.local
```

Look for expression ``^[httpd`` and add the line ``enabled = true`` to the sections found. After all the changes are made, reload the fail2ban service.

```sh
sudo 
```

### WiFi hotspot for VPN connection

Use the below to turn the Raspberry PI into a WiFi router serving up VPN connection:

```sh
sudo dnf install elrepo-release epel-release
sudo dnf install -y wireguard-tools dnsmasq hostapd systemd-resolved

```


###

## Acknowledgements

- [https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt](https://dl.rockylinux.org/pub/sig/8/altarch/aarch64/images/README.txt)
- [https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/](https://alldrops.info/posts/linux-drops/2022-05-11_install-rocky-linux-on-raspberry-pi-for-remote-access/)
- [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-rocky-linux-9)
- [https://bobcares.com/blog/fail2ban-unban-ip/](https://bobcares.com/blog/fail2ban-unban-ip/)
- [https://www.how2shout.com/linux/how-to-disable-or-turn-off-selinux-on-rocky-linux-8/](https://www.how2shout.com/linux/how-to-disable-or-turn-off-selinux-on-rocky-linux-8/)
