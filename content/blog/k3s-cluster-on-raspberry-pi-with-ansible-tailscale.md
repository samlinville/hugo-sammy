---
title: "Build a Raspberry Pi Kubernetes Cluster with k3s, Ansible, and Tailscale"
subtitle: "I mean, honestly, what else was I going to do with four of these?"
date: 2020-12-08T21:13:07-05:00
draft: true
---

Strap in, folks, this is a long one. I've had a homelab in some capacity for about a year, but with the start of grad school, I had to cannibalize my "server" back into a good 'ol desktop PC. As it turns out, a MacBook Air doesn't have enough horsepower to run Zoom.

I spent a good amount of time over the past semester working through some fantastic Docker and Kubernetes courses, and with some spare time over the holiday, I'm ready to pull it all together. Enter the Raspberry Pi 4. Today, my homelab is running on a four-node K3s cluster of Raspberry Pi 4's, with a collective 16 cores of CPU and 28gb of RAM. Here's how I built it:

## Hardware

First, I picked up four Raspberry Pi 4's. Personally, I ordered them from PiShop, but this was just on account of stock limitations elsewhere. I went with a 4GB Pi for the master node, and 8GB Pi's for the worker nodes. With the Raspberry Pi 4, it's critical that you buy a 3amp power supply if you don't want the board to get throttled by power limitations under heavier loads. After that, there were just a handful of other accessories to order.

Altogether, here's the shopping list.

