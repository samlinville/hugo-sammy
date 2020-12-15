---
title: "Connecting the nodes to Tailscale with Ansible"
subtitle: "Tailscale lets us securely access our cluster from anywhere in the world"
date: 2020-12-10T16:12:31-05:00
draft: true
---

The rest of this post will make heavy use of Red Hat Ansible to automate tasks across the four Raspberry Pis, so we don't have to SSH into each Pi individually. If you don't have Ansible on your Mac, I recommend using Homebrew to quickly install it (`brew install ansible`). If you don't have Homebrew, you should get that immediately, it'll change your life.

The first order of business is to install Tailscale on each Raspberry Pi, so each computer will have a static IP address within your secure Tailscale VPN.

{{< callout emoji="🔥" title="A quick note on Tailscale" >}}
I'll go into more depth later, but for now, just know that Tailscale is a fantastic, life-changing little service that creates a Wireguard-based mesh VPN for all of your devices.

It'll ensure that all of the things I run on this cluster are accessible to me, and me only. The best part? No matter what hairy scary network rules I'm dealing with (University IT, anyone?), Tailscale makes it easy to talk to devices anywhere in the world. I'm a huge fan, and at the time of writing, they just secured their Series A funding. Congrats!
{{< /callout >}}

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

Create an empty directory on your Mac and save the file above as `pik3s-tailscale.yml`. Replace `TAILSCALE_AUTHKEY` with a multi-use Tailscale Auth Key.

Be careful not to commit this version of the file to GitHub, since anyone with this Auth Key will be able to connect to your private Tailscale network. It's better to use Ansible Vault to encrypt this key, and I hope to update this article in the coming weeks with instructions for doing that.

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

{{< callout emoji="⁉️" title="Help, I'm getting an SSHpass error!">}}
If this is your first time with Ansible, you'll probably need to install `sshpass` in order to successfully log in to the Pis. The easiest way to get this is with Homebrew:

`brew install hudochenkov/sshpass/sshpass`
{{< /callout >}}