---
title: "Install Code Server on Ubuntu 20.04 with Tailscale"
subtitle: "Code from anywhere, on any device"
date: 2020-09-25T11:25:26-07:00
draft: false
---

I wanted the ability to access my development environment from any device, particularly my iPad Pro. To that end, I decided to host an instance of Visual Studio Code on my home "server" (a computer running Proxmox). Because this dev environment has SSH key access to my Github account and interfaces with various other services, I very explicitly didn't want to expose it directly to the open internet.

As a result, I decided to integrate it into my Tailscale network. Tailscale is a Wireguard-based service that creates a private network for all of your devices, and it has truly changed my life recently, and I highly recommend it to everyone with more than one device that connects to the internet. Best of all, the free tier is *incredibly* generous to home users like me.

[Tailscale](https://tailscale.com/)

## Prerequisites

1. Tailscale is installed and set up on the host machine and any machine you plan to use to access the VS Code instance. I'm using the latest LTS version of Ubuntu, 20.04.

    [Setting up Tailscale on Ubuntu 20.04 LTS (focal)](https://tailscale.com/kb/1039/install-ubuntu-2004)

2. Nginx installed and configured (follow steps 1-4 for this particular setup).

    [How To Install Nginx on Ubuntu 20.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04)

3. You have a non-root user with sudo access, and you should set up key-based SSH access to the host machine. Lastly, disable the ability to authenticate with a password or log in as root. You should also only allow users to SSH into the machine from certain IPs. Set this up as strict as you like, just make sure you don't lock yourself out. 😉

## Installation

**These instructions are derived from a great article over at DigitalOcean.** Because we're installing within a Tailscale network, the process of getting a Let's Encrypt certificate is a little bit trickier.

You need to set code-server up with a real domain name that you actually own, because you'll be adding a TXT record to verify your ownership of the domain with Let's Encrypt.

### Main instructions

Following these instructions exactly, but **stop after step 2**. We'll secure the domain ourselves.

[How To Set Up the code-server Cloud IDE Platform on Ubuntu 20.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-the-code-server-cloud-ide-platform-on-ubuntu-20-04)

Note that you *can* use a domain through a public-facing DNS, but I prefer to use a DNS within my Tailscale network. I installed Pihole on a tiny VM, added it to Tailscale, and got a simple internal DNS along with a DNS-level adblocking tool. Two birds with one stone!

If you decide to use a public-facing DNS, you can still expect a reasonable degree of security, since anyone who attempts to connect from outside your Tailscale network will be denied.

To test that you set everything up successfully, visit http://yourdomain.com. Don't use HTTPS yet, we haven't configured that. You should see a login screen for VS Code, and after you log in with the password, you should see a working instance of VS Code! You will probably notice that there's an alert letting you know that some features are disabled over HTTP.

Next, we'll get HTTPS set up so you have full functionality.

### Securing your domain

Because Let's Encrypt can't communicate with machines inside our Tailscale network, we can't have a signed cert as automagically as is possible on a public-facing machine. Instead, we'll use DNS verification to prove to Let's Encrypt that we really own the domain. **This works even if you're using an internal DNS within the Tailscale network to actually resolve the domain name.**

Make sure you have access to the DNS control panel for the domain. You'll need to add a TXT record as part of the certification process.

#### Update firewall settings

First up, we want to modify the `ufw` rules to allow HTTPS traffic.

```bash
sudo ufw allow https
```

The output should look like this:

```bash
Rule added
Rule added (v6)
```

You need to reload `ufw` since we've changed some configurations.

```bash
sudo ufw reload
```

If it reloads successfully, the output will be `Firewall reloaded`.

#### Install Certbot

Next, let's get Certbot installed. First, we need to add some repositories.

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt update
```

Install Certbot and the accompanying tool for Nginx.

```bash
sudo apt install certbot python-certbot-nginx
```

#### Obtaining a Let's Encrypt signed certificate

Now for the good part– let's get that certificate! Here's what's going on in the command below.

- With `-d [yourdomain.com](http://yourdomain.com)`, we're specifying the domain we want to establish a certificate for.
- The `--manual` flag means that we'll be doing certain parts of the configuration by hand, including adding a TXT record to our DNS.
- `--preferred-challenges dns` specifies that we'd like for Certbot to give us a TXT record to add to our domain to verify our ownership, instead of using HTTP. Remember, we can't verify usng HTTP because the Let's Encrypt server has no ability to talk to our machine via its Tailscale IP address. This is a good thing, even though it does make our work a little more complex.
- `certonly` is another modifier that specifies we simply want to generate the certification, without any other work being done by Certbot. There is a way to use a DigitalOcean plugin to automatically add that TXT record, but for the sake of education, I chose to follow the full process manually.

```bash
certbot -d yourdomain.com --manual --preferred-challenges dns certonly
```

Follow the prompt instructions– You'll likely be asked for your email address, if you agree to the T&C's, if you want to be added to the EFF's mailing list, etc..

Finally, you'll see the instructions below.

```bash
Please deploy a DNS TXT record under the name
_acme-challenge.yourdomain.com with the following value:

667drNmQL3vX6bu8YZlgy0wKNBlCny8yrjF1lSaUndc

Once this is deployed,
Press ENTER to continue
```

Head over to your DNS control panel, and under the domain name you specified in the config files for code-server, create the TXT record. You can use [MXToolbox's TXT Lookup tool](https://mxtoolbox.com/TXTLookup.aspx) to confirm that the record has been successfully added. When it's been added, go back to the CLI and press Enter.

If the process is successful, you'll see a success message that lists out the paths of the signed certificate and private key. Make a note of these file paths.

#### Updating Nginx settings to use code-server over HTTPS

First, let's back up your old `code-server.conf` file.

```bash
sudo cp /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-available/yourdomain.com.bak
```

Now, with your configuration safely backed up, go ahead and open up the original configuration file.

```bash
sudo nano /etc/nginx/sites-available/yourdomain.com
```

Honestly, you can pretty much delete the entire file and just paste in the new configuration. It's probably faster than modifying the original, but either way, you should end up with this:

```bash
server {
    server_name yourdomain.com;
    location / {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection upgrade;
        proxy_set_header Accept-Encoding gzip;
    }
    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = yourdomain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    listen [::]:80;
    server_name yourdomain.com;
    return 404; # managed by Certbot
}
```

You should have replaced **five** instances of `[yourdomain.com](http://yourdomain.com)` with your own URL in the config above.

After this, reload Nginx

```bash
sudo systemctl restart nginx
```

This should be all you need! Visit [https://yourdomain.com](https://yourdomain.com) and if everything is configured correctly, you should be seeing a brand new, shiny instance of VS Code, viewable on any device in your Tailscale network!

### Changing the default user within code-server

There’s one more small detail that you’re probably going to want to tweak. By default, the systemd service that the DigitalOcean article has you establish to run code-server doesn’t specify a user, so it will probably end up running as root. Normally you might not care about this, except that VS Code will allow shell access to anyone through the web UI. If the service is running from the root account...then the VS Code terminal will be signed in as root. Obviously, whether this is protected in a Tailscale network or not, it isn’t ideal.

Luckily, one small tweak to the service configuration file can clear that right up.

Open the configuration file with 

```bash
sudo nano /lib/systemd/system/code-server.service
```

Add a line just below `[Service]` that reads `User=youruser`

```bash
[Unit]
Description=code-server
After=nginx.service

[Service]
User=youruser
Type=simple
Environment=PASSWORD=your_password
ExecStart=/usr/bin/code-server --bind-addr 127.0.0.1:8080 --user-data-dir /var/lib/code-server --auth password
Restart=always

[Install]
WantedBy=multi-user.target
```

You'll need to completely restart the code-server service. I found that the best way to do this for this particular change is to fully **stop** the service, and then **start** it again, rather than just running a restart/reload command.