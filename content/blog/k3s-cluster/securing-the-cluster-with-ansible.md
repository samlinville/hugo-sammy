---
title: "Securing your cluster nodes with Ansible"
subtitle: "Even with a Tailscale VPN, we still need to implement some basic security protocols"
date: 2020-12-10T16:13:50-05:00
draft: false
---
Now that each node in our cluster is accessible through our Tailscale network, let's implement a few additional security measures with Ansible. While Tailscale is a great first line of defense, we don't want to ignore basic server security standards. In this chapter, we'll

- Add our Tailscale IP addresses to the Ansible inventory
- Update all of the system packages
- Create an admin group
- Create a user called `ansible` and give it sudo privileges
- Copy our SSH encryption key over to the nodes
- Harden our SSH configuration
- Delete the default `pi` user

This might sound like a lot of work to replicate across four different nodes, but remember, Ansible is going to automate all of this!

## Add the Tailscale IP addresses to your Ansible inventory

{{< figure src="/media/tailscale-admin-console.png" alt="Tailscale Admin Console" caption="Tailscale Admin Console" >}}

Now that each node of our Pi cluster is in our Tailscale network, we should get in the habit of contacting them via their Tailscale IP addresses. Let's add a section to our `hosts.ini` file with the new IPs, which you can find in the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).

Overwrite the original `[pik3s_nodes_lan]` section in `hosts.ini`, replacing each line with the new static Tailscale IP address. I've censored mine with asterisks for security.

```bash
[pik3s_nodes_tailscale]
100.**.**.**
100.**.**.**
100.**.**.**
100.**.**.**
```

Just like last time, we need to add these IP addresses to `known_hosts` in order for Ansible to access them. To do this, just run the script below from within your project directory using Bash.

```bash
SERVER_LIST=$(tail -n +2 hosts.ini)
for host in $SERVER_LIST; do ssh-keyscan -H $host >> ~/.ssh/known_hosts; done
```

## Create the Ansible playbook

In the project directory, create a file called `pi-security.yml`. Copy the Ansible Playbook below— be sure to change out `GITHUBUSERNAME` for your...ya know...GitHub username.

```bash
---
- name: Pi Security
  hosts: pik3s_nodes_tailscale
  become: yes
  tasks:
    - name: Perform full patching
      package:
        name: '*'
        state: latest

    - name: Add admin group
      group:
        name: admin
        state: present

    - name: Add local user
      user:
        name: ansible
        group: admin
        shell: /bin/bash
        home: /home/ansible
        create_home: yes
        state: present

    - name: Change user password
      user:
        name: ansible
        update_password: always
        password: "{{ newpassword|password_hash('sha512') }}"

    - name: Add SSH public key for user
      authorized_key:
        user: ansible
        key: https://github.com/GITHUBUSERNAME.keys
        state: present

    - name: Add sudoer rule for local user
      copy:
        dest: /etc/sudoers.d/ansible
        src: etc/sudoers.d/ansible
        owner: root
        group: root
        mode: 0440
        validate: /usr/sbin/visudo -csf %s

    - name: Add hardened SSH config
      copy:
        dest: /etc/ssh/sshd_config
        src: etc/ssh/sshd_config
        owner: root
        group: root
        mode: 0600
      notify: Reload SSH

    - name: Reboot each node
      command: /sbin/shutdown -r now

  handlers:
      - name: Reload SSH
        service:
          name: sshd
          state: reloaded
```

Before we can run the playbook, we need to create two additional files that Ansible will copy over to each machine. In the project directory, create an `etc` directory, with two subdirectories: `ssh` and `sudoers.d`. So far, your project directory should look like this:

### Create sshd_config

In `etc/ssh/`, create a file called `sshd_config` and paste in the configuration below. This configuration file sets up stricter rules for SSH access to our nodes: mainly, that encryption keys must be used instead of passwords, and you can't SSH into the machine as the root user.

```bash
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
SyslogFacility AUTHPRIV
AuthorizedKeysFile	.ssh/authorized_keys
PasswordAuthentication no
ChallengeResponseAuthentication no
GSSAPIAuthentication yes
GSSAPICleanupCredentials no
UsePAM yes
X11Forwarding no
Banner /etc/issue.net
AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES
AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT
AcceptEnv LC_IDENTIFICATION LC_ALL LANGUAGE
AcceptEnv XMODIFIERS
Subsystem	sftp	/usr/libexec/openssh/sftp-server
PermitRootLogin no

PermitRootLogin no
```

### Create sudoers.d config

In `etc/sudoers.d/`, create a file called `ansible` and copy in the following line.

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

So far, your project directory should look like this:

```bash
Project/
	- /etc/
		- /ssh/
			- /sshd_config
		- /sudoers.d/
			- /ansible
	- /pi-tailscale.yml
	- /pi-security.yml
	- /hosts.ini
```

## Run the Ansible playbook

Cool, now we're ready to run this playbook and tighten up the security on each of our cluster nodes. To run it, navigate to your project directory in Terminal, and paste in the following command. Be sure to replace `PASSWORD` with your chosen password.

**A note on security**
Because we're entering a plaintext password into a Terminal window, it's always a good idea to prepend the command with a space. This tells Terminal not to log the command in our history, so you won't end up with your password stored anywhere in plaintext on your machine.

```bash
ansible-playbook -i hosts.ini -u pi -k pi-security.yml --extra-vars newpassword=PASSWORD
```

One last time, you'll be prompted for the password to the `pi` accounts. By default, this is just `raspberry`. But remember, once we finish this step, we won't need to use the default account or password anymore. In fact, we'll delete them!

The playbook might run for a few moments as it downloads packages across four different nodes, but the after action report should show no errors.

## Delete default users

There's one last step before we can move on to install K3s: we need to delete the default `pi` users from each node.

{{< callout emoji="🙃" title="There's gotta be a better way to do this">}}
I know there's a way to integrate this step into the main playbook, but I kept running up against errors since we technically initiated the playbook *through* the `pi` user. If you have any suggestions, I'm all ears! Until I sort this out, this tiny extra step will do the job.
{{< /callout >}}

Ansible also allows us to run commands on our inventory directly from Terminal, without using a playbook. We'll make use of that feature here. To delete the `pi` user from each of your nodes, run this command inside your project directory in Terminal.

```bash
ansible all -i hosts.ini -u ansible -m ansible.builtin.user -b -a "name=pi state=absent remove=yes"
```

That's it! The default `pi` user no longer exists on your machine. It's time to install K3s!