# Torified WiFi Hotspot

## Introduction
In this guide you will get the **secret knowledge** on how to create an IPv4/IPv6 enabled TORified Raspberry Pi WiFi hotspot. These instructions are also compatible with any other Ubuntu/Debian system as long as there is a WLAN interface as well as another network interface such as `eth0` for connecting to the Internet.

## Preparations
In order to create a Raspberry Pi hotspot from scratch, we first need an image file that contains a bootable operating system. I decided to use a current Debian system instead of the standard Raspbian because Debian provides more software packages and they are also more up-to-date. Debian images files for the Raspberry Pi can be downloaded from [https://raspi.debian.net/tested-images/](https://raspi.debian.net/tested-images/). It is best to create a new folder and then carry out the further steps in it. Open a new terminal windows and and create a new directory as follows:

```shell
mkdir ~/Downloads/torified-wifi-hotspot
```

Then change into this new directory:

```shell
cd ~/Downloads/torified-wifi-hotspot
```

Download the image file:

```shell
wget https://raspi.debian.net/tested/20230612_raspi_3_bookworm.img.xz
```

Now, write the image to your SD card! Replace `/dev/{YOUR_DEVICE}` in the following command with your actual device path. You can run the command `lsblk` in your terminal to see a list of block devices, or you can run the command `gnome-disks` to open the GUI application for managing block devices. The device path is usually something like `/dev/sdd`. **Make sure you use the correct device path, otherwise some of your data may be destroyed.** If you are 100% sure that you have selected the correct device path, run this command:

```shell
xzcat 20230612_raspi_3_bookworm.xz | sudo dd of=/dev/{YOUR_DEVICE} bs=64k oflag=dsync status=progress
```

After the image file has been written to the SD card, run this command to reread the partition table:

```shell
partprobe --summary /dev/{YOUR_DEVICE}
```

To verify that everything worked, query the list of block devices again:

```shell
lsblk
```

You should see something like this:

```text
NAME              MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
...
sdd                 8:48   1  14,4G  0 disk  
├─sdd1              8:49   1   508M  0 part  
└─sdd2              8:50   1   1.9G  0 part  
...
```

Now we have to make sure that we can log in as root. To do this, we mount the first partition which contains the firmware and configuration files:

```shell
mkdir firmware && sudo mount /dev/sdd1 firmware/
```

There is a configuration file in which we can store a root password and/or an ssh public key. I recommend using a public key because a password-based login would require further changes to the SSH configuration, and a root password is also considered insecure these days:

```shell
nano firmware/sysconf.txt
```

You can find the public key in your home diretory in the file `~/.ssh/id_rsa.pub`, just copy and paste it. You can also change the hostname to `debian-rpi64`:

```text
# root_authorized_key - Set an authorized key for a root ssh login
root_authorized_key=ssh-rsa YOUR_KEY

# hostname - Set the system hostname.
hostname=debian-rpi64
```

Press `F3` to save the file and then `F2` to quit nano.

Finally unmount the partition and remove the firmware directory:

```shell
sudo umount firmware/ && rmdir firmware/
```

Now insert the SD card into your Raspberry Pi and start it! If you don't know which ip address to use for the SSH login run `ip -4 neigh show` or `arp -a`. My Pi has the ip address `192.168.2.121`, so I use this command for the login:

```shell
ssh root@192.168.2.121
```
## Basic settings

Before setting up the hotspot, you should deactivate the root password, set the locale, time zone and hostname of the system, pimp up the terminal with colors and carry out a software upgrade.

First, we deactivate the root password by editing the file in which the hashed passwords are stored:

```shell
nano /etc/shadow
```

Now we simply add an asterisk between the first and second colon for the user root so that the file then looks something like this:

```text
root:*:19521:0:99999:7:::
```

Press `F3` to save the file and then `F2` to quit nano.

Make sure that the packages `locales` and `tzdata` are installed before setting the locale and time zone of the system:

```shell
apt install locales tzdata
```

Then run this command to change the locale through a dialogue:

```shell
dpkg-reconfigure locales
```

Then run this command to change the time zone through a dialogue:

```shell
dpkg-reconfigure tzdata
```

Now run this command to change the hostname:

```shell
hostnamectl set-hostname debian-rpi64
```

Then run this command to edit the hosts file:

```shell
nano /etc/hosts
```

Insert this line after the first line:

```shell
127.0.1.1       debian-rpi64
```

Press `F3` to save the file and then `F2` to quit nano.

To enable colors we have to edit the file in which the corresponding settings are specified:

```shell
nano /root/.bashrc
```

Simply append the following lines to the end of the file:

```text
PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u\[\033[00m\]\[\033[01;31m\]@\[\033[00m\]\[\033[01;34m\]\h\[\033[00m\]\[\033[01;31m\]:\[\033[00m\]\[\033[01;33m\]\w\[\033[00m\]# '
export LS_OPTIONS='--color=auto'
eval "$(dircolors)"
alias ls='ls $LS_OPTIONS'
alias grep='grep --color=auto'
```

Press `F3` to save the file and then `F2` to quit nano.

Then run this command to apply the changes without re-login:

```shell
source /root/.bashrc
```

Your terminal should now look *cool* :)

