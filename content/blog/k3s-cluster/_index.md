---
title: "Build a Raspberry Pi Kubernetes Cluster with k3s, Ansible, and Tailscale"
subtitle: "I mean, honestly, what else was I going to do with four of these?"
date: 2020-12-10T02:45:40-07:00
draft: true
---
Strap in, folks, this is a long one. I've had a homelab in some capacity for about a year, but with the start of grad school, I had to cannibalize my "server" back into a good 'ol desktop PC. As it turns out, a MacBook Air doesn't have enough horsepower to run Zoom.

I spent a good amount of time over the past semester working through some fantastic Docker and Kubernetes courses, and with some spare time over the holiday, I'm ready to pull it all together. Enter the Raspberry Pi 4. Today, my homelab is running on a four-node K3s cluster of Raspberry Pi 4's, with 16 cores of CPU and 28gb of RAM. Here's how I built it: