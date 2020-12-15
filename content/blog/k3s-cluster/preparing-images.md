---
title: "Preparing images for each Raspberry Pi node with pi-boot-script"
subtitle: "Automating as much of the setup process as possible"
date: 2020-12-10T16:11:11-05:00
draft: true
---
Before we can get to the Kubernetes portion of this project, we have to get some OS images onto SD cards and booted on the Pis. Because I'd like to start this project on the right foot, we're going to do as little manual work as possible here, meaning that my goal is to not need to SSH into each *individual* Pi to complete any configuration steps. Partially, this is because at some point in the future, I'd like to remove K3s from the cluster and try out Hashicorp Nomad as an orchestration platform instead, so I'd like to have a very easy way to deploy fresh, pre-configured images.

{{< callout emoji="😅" title="Don't let the good be the enemy of the perfect." >}}
Eventually, I'd like to integrate HashiCorp Packer into this workflow to build this custom image in its entirety, rather than loading a first-boot script onto a vanilla image. I've been fiddling with it for awhile, and I'm still working out some kinks, so the script will have to work for now.

But mostly, I want to practice using some new industry-standard infrastructure-as-code techniques, because learning!
{{< /callout >}}

For the first step, we'll use a first-boot script loaded directly into a Raspberry Pi OS image, and any further configuration of the cluster will happen with Ansible.

## Download a fresh Raspberry Pi OS image

Visit the [Raspberry Pi Foundation's Downloads](https://www.raspberrypi.org/software/operating-systems/) page and get the latest release of **Raspberry Pi OS Lite**. We're using the Lite version because these Pis are going to run *headless—* without a keyboard, monitor, or mouse attached. Therefore, we definitely don't need the desktop functionality.

## Add the first-boot script

Here's what that script will do for us:

1. Expand the root filesystem to cover the entire SD card
2. Enable SSH and load my SSH public keys from GitHub
3. Set a hostname
4. Download and install Tailscale
5. Use a multi-use Tailscale auth key to start the Tailscale service on boot.

On a Mac, I can just double-click on the downloaded image to mount the `/boot` partition and view what's inside. We're going to add a file called `unattended` to the image, and modify an existing file called `cmdline.txt`.

First, we'll create the `unattended` file. In a text editor, copy and paste the code below.

Save the file as `unattended` *with no file extension.*

```bash
# 1. MAKING THE SYSTEM WORK. DO NOT REMOVE
mount -t tmpfs tmp /run
mkdir -p /run/systemd
mount / -o remount,rw
sed -i 's| init=.*||' /boot/cmdline.txt

# 2. THE USEFUL PART OF THE SCRIPT
raspi-config --expand-rootfs
sudo systemctl enable ssh
sudo systemctl start ssh
raspi-config nonint do_hostname CLUSTERNAME-ROLE

# 3. CLEANING UP AND REBOOTING
sync
umount /boot
mount / -o remount,ro
sync
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
sleep 5
```

Next, copy the file into the root folder of the `/boot` partition.

Finally, open up the `cmdline.txt` file that already exists in the partition. Look for a portion that begins with `init=` (it may or may not exist). Replace the section with the code below, or just add it to the file if there isn't a line that starts with `init=` yet.

```bash
init=/bin/bash -c "mount -t proc proc /proc; mount -t sysfs sys /sys; mount /boot; source /boot/unattended"
```

## Write the image onto SD cards

Now that our image is prepped, we need to write it to our four SD cards. There's one more customization step here: Before we write each SD card, we need to change the hostname (`CLUSTERNAME-ROLE`) in the `unattended` file. This ensures that we can easily identify our machines on our network and in Tailscale's admin console. Here's the easiest way to accomplish this:

1. Mount the image file you're going to write to the SD cards. On Mac, just double-click the file and look for it in your Finder sidebar.
2. Find the `unattended` file you copied into this image in the previous step. Open it directly in a text editor.
3. Change the hostname to suit your first Pi and save the file.
4. Use the Raspberry Pi Imager app to write the image to the first SD card.
5. Repeat steps 3 and 4 for each of your remaining SD cards.

For each Pi, my hostnames will be as follows:

```bash
4gb RPi (controller): pik3s-controller
8gb RPi (worker):     pik3s-worker1
8gb RPi (worker):     pik3s-worker2
8gb RPi (worker):     pik3s-worker3
```

## Boot each Pi for the first time

Once your Pi is connected to your home LAN over ethernet, insert the SD card and turn it on. If you scan your network, you should see the Pis with the hostnames you assigned in the previous step. Make a note of each local IP address. Mine ended up with these local IPs:

```bash
pik3s-controller: 192.168.0.100
pik3s-worker1:    192.168.0.101
pik3s-worker2:    192.168.0.102
pik3s-worker3:    192.168.0.103
```

If you reach this point, the first-boot script was successful, and we're ready to start installing some software!