Now update the software packages using this command:

```shell
apt-get update && apt-get upgrade
```

Now make sure that the package `systemd-resolved` is installed:

```shell
apt install systemd-resolved
```

Also make sure that the `systemd-resolved` service is enabled and started.

```shell
systemctl enable systemd-resolved && systemctl start systemd-resolved
```

Finally reboot using this command:

```shell
reboot
```

Your Debian Raspberry Pi is now up to date!

## Setting up the hotspot

The first two sections of this guide can be skipped if you already have an Ubuntu or Debian system set up. The actual setup of the hotspot begins here. The good thing is that all you need to set up a hotspot are just five configuration files.

First we create the hotspot configuration file for wpa_supplicant as follows (change `country`, `ssid` and `psk` as per your preference):

```shell
cat << HOTSPOT >/etc/wpa_supplicant/wpa_supplicant-wlan0.conf
#ctrl_interface=/var/run/wpa_supplicant.wlan0
#ap_isolate=1
country=DE

network={
  ssid="easylink"
  mode=2
  key_mgmt=WPA-PSK
  proto=RSN
  pairwise=CCMP
  psk="12345678"
}
HOTSPOT
```

Now we can enable the hotspot service using this command:

```shell
systemctl enable wpa_supplicant@wlan0.service
```

And we can then start the hotspot service using this command:

```shell
systemctl start wpa_supplicant@wlan0.service
```

Now we have to create an IPv4-only and an IPv4+IPv6 configuration file for the network interfaces `eth0` and `wlan0`. Whether the IPv4-only configuration or the IPv4+IPv6 configuration is selected will later depend on whether the `ipv6.disable=1` parameter has been specified in the kernel cmdline or not. This will be discussed in more detail at the end of this guide.

Create the IPv4-only configuration file for eth0:

```shell
cat << NETWORK >/etc/systemd/network/00-eth0-ip4.network
[Match]
Name=eth0
KernelCommandLine=ipv6.disable=1

[Network]
DHCP=yes                  # if you prefer a static IPv4 instead of DHCP comment out this line
#Address=192.168.2.121/24 # if you prefer a static IPv4 instead of DHCP uncomment this line
#Gateway=192.168.2.1      # if you prefer a static IPv4 instead of DHCP uncomment this line
DNS=208.67.222.222
DNS=208.67.220.220
IPMasquerade=ipv4
IPForward=yes

[DHCPv4]
UseDNS=no
UseNTP=no
NETWORK
```

Create the IPv4+IPv6 configuration file for eth0:

```shell
cat << NETWORK >/etc/systemd/network/00-eth0-ip6.network
[Match]
Name=eth0
KernelCommandLine=!ipv6.disable=1

[Network]
DHCP=yes                  # if you prefer a static IPv4 instead of DHCP comment out this line
#Address=192.168.2.121/24 # if you prefer a static IPv4 instead of DHCP uncomment this line
#Gateway=192.168.2.1      # if you prefer a static IPv4 instead of DHCP uncomment this line
DNS=208.67.222.222
DNS=208.67.220.220
Address=fd00:41::1/64     # It is necessary to set a static IPv6 because TOR cannot listen on link-local addresses and there is no guarantee that we will get an IPv6 via DHCPv6 or IPv6 auto-configuration.
#Gateway=
DNS=2620:119:35::35
DNS=2620:119:53::53
IPMasquerade=both
IPForward=yes

[DHCPv4]
UseDNS=no
UseNTP=no
NETWORK
```

