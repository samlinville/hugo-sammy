---
title: "Backups for the Nerdy and Paranoid"
subtitle: "One is none."
date: 2020-09-25T11:30:34-07:00
draft: false
---

In 2017, I had to reinstall macOS on my MacBook Pro. Before taking the plunge, I was careful to back up all of my files, *or so I thought.* In reality, I neglected the only folder on my computer that actually mattered: An archive of all of my work from design school up that point, carefully sorted by course and semester. I scoured all of my external hard drives and Google Drive folders, and in the end, the only trace of these projects were a few slide decks and thumbnail images from my portfolio site.

Losing that archive was like losing a huge piece of my history. There were probably thousands of hours worth of work represented in those files, and I called my parents and sobbed while I told them what happened.

The most frustrating part was that this situation was entirely avoidable, had I simply had a more robust backup system. For the rest of my time in school, I meticulously backed up important files on multiple physical drives and cloud accounts. I think I have seven different copies of my photos from the semester I spent in Prague.

As comforting as it has been to have those important files properly preserved, it's also a lot of work to do manually! When I graduated and started my first full-time job, all of my work files were automatically backed up with a company-wide backup tool—I didn't have to think about it all the time, I could just get my work done knowing that background processes were keeping everything safe. Over those three years, I really took that managed service for granted!

Then, a few months ago, I left my job to go back to school, and realized I would need a new background backup service. Not wanting to be locked into a managed service with monthly or annual fees, I decided to strike out on my own.

## Here's how I did it

My backup system follows the 3-2-1 principle, which states that for any "safe" backup system, you'll have three copies of the data, two of which are local but on different media and a third copy at an offsite location. I'll go through each of these components in detail, but at a high level, my system is made up of my actual computers (a MacBook Air and a Windows 10 desktop) a Synology NAS, and a Raspberry Pi 4 with a MyBook external hard drive.

### The live copy

My first copies of the data are always on my daily drivers. I use my MacBook Air and a Windows 10 PC that I built for heavier-duty work (and Zoom, because Zoom requires a truly absurd amount of CPU).

On both devices, I use Duplicati, an open-source backup tool to package up all of my files, encrypt them, and move them to the Synology NAS. This is a fairly low-latency operation, since the Synology lives within my LAN in California. I run this process every day on both devices.

At any given time, Duplicati keeps multiple historical copies of my data from the MacBook Air and PC:

- One copy from each of the last 7 days
- One copy from each of the last four weeks
- One copy from each of the last twelve months

This method seems to work from me—theoretically, there's an extremely low probability that one of these historical backups won't have the files I'm looking for in the event of a catastrophe.

In the event that one of my daily drivers kicks the bucket, Duplicati will also handle pulling data backwards from the Synology onto a new machine.

So far, I can't say enough good things about Duplicati. I highly recommend you give it a try, it has worked flawlessly so far and is extremely flexible. That said, I don't recommend it for users who aren't familiar with intermediate concepts of computer networking.

### The local backup

All of the data from my daily drivers flows to a Synology DS214 NAS with two Seagate IronWolf 4TB drives. The Synology box is pretty old—I got it secondhand on eBay, because data transfer speeds really aren't important enough to me to justify paying for a more recent model. Synology also tends to be fairly overpriced as a brand, though they have a good reputation. I set all of my backup tasks to run overnight, so it can go slowly and I'll never know the difference.

In hindsight, I wish I'd spent even less money. Really, all NAS devices are overpriced for what they are: weak computers with swappable drive bays. I think the premium for Synology is centered around DSM, their custom OS with a web-based management app. Truly, DSM sucks. It has some nice built-in features that made some of my setup easier, but it's also extremely vulnerable to vendor lock-in since it isn't based on a Linux distro. Most of all, I hate the web app. It's designed to look like an OS desktop, like you'd see in Windows or Linux—truly, one of the most awkward user experiences I've ever encountered. 

