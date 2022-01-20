---
title: "Setup Matrix Synapse Without Port Forwarding"
date: 2022-01-09T14:04:09+05:30
draft: true
---

Have you ever wanted to host your own matrix server but then realized you couldnt port forward because you are on a double nat or your isp blocks it.

TLDR: What is matrix?
Matrix is a federated, decentralized messaging platform.

One of the biggest features of matrix is bridging which allows you to recieve and send messages to other platforms. Because matrix integration is built in to libera.chat, you can join any libera.chat irc channel by joining #chan:libera.chat and pm any libera user from matrix

It has end-to-end encryption for personal use and also has non-encrypted rooms.

Many big projects have official matrix rooms (firefox, gnome, arch, manjaro, codeberg) and there are many unofficial ones with lots of members.

Yes, you can host it on a vps but they are expensive and for something that needs as much resources as synapse when joining matrix HQ, you are looking at absolutely humongous bills.

This tutorial is using Rocky Linux 8 but this should be the same on most RHEL based distros.

Firstly, the things you need:
- A computer or a VM/LXC with atleast 32 gigs of storage (trust me it becomes big quick), a minimum of 2 gigs of ram (its usually on 1.3 gigs but it spikes a lot) and 1 core 
- A cloudflare account and a domain name (subdomains wont do, you need a real domain. Freenom works well for this but if you want this for serious, use namecheap)
- A basic knowlege of the UNIX Command Line

Things we will setup:
- Cloudflare Argo Tunnel
- Synapse along with nginx reverse proxy for it
- Riot (Element)
- Whatsapp bridge

Pros:
- Helps matrix become more decentralized (Most users use matrix.org or kde.org (which is hosted by matrix lmao))
- You get faster connections as its closer to you. (matrix.org servers are located in England and The Netherlands)
- You own your data and can host whatever the heck you want (But it passes through cloudflare, remember that)
- Doesnt Leak Your IP

Cons:
- All traffic is passed through cloudflare
- Because of the previous point, you cant upload big files even if your server supports it 
- Your internet maybe noticably slower (But if you have a good connection, you probably wont notice it)
- This list will be updated as I continue to use synapse :)

Ok? Lets get into it

You dont need any special PC.

Personally I am running on an old laptop with i3 6th gen processor and 8 gigs of ram which has proxmox. Inside proxmox I have an LXC container of rocky linux which has 1 Core (May make it scalable in the future), 2Gb ram and 32 Gb Storage allocated to it.

On the ram side you might want to allocate more.

BTW using DHCP allocation is enough in my experience.

Even if changing IPs makes synapse glitch, from what I have seen IP addresses for my devices with DHCP rarely change.

Firstly install nginx via dnf (don't forget to enable the service) and keep the VM aside.

Switch the nameservers of your domain to cloudflare, this [article](https://community.cloudflare.com/t/step-1-adding-your-domain-to-cloudflare/64309) from cloudflare might help.

With your domain setup we can continue configuring the server

Enable the epel and powertools repositories:
```bash
# dnf repolist
repo id                           repo name
appstream                         Rocky Linux 8 - AppStream
baseos                            Rocky Linux 8 - BaseOS
extras                            Rocky Linux 8 - Extras
# dnf install dnf-plugins-core
# dnf config-manager --set-enabled powertools
# dnf install epel-release
```

Lets get the tunnel set up now: 
- Get the latest rpm for cloudflared from https://github.com/cloudflare/cloudflared/releases/latest and install the rpm with `rpm -i`
- Login with `cloudflared tunnel login`
- With that done, lets get to configuring the tunnel

Create a new tunnel `cloudflared tunnel create matrix`

This can be viewed by running `cloudflared tunnel list`

Now create config.yml in /etc/cloudflared with the following contents
```yaml
tunnel: xxxx-xxxx-xxxx-xxx-xxx # Tunnel UUID, get with cloudflared tunnel list
credentials-file: /root/.cloudflared/xxxx-xxxx-xxxx-xxx-xxx.json # Tunnel UUID
ingress:
  - hostname: matrix.yourdomain.com # for matrix server
    service: http://localhost:8008 # synapse port
  - hostname: riot.yourdomain.com # for element
    service: http://localhost:80 
  - service: http_status:404 # 404 redirect
```
- Now route the tunnels to the dns via cname with 

```bash
# cloudflared tunnel route dns matrix matrix.yourdomain.com 
2022-01-10T04:31:15Z INF Added CNAME matrix.yourdomain.com which will route to this tunnel tunnelID=xxx-xxx-xxx-xxxxx-xxx
# cloudflared tunnel route dns matrix riot.yourdomain.com
2022-01-10T04:31:15Z INF Added CNAME riot.aryak.ml which will route to this tunnel tunnelID=xxxxx-xxxx-xxxx-xxxx-xxxx
```
- Now test and see if it works by running `cloudflared tunnel run matrix`

Note: Matrix.yourdomain.com won't load as its trying to connect to port 8008. Riot.yourdomain.com will load tho.
- Create systemd service with cloudflared service install and enable it