Create the IPv4-only configuration file for wlan0:

```shell
cat << NETWORK >/etc/systemd/network/00-wlan0-ip4.network
[Match]
Name=wlan0
KernelCommandLine=ipv6.disable=1

[Network]
DHCP=no
DHCPServer=yes
IPMasquerade=ipv4
IPForward=yes
Address=192.168.42.1/24 # The IPv4 address of our hotspot
DNS=208.67.222.222
DNS=208.67.220.220

[DHCPServer]
EmitDNS=yes
DNS=208.67.222.222 # OpenDNS server #1
DNS=208.67.220.220 # OpenDNS server #2

# use one or more of this section to assign static addresses via DHCPv4
#[DHCPServerStaticLease]
#MACAddress=xx:xx:xx:xx:xx:xx
#Address=192.168.42.100
NETWORK
```

Create the IPv4+IPv6 configuration file for wlan0:

```shell
cat << NETWORK >/etc/systemd/network/00-wlan0-ip6.network
[Match]
Name=wlan0
KernelCommandLine=!ipv6.disable=1

[Network]
DHCP=no
DHCPServer=yes
IPMasquerade=both
IPForward=yes
LinkLocalAddressing=ipv6
IPv6AcceptRA=no
IPv6SendRA=yes
#DHCPPrefixDelegation=yes
IPv6PrivacyExtensions=no
IPv6DuplicateAddressDetection=1
Address=192.168.42.1/24 # The IPv4 address of our hotspot
DNS=208.67.222.222
DNS=208.67.220.220
Address=fd00:42::1/64   # The IPv6 address of our hotspot
DNS=2620:119:35::35
DNS=2620:119:53::53

[DHCPServer]
EmitDNS=yes
DNS=208.67.222.222 # OpenDNS server #1
DNS=208.67.220.220 # OpenDNS server #2

[IPv6SendRA]
EmitDNS=yes
DNS=2620:119:35::35 # OpenDNS server #1
DNS=2620:119:53::53 # OpenDNS server #2

#[DHCPPrefixDelegation]
#UplinkInterface=:self
#SubnetId=1
#Announce=yes

#[IPv6Prefix]
#Prefix=fd00:42::/64
#ValidLifetimeSec=7200
#PreferredLifetimeSec=3600

# use one or more of this section to assign static addresses via DHCPv4
#[DHCPServerStaticLease]
#MACAddress=xx:xx:xx:xx:xx:xx
#Address=192.168.42.100
NETWORK
```

Make sure that your router supports DHCP or change the `eth0` configuration so that a static IPv4 configuration is used (see the comment in the file), otherwise you may loose the connection after the next command, and then you will have to change or remove the files on another computer. Restart `systemd-networkd` using this command:

```shell
systemctl restart systemd-networkd.service
```

Since we now use systemd-networkd, we no longer need the standard configuration. To avoid conflicts, we simply uninstall the corresponding software packages with this command:

```shell
apt purge ifupdown isc-dhcp-client isc-dhcp-common
```

Also run this command to delete old configuration files:

```shell
rm -fr /etc/network/interfaces.d/
```

The hotspot is now ready and should work. Try to connect using your smartphone or another computer.

Another reboot is also recommended to check whether everything still works after a restart:

```shell
reboot
```

## TORification

