---
layout: post
title: "Set-up a Raspberry Pi with dynamic IP and Bittorrent Sync"
date: 2014-01-21 03:06:51
categories: raspberry-pi
post_author: "Daniel Davidson"
---

![Raspberry Pi]({{ site.url }}/images/Raspberry_Pi_-_Model_A.jpg)

The Raspberry Pi is an amazing little device. When I purchased it was to have a simple low-power 24/7 server which could do a few cron jobs to backup remote servers, pretty simple stuff. I have since added a couple of really awesome services, one is [Bittorrent Sync](http://labs.bittorrent.com/experiments/sync.html) which although (currently) closed-source is the best de-centralised file-sync system I have used when compared to Dropbox. The other is [Stringer](https://github.com/swanson/stringer), which is a really nice, simple RSS reader which includes a clone of the Fever API so you can use compatible apps on Android/iPhone etc.

Although I loved the setup I had on the Raspberry Pi I felt like things had become a bit messy during various experiments, so I decided to start over from scratch so I could do a clean installation and document the entire process for reference. This is a big guide, so I thought other people might find it interesting and use/adapt it for their own needs or point out some things I could have done better (feel free to leave comments). Here is the complete walkthrough.

### Preamble

Firstly a word of caution, if you are completely new to the command line this may not be for you. There will be times where you need at least basic *nix knowledge which is not documented below.

Secondly I will using a Mac throughout this guide, as much as love Linux I haven't quite switched over yet on the desktop. The only real differences of note are the preparation of the SDCard, everything else is done directly on the Raspberry Pi over SSH on the Raspberry Pi so the commands will be identical.

### 1. Components

This will be a [headless system](http://en.wikipedia.org/wiki/Headless_system), no monitor is required. The most expensive components here will be the USB Hard Drive(s), but hopefully you will have an old one laying around you can repurpose. I am not using WiFi for speed reasons. Here is the complete list:

1. Raspberry Pi Model B with mains power adapter
2. 8GB SD card (anything over 4GB is fine)
3. USB hard drive(s)
4. Powered USB hub[1]
5. Ethernet and USB cables

### 2. Get Raspian "wheezy" on an SD card

If you have an SD card with Raspian "wheezy" pre-installed you can skip this and go to the next step. This step is to prepare the SD card with the standard Raspian "wheezy" image.

First download the Raspian "wheezy" image from [http://www.raspberrypi.org/downloads](http://www.raspberrypi.org/downloads). Other images are available, including more minimal ones (no GUI) but I prefer to stick to the official one.

After your download is finished it's probably a good idea to check that the SHA-1 matches. On a Mac you can open a terminal and type the following, adapting the path to the location and file of the downloaded image file:

{% highlight console %}
shasum /path/to/file
{% endhighlight %}

Compare the resulting SHA-1. If it matches, unzip the file so you have uncompressed version (this will end in **.img**) and delete the zipped version.

#### Warning

What follows next are instructions adapted from [http://elinux.org/RPi_Easy_SD_Card_Setup](http://elinux.org/RPi_Easy_SD_Card_Setup). I recommend reading that page as it has much more comprehensive instructions and includes more platforms, this is Mac specific. This is the most dangerous step throughout this guide as doing this wrong will result in overwriting an incorrect disk, which guarantees losing data and facing a potential catastrophe. I am still noting the basic instructions here for completeness, but if you do not understand these commands exercise extreme caution and go through the instructions linked above.

#### dd

The next step is to **dd** the Raspian image onto our SD card, start by running the following command:

{% highlight console %}
df -h
{% endhighlight %}

Next connect the SD card and run the `df -h` command again. Now look for a new device and carefully note the name, an example would be `/dev/desk1s1`.

Now unmount the mounted SD card partition by adapting the command below so that it matches the device name noted earlier:

{% highlight console %}
sudo diskutil unmount /dev/DISK_NAME
{% endhighlight %}

We now need the establish the raw device name for the entire SD card disk. To do so omit the final "s1" from the partition name and replace "disk" with "rdisk". Using the example of **/dev/disk1s1** the raw disk would be **/dev/rdisk1**. *Again it is critical that you get this correct, if you mistakenly choose the wrong drive it will wipe out all data which would have disastrous consequences*. Once you have established the raw device name, enter the following command adapting it as needed:

{% highlight console %}
sudo dd bs=1m if=~/path/to/file.img of=/dev/RAW_DISK_NAME
{% endhighlight %}

Example (note I have omitted the sudo for safety):

{% highlight console %}
dd bs=1m if=~/Downloads/2013-07-26-wheezy-raspbian.img of=/dev/rdisk1
{% endhighlight %}

After the dd command finishes, eject the card using the following command:

{% highlight console %}
sudo diskutil eject /dev/RAW_DISK_NAME
{% endhighlight %}

You can now insert the SD card into your Raspberry Pi.

### 3. Initial setup

At this point you will have the Raspbian "wheezy" SD card inserted into your Raspberry Pi, have connected the ethernet cable (onto your local network), have plugged-in the USB hard drive routed via the powered USB hub and the USB power adapter for the Raspberry Pi which is now powered on. Let's start talking to your Raspberry Pi.

#### Getting the IP address

The first thing we need is to find out the IP address of the Raspberry Pi. Many routers include a topology of the local network inside the control panel, if you can log in and find out the IP address from their it is certainly the easiest method. Another way is a tool such as [Nmap](http://nmap.org/), a command line utility[1]. On a Mac with [Homebrew](http://brew.sh/) installed it is simply a matter of `brew install nmap` to get it installed, for other platforms just following the instructions on Nmap's website. Once installed you can issue the following command:

{% highlight console %}
nmap -v -A 192.168.1.0/24
{% endhighlight %}

This will take a few minutes and spit out a lot of information, what you want is to scroll back through the report and find something similar to this:

  Nmap scan report for 192.168.1.10
  [removed]
  22/tcp open  ssh     OpenSSH 6.0p1 Debian 4 (protocol 2.0)
  [removed]
  Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

*Note I have removed a few lines and information from the above output*

You can see it found a Debian based OS on the network and it only 1 open port, 22/SSH. This is almost certainly the Raspberry Pi so we now have the IP address of **192.168.1.10**.

#### SSH

With the IP address in hand we can SSH into our Raspberry Pi. The default username and password in Raspian "wheezy" are **pi** and **raspberry** respectively, so in the terminal issue the following command:

  ssh pi@192.168.1.10

If you need to confirm adding to known hosts do so, and enter the default password above. If everything went well you should now be sitting at a command prompt showing something like **pi@raspberrypi**. We're now logged into the Raspberry Pi.

#### raspi-config

One thing you may have noticed was the following statement shown when you logged-in **"NOTICE: the software on this Raspberry Pi has not been fully configured. Please run 'sudo raspi-config'"**. Even if this mesaage was not shown let's go through the initial configuration. Run the following command:

  sudo raspi-config

You should see something like the following, you can navigate using arrow keys and the return key to select items.

PICTURE OF RASPI-CONFIG

Inside raspi-config we need to do the following:

1. Run **Change User Password** and set a new, strong password.
2. Run **Expand Filesystem** which will maximise the SD card space.
3. Run **Enable Boot to Desktop** and select **No**, we do not want the desktop to autoload.
4. Run **Advanced Options -> Memory Split** and set it to 16 (we are setting-up a headless system so graphics are not important).

Now exit by selecting **Finish** and rebooting. Your connection will be closed so connect again with SSH using the same command earlier (note you will need to wait until the Raspberry Pi has fully rebooted).

### 4. Users, security, port forwarding and dynamic DNS.

Since this machine is going to be internet facing, we need some security measures in place. For this reason we want to start by creating some new users for accessing the machine.

#### Adding users

For the first user you can replace `raspi` with a user name of your choice:

  sudo adduser raspi

Ensure you use a long secure password and provide any information you would like (optional) and confirm. You also want to add a user for Bittorrent Sync, so repeat the process:

  sudo adduser btsync

Next we need to allow this user **sudo** privilegdes, issue the following command:

  sudo visudo

At the bottom you will see the following line:

  pi ALL=(ALL) NOPASSWD: ALL

Add identical lines underneath changing only the user names we created earlier. You should have something like the following:

  pi ALL=(ALL) NOPASSWD: ALL
  raspi ALL=(ALL) NOPASSWD: ALL
  btsync ALL=(ALL) NOPASSWD: ALL

You can hit **CTRL** + **x** and confirm the changes with **Y** to save and exit. Now we need to edit another file, issue the following command:

  sudo nano /etc/ssh/sshd_config

Find and change `PermitRootLogin` from **yes** to **no** so that it reads:

  PermitRootLogin no

Lastly add the following to the bottom of the file:

  UseDNS no
  AllowUsers pi raspi btsync

Now reload SSH

  sudo service ssh restart

*One thing you might want to consider if you even want to keep the default user 'pi' as this is a known user of the Raspberry Pi so presents a potential security threat.*

#### iptables

[iptables](http://www.netfilter.org/projects/iptables/index.html) is the userspace command line program used to configure the Linux 2.4.x and later packet filtering ruleset. I won't go into details for each of these commands but if you do want a much more thorough walkthough I recommend the guide [How to Set Up a Firewall Using IP Tables on Ubuntu 12.04](https://www.digitalocean.com/community/articles/how-to-set-up-a-firewall-using-ip-tables-on-ubuntu-12-04) by [Digital Ocean](https://www.digitalocean.com/). Here are the commands:

{% highlight console %}
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport ssh -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 5000 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8888 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables -I INPUT 1 -i lo -j ACCEPT
sudo iptables -L -v
{% endhighlight %}

If you have entered the above commands correctly you should see something very similar to this printed to the console (ignore the **pkts** and **bytes**):

{% highlight console %}
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  lo     any     anywhere             anywhere
  543 35064 ACCEPT     all  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
    0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
    0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:http
    0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:5000
    0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:8888
    3   301 DROP       all  --  any    any     anywhere             anywhere

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 94 packets, 13736 bytes)
 pkts bytes target     prot opt in     out     source               destination
{% endhighlight %}

The rules above are not yet persistent and will be lost upon the reboot of the machine, so we next need to make them persistent:

{% highlight console %}
sudo apt-get update
sudo apt-get install iptables-persistent
{% endhighlight %}

You will be asked if you want to save current IPv4 and IPv6 rules, answer yes to both. Check that the service is running by issuing this command:

  sudo service iptables-persistent start

On the next reboot these rules will remain in place.

#### Fail2ban

From [http://www.fail2ban.org/](http://www.fail2ban.org/):

***Fail2ban*** scans log files (e.g. `/var/log/apache/error_log`) and bans IPs that show the malicious signs -- too many password failures, seeking for exploits, etc. Generally Fail2Ban then used to update firewall rules to reject the IP addresses for a specified amount of time, although any arbitrary other **action** (e.g. sending an email, or ejecting CD-ROM tray) could also be configured. Out of the box Fail2Ban comes with **filters** for various services (apache, curier, ssh, etc).

The following is adapted from the much thorough guide [How to Protect SSH with fail2ban on Ubuntu 12.04](https://www.digitalocean.com/community/articles/how-to-protect-ssh-with-fail2ban-on-ubuntu-12-04), I recommend reading that if you want more information about what we are doing. Misconfiguring Fail2ban could result in being locked out of your own server, so make sure you proceed with some caution. First let's install Fail2ban:

  sudo apt-get update
  sudo apt-get install fail2ban

Now we need to make a copy of the default configuration file, you should not edit the default one directly:

  sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

Now we can edit the new configuration file:

  sudo nano /etc/fail2ban/jail.local

Feel free to change some of the values such as **bantime**, **maxretry** etc however since we don't have a mailserver and most ISP's block outgoing port 25 changing the **destemail** will have little effect. I am going to revisit this problem again in future and find a way around it probably using **SSMTP**, but for you will need to just check the logs manually. Restart Fail2ban with the following:

  sudo service fail2ban restart

If you would like to see what rules Fail2ban has in effect you can do so with the following:

  sudo iptables -L

If you ever want to check the logs you will find them in `/var/log/fail2ban.log`.

#### Port forwarding and dynamic DNS

This is optional if you only want BitTorrent Sync then you probably don't need this. Personally I like the idea of having access from outside my local network if needed, but it does mean you're increasing your attack surface. If you are worried about this then you should consider skipping this step.

The vast majority of home routers have the ability to forward ports, and most allow dynamic DNS. It is beyond the scope of this tutorial to provide instructions for all routers, but if you simply search online for **your router port forward** and **your router dynamic dns** you will almost certainly find guides. I have a BT Internet router and it was simply a question of logging into the control panel and setting up the various port forwards and entering the DnyDNS information.

#### Port forward

In terms of port forwarding we want these ports to be forwarded:

  port 22
  port 5000
  port 8888

You can map them to the same port numbers and just ensure they are assigned to the Raspberry Pi.

#### Dynamic DNS

If you are fortunate enough to have a static IP you won't need to worry about dynamic DNS, but for the vast majority of home users they will be using a non-static IP address. For dynamic DNS I use [DtDNS](http://www.dtdns.com/) but there is also [No-IP](http://www.noip.com/) and [Dyn](http://www.dyn.com/). In my router I can provide my username and password and the host url and it will then allow the port forward to map to the Raspberry Pi, for example:

  http://user.dtdns.net:8888

Will be mapped to the Bittorrent Sync control panel and:

  http://user.dtdns.net:5000

Will be mapped to the Stringer web interface and allow us to use `http://user.dtdns.net:5000/fever` as our Fever API compatible server address which we will setup shortly.

### 5. USB hard drive setup

First, a quick note. As I was compiling this guide, I decided to switch around a couple of things in the physical setup of the Raspberry Pi, one of those was the USB cable for the hard drive. This cable was bad, but it was only after about 12 hours of use before I realised it. These were strange intermittent connection issues, sometimes it would be fine for several hours, other times it would not mount at all, other times it would show up and be write-locked etc. I thought I was doing something drastically wrong with my configuration so tried many attempts to repair permissions etc, before finally realising it was a hardware issue when the following popped-up `ls: reading directory .: Input/output error`. Ugh. Anyway, the lesson here if you could just remove your palm from your face, is to always check your cables, don't be an idiot like me!

Now let's get this USB disk set up. For paranoia reasons I like to encrypt any external hard drives I use. On Mac it is built-in, on Linux there are many options but we will be using dm-crypt LUKS which seems excellent and means if someone walks off with your USB drive they can't simply mount it and browse all your files.

#### USB device ID

We need to first get the ID of the USB drive. First ensure the USB disk is unplugged, and then plug connect it and allow it to power up and immediately enter the following command:

  dmesg | tail -20

You should see something like the following in the output:

  [32297.367054] scsi 0:0:0:0: Direct-Access     Toshiba  StorE HDD        0000 PQ: 0 ANSI: 4
  [32297.371827] sd 0:0:0:0: [sda] 625142448 512-byte logical blocks: (320 GB/298 GiB)

I know that my drive is a Toshiba and that it is 320GB, so from the above output I can see a reference to it along with the device ID in brackets, `[sda]`. You can confirm this by running the following command:

  sudo fdisk -l

In the output shown it will also display the size of the disk, and again since we know the USB disk is **320GB** and also that the SD card is only **8GB** the following line re-confirms that the drive ID is indeed **sda**:

  Disk /dev/sda: 320.1 GB, 320072933376 bytes

Now make a note of your device ID, if you get this wrong in the following steps you could end up overwriting the SD card or another partition which would be a serious pain.

#### LUKS

Now we're going to employ [LUKS](https://code.google.com/p/cryptsetup/) for disk encryption. As credit, this part of the guide is adapted from [How to create an encrypted disk partition on Linux](http://xmodulo.com/2013/01/how-to-create-encrypted-disk-partition-on-linux.html), I recommend reading the full article if you would like more detailed information. We start by issuing the following commands:

  sudo apt-get update
  sudo apt-get install cryptsetup

Next we use **fdisk** to create some partitions, you will want to name and size these according to your own needs but here is an example partition I am using called **btsync** sized at 100GB, and another called **backup** also sized at 100GB. I am leaving a bit of space for future needs. Again, make sure you have the correct device ID and adapt the commands below to match:

  sudo fdisk /dev/sda

You should see the following command prompt

PICTURE OF FDISK COMMAND PROMPT

The GUI for fdisk is very unusual, you simply hit a letter and enter to get around. You can enter **m** for help. Enter **p** followed by enter to see the current partition table, you should see something like the following:

{% highlight console %}
  Disk /dev/sda: 320.1 GB, 320072933376 bytes
  255 heads, 63 sectors/track, 38913 cylinders, total 625142448 sectors
  Units = sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 512 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes
  Disk identifier: 0xa8725e51

     Device Boot      Start         End      Blocks   Id  System
  /dev/sda1            2048   251660287   125829120   83  Linux
  /dev/sda2       251660288   625142447   186741080   83  Linux
{% endhighlight %}

My disk already had a couple of partitions on it, so I want to remove those and create new ones.

**WARNING: What you are about to do will remove the data on your USB drive. You will not be able to recover from this. Ensure there is nothing you want to keep or copy it onto another computer first. There is no undo for this operation.**

These are the commands I entered to remove the existing partitions, I have noted return key as [return]:

{% highlight console %}
  Command (m for help): d [return]
  Partition number (1-4): 1 [return]

  Command (m for help): d [return]
  Partition number (1-4): 2 [return]

  Command (m for help): p [return]

  Disk /dev/sda: 320.1 GB, 320072933376 bytes
  255 heads, 63 sectors/track, 38913 cylinders, total 625142448 sectors
  Units = sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 512 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes
  Disk identifier: 0xa8725e51

     Device Boot      Start         End      Blocks   Id  System

  Command (m for help): n [return]
  Partition type:
     p   primary (0 primary, 0 extended, 4 free)
     e   extended
  Select (default p): [return]
  Using default response p
  Partition number (1-4, default 1): [return]
  Using default value 1
  First sector (2048-625142447, default 2048): [return]
  Using default value 2048
  Last sector, +sectors or +size{K,M,G} (2048-625142447, default 625142447): +100G [return]

  Command (m for help): n [return]
  Partition type:
     p   primary (1 primary, 0 extended, 3 free)
     e   extended
  Select (default p): [return]
  Using default response p
  Partition number (1-4, default 2): [return]
  Using default value 2
  First sector (209717248-625142447, default 209717248):
  Using default value 209717248
  Last sector, +sectors or +size{K,M,G} (209717248-625142447, default 625142447): +100G [return]

  Command (m for help): w [return]
{% endhighlight %}

It should now have written the changes, synced the disks and returned you back to the bash prompt. Now we want to run the cryptsetup command, first double check the device ID's

  sudo fdisk -l

You will see the 2 new partitions listed, in my case they are **sda1** and **sda2**. Now run the following command, you will need to adapt the device ID to match your own and enter a passphrase (make this very strong):

  sudo cryptsetup --verbose --verify-passphrase luksFormat /dev/sda1
  sudo cryptsetup --verbose --verify-passphrase luksFormat /dev/sda2

Once you have done the above on both partitions, the LUKS partition can be opened as follows (you will need the passphrase(s) created earlier):

  sudo cryptsetup luksOpen /dev/sda1 sda1
  sudo cryptsetup luksOpen /dev/sda2 sda2

Now we can create a filesystem on these partitions, and mount them on to the system (this may take a few minutes):

  sudo mkfs.ext4 /dev/mapper/sda1
  sudo mkfs.ext4 /dev/mapper/sda2

  sudo mkdir /mnt/btsync
  sudo mkdir /mnt/backup

  sudo mount /dev/mapper/sda1 /mnt/btsync
  sudo mount /dev/mapper/sda2 /mnt/backup

  sudo chmod 777 /mnt/btsync
  sudo chmod 777 /mnt/backup

Now we can us **df** to check the disks are mounted correctly:

  df

*Tip: if you would like to the output of **df** with megabytes, use `df -m` instead.*

#### Mount LUKS partitions automatically

There is no point having encrypted drives if we need to SSH in, enter passphrases and run these commands everytime we we reboot. Follow these instructions to have them mounted automatically. Enter the following command, again adapt it so you have the same device ID:

  sudo dd if=/dev/urandom of=/root/key.sda1 bs=1024 count=4
  sudo dd if=/dev/urandom of=/root/key.sda2 bs=1024 count=4

  sudo chmod 400 /root/key.sda1
  sudo chmod 400 /root/key.sda2

  sudo cryptsetup luksAddKey /dev/sda1 /root/key.sda1
  sudo cryptsetup luksAddKey /dev/sda2 /root/key.sda2

Now we need to get the UUID of the block devices:

  sudo cryptsetup luksUUID /dev/sda1
  sudo cryptsetup luksUUID /dev/sda2

Now edit `/etc/crypttab` and add the following line, adapted to your own device ID and UUID's:

  sda1 /dev/disk/by-uuid/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /root/key.sda1 luks
  sda2 /dev/disk/by-uuid/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /root/key.sda2 luks

Now edit `/etc/fstab` and add the mount point information:

  /dev/mapper/sda1        /mnt/btsync     auto    auto            0       0
  /dev/mapper/sda2        /mnt/backup     auto    auto            0       0

Now we need to setup a group and alter some permissions:

  # add information about adding a group, and adding users to that group
  sudo chgrp pishare -R /mnt/btsync
  sudo chmod g+rwxs -R /mnt/btsync
  sudo chgrp pishare -R /mnt/backup
  sudo chmod g+rwxs -R /mnt/backup

Now reboot, test if the partitions are auto-mounted, and see if you can write to the partition as a normal user.

### 6. BitTorrent Sync

I'll admit, so far this has been pretty boring stuff, now we can start adding more fun software like Bittorrent Sync. First let's get a few basic packages such as Git[2], htop[3] and Vim[4], run the following command:

  sudo apt-get install git htop vim rcconf

*Note: all the commands so far have used **nano** for text editing, however I much prefer Vim. It's completely optional so you can continue to use **nano** if you prefer and can remove it from the above command.*

Now we have to get Bittorrent Sync from elsewhere as it doesn't have a listing in the repositories. First we should switch to the **btsync** user and cd to that users home directory:

  su btsync

Add the following to `~/.bashrc` file

  umask 002

Now run the following commands:

{% highlight console %}
cd
mkdir btsync && cd btsync
wget http://btsync.s3-website-us-east-1.amazonaws.com/btsync_arm.tar.gz
tar -xvf btsync_arm.tar.gz
sudo mv btsync /usr/bin/
cd
mkdir .sync && cd .sync
nano config.json
{% endhighlight %}

Now add the following into the editor, adapt as needed and ensure to put a strong password into the **"password"** value:

{% highlight json %}
{
  "device_name": "raspberrypi",
  "listening_port": 0,
  "storage_path": "/home/btsync/.sync",
  "check_for_updates": true,
  "use_upnp": true,
  "download_limit": 0,
  "upload_limit": 0,
  "webui": {
    "listen": "0.0.0.0:8888",
    "login" : "admin",
    "password" : "password"
  },
  "shared_folders": []
}
{% endhighlight %}

**Note: I had some strange issues when I tried changing the login name value, I am guessing this is a bug. For now I would keep this as "admin".**

Next we need to create an **init.d** script, enter the following:

  sudo nano /etc/init.d/btsync

And paste this script in:

{% highlight bash %}
#!/bin/sh
### BEGIN INIT INFO
# Provides: btsync
# Required-Start: $local_fs $remote_fs
# Required-Stop: $local_fs $remote_fs
# Should-Start: $network
# Should-Stop: $network
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: Multi-user daemonized version of btsync.
# Description: Starts the btsync daemon for all registered users.
### END INIT INFO

# Replace with linux users you want to run BTSync clients for
BTSYNC_USERS="btsync"
DAEMON=/usr/bin/btsync

start() {
  for btsuser in $BTSYNC_USERS; do
    HOMEDIR=`getent passwd $btsuser | cut -d: -f6`
    config=$HOMEDIR/.sync/config.json
    if [ -f $config ]; then
      echo "Starting BTSync for $btsuser"
      start-stop-daemon -b -o -c $btsuser -S -u $btsuser -x $DAEMON -- --config $config
    else
      echo "Couldn't start BTSync for $btsuser (no $config found)"
    fi
  done
}

stop() {
  for btsuser in $BTSYNC_USERS; do
    dbpid=`pgrep -fu $btsuser $DAEMON`
    if [ ! -z "$dbpid" ]; then
      echo "Stopping btsync for $btsuser"
      start-stop-daemon -o -c $btsuser -K -u $btsuser -x $DAEMON
    fi
  done
}

status() {
  for btsuser in $BTSYNC_USERS; do
    dbpid=`pgrep -fu $btsuser $DAEMON`
    if [ -z "$dbpid" ]; then
      echo "btsync for USER $btsuser: not running."
    else
      echo "btsync for USER $btsuser: running (pid $dbpid)"
    fi
  done
}

case "$1" in
 start)
start
;;
stop)
stop
;;
restart|reload|force-reload)
stop
start
;;
status)
status
;;
*)
echo "Usage: /etc/init.d/btsync {start|stop|reload|force-reload|restart|status}"
exit 1
esac

exit 0
{% endhighlight %}

Save and exit. Now we need to make it executable and register it as a service, issue the following commands:

  sudo chmod +x /etc/init.d/btsync
  sudo rcconf
  sudo update-rc.d btsync defaults
  sudo shutdown -r "now"

**Note: using rcconf may be an unnecessary step, but I came across an issue where the service would not register and this seemed to help. All you need to do is ensure that the btsync service is marked.**

Your Raspberry Pi will now reboot, and all being well you should have the new btsync service start automatically at boot time. To test it navigate to **http://IP_OF_RASPBERRY_PI:8888/**, you should be prompted for your username and password and see the BitTorrent Sync GUI come up. You can now also test your dynamic DNS and port-forwarding, for example **http://USERNAME.dtdns.net:8888/**.

To get up and running with BitTorrent Sync you need to add shared folders. First click on **Add Folder**, then if you already have an existing secret you can paste it in, or you can create a new one by clicking **Generate**. You can find the hard drive under **/mnt/btsync**.
Congratulations you now have BitTorrent Sync running on your Raspberry Pi.

### 7. Stringer RSS

If you subscribe to RSS feeds then you might be interested in [Stringer](http://github.com/swanson/stringer/), an awesome, simple, un-social RSS reader and Fever compatible API system. It has a nice web ui, but probably the best thing it has is compatibility with Fever's API, so you can use an app like Press on Android. This part is adapted from the official documentation on [GitHub](https://github.com/swanson/stringer/blob/master/VPS.md), that guide is much more comprehensive so I recommend reading it for more in-depth information. Installation is pretty straight-forward, start by entering the following commands:

  sudo apt-get update
  sudo apt-get install libxml2-dev libxslt-dev libcurl4-openssl-dev libpq-dev libsqlite3-dev build-essential postgresql libreadline-dev

Now you will need to create a database for Stringer, enter the following (you'll need to create a password):

  sudo -u postgres createuser -D -A -P stringer
  sudo -u postgres createdb -O stringer stringer_live

Next we'll add a user for Stringer and login

  adduser stringer --shell /bin/bash
  su stringer

Now we're going to install Ruby using Rbenv, be prepared this is going to take a very, very long time (specifically the command `rbenv install`). When I did it, the time to complete was at least an hour.

  cd
  git clone git://github.com/sstephenson/rbenv.git .rbenv
  echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> $HOME/.bash_profile
  echo 'eval "$(rbenv init -)"' >> $HOME/.bash_profile
  git clone git://github.com/sstephenson/ruby-build.git $HOME/.rbenv/plugins/ruby-build
  source ~/.bash_profile
  rbenv install 1.9.3-p448
  rbenv local 1.9.3-p448
  rbenv rehash
  gem install bundler
  rbenv rehash

Now after that huge wait, we can install Stringer itself:

  git clone https://github.com/swanson/stringer.git
  cd stringer
  bundle install
  rbenv rehash

Next we need to set some environment variables, just ensure that you adapt the following so that it matches the password you created earlier:

  echo 'export STRINGER_DATABASE="stringer_live"' >> $HOME/.bash_profile
  echo 'export STRINGER_DATABASE_USERNAME="stringer"' >> $HOME/.bash_profile
  echo 'export STRINGER_DATABASE_PASSWORD="EDIT_ME"' >> $HOME/.bash_profile
  echo 'export RACK_ENV="production"' >> $HOME/.bash_profile
  source ~/.bash_profile
  cd $HOME/stringer
  rake db:migrate RACK_ENV=production

Now finally we can run the application itself:

  bundle exec foreman start

The only thing left we need to do is setup a cron job for Stringer, enter the following to edit your crontab:

  crontab -e

And add these lines:

  SHELL=/bin/bash
  PATH=/home/stringer/.rbenv/bin:/bin/:/usr/bin:/usr/local/bin/:/usr/local/sbin
  */10 * * * *  source $HOME/.bash_profile; cd $HOME/stringer/; bundle exec rake fetch_feeds;

Now, if you set up port-forwarding ealier correctly and are using dynamic DNS you should be able to see Stringer at either of these example locations:

  http://192.168.1.10:5000 # from inside the lan
  http://user.dtdns.net:5000 # from anywhere

### Conclusion

This is an exhaustive guide and I mainly made it for my own reference, but if you found it helpful or have any suggestions please leave them in the comments. Thanks for reading.

##### Footnotes

[1] There is also a GUI version Nmap.
[2] [Git](http://git-scm.com/) is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency.
[3] [Htop](http://htop.sourceforge.net/) is an interactive system-monitor process-viewer written for Linux. It is designed to replace the Unix program top. It shows a frequently updated list of the processes running on a computer, normally ordered by the amount of CPU usage. Unlike top, htop provides a full list of processes running, instead of the top resource-consuming processes. Htop uses color and gives visual information about processor, swap and memory status.
[4] [Vim](http://www.vim.org/) is a highly configurable text editor built to enable efficient text editing. It is an improved version of the vi editor distributed with most UNIX systems.