If I could do it over again, I'd buy a NUC or micro-ITX board from the early 2010s and install one of the many excellent open source Linux-based NAS management tools. Doing this would have been a great learning experience, and it probably would have been slightly less costly. Honestly, I probably could've ended up with better specs, too.

Anyways, Duplicati (running on my daily drivers) sends all the backup data to this NAS over my LAN every day. The Synology box doesn't do any intelligent work here—it just stores what Duplicati sends over. All of the historical backups and encryption are handled on the daily drivers by Duplicati.

Then, every night at 3:00am, the Synology syncs its files with my offsite backup through rsync. This is also a fairly unintelligent process—it just copies all of the new data it received over to the offsite drive. There's no encryption at this stage, since all of that data was encrypted on the daily drivers by Duplicati.

### The offsite backup

This brings me to my favorite piece of the system: my offsite drive. I bought a Raspberry Pi 4, a Western Digital MyBook external hard drive, and mailed them to my grandfather to plug into his router. These days, I'm on the West Coast and they're all the way across the country, so it feels far enough to avert any data loss caused by a natural disaster.

I'm running Ubuntu 20.04 LTS on the Raspberry Pi because I'm far more familiar with Ubuntu than raspberryOS, and my requirements for this computer are extremely simple. It simply receives files from the Synology NAS, and places them in the external hard drive. There's no redundancy here yet, though I'm considering adding another external hard drive to have a very makeshift RAID1 configuration there.

### Moving data

The hardest part of a multi-site backup solution is securely moving data over the internet. When I set out on this project, I was mostly concerned about exposing my Synology NAS or Raspberry Pi up to the open web.

Then, I found Tailscale. Tailscale is a mesh VPN service built on the Wireguard protocol, and it couldn't be any easier to get running. Authentication happens through my MFA-secured Google account, and each device within the VPN gets an internal static IP.

Tailscale is also great at recognizing when my MacBook Air is on my LAN, and when it's sitting in a coffee shop or airport. I can connect to all of my devices either way, but if I'm on the LAN, Tailscale will still route all the traffic within the LAN without transmitting out to their servers. This is a huge advantage to a VPN with a mesh configuration.

Even with Tailscale, I still take some other security precautions. Connections are still made over SSH, and each device only accepts SSH connections from explicitly-defined IP addresses (the static IPs in my Tailscale network). I also disabled password authentication and root login for SSH on each of these machines. This might be overkill, I don't know. At best, it saves my ass one day. At worst, I spent an extra 45 minutes configuring each device.

Lastly, my logic for moving data around is predicated on the idea that I want as much bandwidth for myself during the day. I don't care what happens while I'm asleep, but don't slow me down when I'm watching Netflix! For that reason, my Synology box waits until the middle of the night to start moving files over the internet to my off-site backup.

## So, how should you do it?

You should backup your files in whatever way you're most likely to maintain. A backup system is no good if it's not actually working! If that means buying a yearly subscription to Backblaze, go for it! There's nothing wrong with managed services, it just wasn't how I wanted to run my system.

That said, if you're considering building your own on-premise backup solution for personal use, here's what I recommend.

1. Spend as little money as possible. If you have the technical expertise, build your own NAS instead of buying one. At the very least, buy used NAS hardware.
2. Hard drives fail frequently. Buy them new, make redundant systems, and resign yourself to the knowledge that someday, one of your drives will crap out. I chose to buy drives specifically designed for NAS applications. Apparently, there's some extra hardware and firmware in the drive that's designed to increase its longevity in an "always-on" NAS solution. I don't know if this is really worth the money, nobody on the internet can come to an agreement.
3. If you're moving data to an offsite or cloud solution, encrypt it first! If you're building your own offsite machine too, make sure that your data transmission is also secure. I recommend making this process extremely simple: use Tailscale.
4. Set a reminder on your phone to check in once a month to ensure that the backup software that's moving your data around is still working. Good tools will send you an alert if something fails, but don't assume anything!