Having an IPv6-enabled hotspot is all well and good, but our goal is to route all traffic over the TOR network. This is pretty easy to do with the [`tor-router`](https://github.com/m4dm4x1337/tor-router/) Debian package. Together with the [`tor-router-web-gui`](https://github.com/m4dm4x1337/tor-router-web-gui/) Debian package, it is child's play to turn system-wide TOR routing on and off.

First download the two Debian packages onto your local computer. Then open a new terminal, change into the download diretory and run the following command to copy the Debian packages to the Raspberry Pi:

```shell
scp tor-router*_1.0_all.deb root@192.168.2.121:/tmp/
```

Then run the following command to install `tor-router` and `tor-router-web-gui` as well as the soft dependencies `dnsutils` and `geoip-bin`:

```shell
apt install dnsutils geoip-bin /tmp/tor-router_1.0_all.deb /tmp/tor-router-web-gui_1.0_all.deb
```

We can stop and disable the `tor.service` since it is not needed by tor-router:

```shell
systemctl stop tor.service && systemctl disable tor.service && systemctl mask tor.service
```

Next we need to make a few changes in the tor-router configuration file to enable router functionality and IPv6 support:

```shell
nano /etc/default/tor-router
```

By default, all configuration parameters are commented out, we need to set the following values to enable TOR-routing:

```text
ENABLE_IPV6=@AUTO
PREFER_IPV6=yes
ROUTER=@AUTO
VERBOSE=yes
```

Press `F3` to save the file and then `F2` to quit nano.

Next we need to make a change in the tor-router-web-gui configuration file to enable access from other computers in our LAN:

```shell
nano /etc/default/tor-router-web-gui
```

By default, all configuration parameters are commented out, we need to set the following values to enable password protected access from other computers in our LAN:

```text
ADDRESS=0.0.0.0
USERNAME=admin
PASSWORD=123456
```

Press `F3` to save the file and then `F2` to quit nano.

You can start the services manually and then check whether everything works:

```shell
systemctl start tor-router.service tor-router-web-gui.service
```

Then run this command to monitor the status of the `tor-router.service`:

```shell
watch -c -n1 SYSTEMD_COLORS=1 systemctl status tor-router.service --full --lines=10 --no-pager
```

Wait until the line **Bootstrapped 100%: Done** appears. After that everything should work. All your traffic will now be sent over the TOR network. No application will be able to reveal your IP address by connecting directly.

To check whether your IP address has actually changed, you can run the following command:

```shell
/usr/share/tor-router/sbin/whatsmyip
```

If you have IPv6 support enabled you can also use this command:

```shell
/usr/share/tor-router/sbin/whatsmyip6
```

You can also run this command to check the tor-router logs:

```shell
journalctl -aelq -u tor-router -n 1000 --no-pager _SYSTEMD_INVOCATION_ID="$(systemctl show -p InvocationID --value tor-router)"
```

And you can run this command to check the tor-router-web-gui logs:

```shell
journalctl -aelq -u tor-router-web-gui -n 1000 --no-pager _SYSTEMD_INVOCATION_ID="$(systemctl show -p InvocationID --value tor-router-web-gui)"
```

The tor-router service and the tor-router-web-gui service are enabled by default, which means that the TORification should work after a reboot. Let's test it:

```shell
reboot
```

## Fine tuning

It is advisable to install a few additional software packages that may be needed. I recommend:

* `bash-completion` for auto-completion in the terminal
* `cryptsetup` and `lvm2` if you want to connect an external encrypted hard drive
* `curl` and `wget` to download files
* `firmware-misc-nonfree` and `usbutils` if you want to connect additional hardware such as a WiFi USB stick
* `htop` to monitor the processes, CPU usage and RAM usage
* `transmission-cli` and `transmission-daemon` for bittorrent file sharing
* `zram-tools` to double your RAM
* `overlay-root` to protect the root partition from any changes and to extend the lifetime of your SD card.

We configure the future kernel cmdline before we install this packages since the installation will trigger `update-initramfs` which will cause that `/boot/firmware/cmdline.txt` gets overwritten. The persistent kernel cmdline is stored in `/etc/default/raspi-extra-cmdline` and any changes to `/boot/firmware/cmdline.txt` should be considered temporary since, as stated, this file can be overwritten.

Run this command to store the persistent kernel cmdline in `/etc/default/raspi-extra-cmdline`:

```shell
echo "elevator=deadline fsck.mode=skip loop.max_part=63 ipv6.disable=0 disable-root-ro= _systemd.mask=tor-router.service _systemd.mask=tor-router-web-gui.service _systemd.mask=wpa_supplicant@wlan0.service
" | tee /etc/default/raspi-extra-cmdline
```

* `elevator=deadline` extends the lifetime of your SD card
* `fsck.mode=skip` skips the filesystem check at boot (it's not needed because `overlay-root` makes the root filesystem read-only and all changes are actually written to a RAM disk)
* `loop.max_part=63` causes that `losetup` reads the partition table automatically (only important if you work with image files)
* `ipv6.disable=0` can be changed to `ipv6.disable=1` to turn the IPv4+IPv6 hotspot into an IPv4-only hotspot
* `disable-root-ro=` can be changed to `disable-root-ro=true` if you want to disable the `overlay-root` feature
* The parameters beginning with an underscore `_` can be used to disable the hotspot and/or the tor-router service by removing the underscore.

The `overlay-root` package is not an official Debian package and must be [downloaded  separately](https://github.com/m4dm4x1337/overlay-root/).

You can then copy it to the Raspberry Pi using `scp`:

```shell
scp overlay-root_3.0.0_all.deb root@192.168.2.121:/tmp/
```

Now you can install all at once:

```shell
apt install \
  bash-completion \
  cryptsetup \
  curl \
  firmware-misc-nonfree \
  htop \
  lvm2 \
  transmission-cli \
  transmission-daemon \
  usbutils \
  wget \
  zram-tools \
  /tmp/overlay-root_3.0.0_all.deb
```

It may take some time since `update-initramfs` will be triggered.

When the installation is complete `/boot/firmware/cmdline.txt` should contain the parameters that have been written to `/etc/default/raspi-extra-cmdline`.

Now open the zram configuration file with an editor:

```shell
nano /etc/default/zramswap
```

Uncomment this line:

```text
PERCENT=50
```

Press `F3` to save the file and then `F2` to quit nano.

Now we deactivate and mask services that are not absolutely needed:

```shell
systemctl disable \
  apt-daily.service \
  apt-daily.timer \
  apt-daily-upgrade.service \
  apt-daily-upgrade.timer \
  blk-availability.service \
  dm-event.socket \
  e2scrub_all.timer \
  e2scrub_reap.service \
  fstrim.timer \
  lvm2-monitor.service \
  lvm2-lvmpolld.socket \
  transmission-daemon.service
```

```shell
systemctl mask \
  apt-daily.service \
  apt-daily.timer \
  apt-daily-upgrade.service \
  apt-daily-upgrade.timer \
  blk-availability.service \
  dm-event.socket \
  e2scrub_all.timer \
  e2scrub_reap.service \
  fstrim.timer \
  lvm2-monitor.service \
  lvm2-lvmpolld.socket \
  transmission-daemon.service
```

Now we delete the journald logs and restart journald:

```shell
rm -fr /var/log/journal/ && systemctl restart systemd-journald.service
```

This will switch journald from the persistent logging mode to the temporary mode. The logs will now be stored in `/run/log/journal/`. Since we work with a RAM overlay, that's absolutely okay.

Finally we delete some cache files:

```shell
rm -fr \
  /root/.config/ \
  /root/.lesshst \
  /root/.local/ \
  /var/cache/apt/pkgcache.bin \
  /var/cache/apt/srcpkgcache.bin \
  /var/lib/apt/lists/ \
  /var/log/apt/eipp.log.xz
```

And we truncate some log files:

```shell
truncate -s 0 \
  /root/.bash_history \
  /var/log/alternatives.log \
  /var/log/auth.log \
  /var/log/btmp \
  /var/log/cron.log \
  /var/log/dpkg.log \
  /var/log/faillog \
  /var/log/kern.log \
  /var/log/lastlog \
  /var/log/syslog \
  /var/log/user.log \
  /var/log/wtmp \
  /var/log/apt/history.log \
  /var/log/apt/term.log
```

Deleting or truncating these files is absolutely okay because, as already mentioned, we work with a RAM overlay.

You may need to logout from SSH and login again and truncate `/root/.bash_history` again because only then the file will be almost empty, because the commands of the current session are only written when you leave the session.

Now reboot one more time to activate the overlay-root feature:

```shell
reboot
```

Enjoy your IPv6 capable TORified Raspberry Pi WiFi Hotspot!


## Contributing

Contributions are welcome! Feel free to fork this repository and submit pull requests.

## License

This project is licensed under the GPLv3 License. See the COPYING file for details.

## Author

This tutorial was written by m4dm4x1337.

## Donations 🥺

 ❤️ Please donate ❤️

![QR code for donations](https://raw.githubusercontent.com/m4dm4x1337/tor-router-gnome/master/tor-router-gnome/usr/share/pixmaps/tor-router-gnome-donation.png)

This project is open source, and the only income comes from the donators. If you like the project, please donate, thank you!

[bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx](bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx)