[Shopping List](https://www.notion.so/0839fdd7eaf546c9a2332cca42caa7e2)

## Tools I used

- A MacBook air, but any Linux/Unix machine will work. If you're using Windows, I'm assuming you know how to access things from the Command Prompt, but you're on your own there.
- Jim Danner's excellent [pi-boot-script](https://gitlab.com/JimDanner/pi-boot-script) project, to do just enough customization on each Pi to get it on my network
- Tailscale, for creating a Wireguard-based mesh VPN with static IP addresses for each Pi
- Red Hat Ansible, for managing the group of Pi boards and installing Kubernetes
- K3s, a lightweight Kubernetes distribution that's perfect for a small hobby setup like this.

## Preparing images for the Raspberry Pis

Before we can get to the Kubernetes portion of this project, we have to get some OS images onto SD cards and booted on the Pis. Because I'd like to start this project on the right foot, we're going to do as little manual work as possible here, meaning that my goal is to not need to SSH into each *individual* Pi to complete any configuration steps. Partially, this is because at some point in the future, I'd like to remove K3s from the cluster and try out Hashicorp Nomad as an orchestration platform instead, so I'd like to have a very easy way to deploy fresh, pre-configured images.

**Don't let the good be the enemy of the perfect.**
Eventually, I'd like to integrate HashiCorp Packer into this workflow to build this custom image in its entirety, rather than loading a first-boot script onto a vanilla image. I've been fiddling with it for awhile, and I'm still working out some kinks, so the script will have to work for now.

But mostly, I want to practice using some new industry-standard infrastructure-as-code techniques, because learning! For the first step, we'll use a first-boot script loaded directly into a Raspberry Pi OS image, and any further configuration of the cluster will happen with Ansible.

### Download a fresh Raspberry Pi OS image

Visit the [Raspberry Pi Foundation's Downloads](https://www.raspberrypi.org/software/operating-systems/) page and get the latest release of **Raspberry Pi OS Lite**. We're using the Lite version because these Pis are going to run *headless—*without a keyboard, monitor, or mouse attached. Therefore, we definitely don't need the desktop functionality.

### Add the first-boot script

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

### Write the image onto SD cards

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

### Boot each Pi for the first time

Once your Pi is connected to your home LAN over ethernet, insert the SD card and turn it on. If you scan your network, you should see the Pis with the hostnames you assigned in the previous step. Make a note of each local IP address. Mine ended up with these local IPs:

```bash
pik3s-controller: 192.168.0.100
pik3s-worker1:    192.168.0.101
pik3s-worker2:    192.168.0.102
pik3s-worker3:    192.168.0.103
```

If you reach this point, the first-boot script was successful, and we're ready to start installing some software!

## Installing Tailscale with Ansible

The rest of this post will make heavy use of Red Hat Ansible to automate tasks across the four Raspberry Pis, so we don't have to SSH into each Pi individually. If you don't have Ansible on your Mac, I recommend using Homebrew to quickly install it (`brew install ansible`). If you don't have Homebrew, you should get that immediately, it'll change your life.

The first order of business is to install Tailscale on each Raspberry Pi, so each computer will have a static IP address within your secure Tailscale VPN.

**A quick note on Tailscale**
I'll go into more depth later, but for now, just know that Tailscale is a fantastic, life-changing little service that creates a Wireguard-based mesh VPN for all of your devices. It'll ensure that all of the things I run on this cluster are accessible to me, and me only. The best part? No matter what hairy scary network rules I'm dealing with (University IT, anyone?), Tailscale makes it easy to talk to devices anywhere in the world. I'm a huge fan, and at the time of writing, they just secured their Series A funding. Congrats!

I wrote a *very* rudimentary Ansible playbook to install Tailscale on each of the Pis in our cluster. In the future I'd like to make use of [artis3n's fantastic Ansible role](https://github.com/artis3n/ansible-role-tailscale) for this, but at the moment it doesn't support Raspberry Pi OS.

```yaml
---

- name: pik3s_tailscale
  hosts: pik3s_nodes_lan
  become: true

  tasks:
  - name: Install apt-transport-https
    apt:
      name: apt-transport-https
      update_cache: yes

  - name: Add GPG key with apt-key
    apt_key:
      url: https://pkgs.tailscale.com/stable/raspbian/buster.gpg
      state: present

  - name: Add list
    shell: curl -fsSL https://pkgs.tailscale.com/stable/debian/buster.list | sudo tee /etc/apt/sources.list.d/tailscale.list

  - name: Update all packages to their latest version
    apt:
      name: "*"
      state: latest

  - name: Install Tailscale
    apt:
      name: tailscale
      update_cache: yes

  - name: Add list
    shell: tailscale up --authkey=TAILSCALE_AUTHKEY
```

Create an empty directory on your Mac and save the file above as `pik3s-tailscale.yml`.

Next, we need to create an inventory of our Pi nodes, so that Ansible knows which machines to target when we start running playbooks. In the same directory, create a file called `hosts.ini` and copy in the code below. Be sure to change the IP addresses to match what your router assigned to each Pi.

```yaml
[pik3s_nodes_lan]
192.168.0.100
192.168.0.101
192.168.0.102
192.168.0.103
```

We also need to add these addresses to `.ssh/known_hosts` so that the Ansible playbook won't get hung up on that prompt to add the fingerprint for each machine when it tries to connect for the first time. In Terminal, navigate to your project directory and run the following command to add the addresses you listed in your hosts file.

```bash
SERVER_LIST=$(tail -n +2 hosts.ini)
for host in $SERVER_LIST; do ssh-keyscan -H $host >> ~/.ssh/known_hosts; done
```

This script will read your `hosts.ini` starting on line 2 (to skip the bracketed name of the inventory group), and loop through each of the addresses, running it through ssh-keyscan and adding it to `.ssh/known_hosts`.

We're finally ready to run the playbook! Still in your project directory, run the following command:

```bash
ansible-playbook -i hosts.ini -u pi -k pi-tailscale.yml
```

Here's what's happening:

- `-i hosts.ini`: This flag sets the inventory and specifies where the file is located.
- `-u pi -k`: These flags specify a username to use with SSH (`pi`) and tell Ansible to prompt us for the password (`-k`).
- `pi-tailscale.yml`: The playbook we want to execute.

If this is your first time with Ansible, you'll probably need to install `sshpass` in order to successfully log in to the Pis. The easiest way to get this is with Homebrew: `brew install hudochenkov/sshpass/sshpass`.