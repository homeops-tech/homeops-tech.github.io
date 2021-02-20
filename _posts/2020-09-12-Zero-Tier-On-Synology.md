---
layout: post
title:  "Using Synology as a ZeroTier Bridge"
date:   2020-07-15
image: /images/posts/network.jpg
tags: [synology,zero tier,dsm]
---

I was introduced to [Zero tier](https://www.zerotier.com/) a few years ago. I find it pretty solid and the free plan works well as a sort of reverse NAT style VPN'esk solution.  Each node gets a multi homed IP that's always available. This means I can ssh into my laptop no matter where it is, or have it backup to an internal server consistently. You don't have to open any ports up as your routing traffic through their servers. Additionally you don't need a static IP to make this work.

<!--more-->

<table>
    <caption>Versions tested</caption>
    <tbody>
        <tr>
            <th>Software</th>
            <th>Version</th>
            <th>OS</th>
        </tr>
        <tr>
            <td>DSM</td>
            <td>DSM 6.2.2-24922 Update 6</td>
            <td>Synology</td>
        </tr>
    </tbody>
</table>

You need at least one node that's consistently online to route the traffic from Zero Tier to your devices. You can buy stand alone units for this but I often use my Synology for central services like this. To get started you will need the following:

![ZeroDownload](/images/posts/zerodownload.png)

* Sign up for account: [https://my.zerotier.com/](https://my.zerotier.com/)
* Download correct Synology Package for your arch [https://download.zerotier.com/dist/synology/](https://download.zerotier.com/dist/synology/)

> Use the [Github Wiki](https://github.com/SynoCommunity/spksrc/wiki/Architecture-per-Synology-model) to find the right arch


# Install Package

Using the Package Center add the package via the Manual Install Button

![ManualInstall](/images/posts/zerominstall.png)


# Join the Zero Tier network

![ZerConfig](/images/posts/zeroconfig.png)

Grab your network ID from [https://my.zerotier.com/](https://my.zerotier.com/) and launch the Zero Tier app you installed via Package Center.


# Enable Routing & Nat

  
`/etc/sysctl.conf`
  

```shell
net.ipv4.conf.all.forwarding=1
net.ipv4.conf.default.forwarding=1
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.default.forwarding=1
```

ssh into your Synology and edit the sysctl configuration. Then apply your changes

```shell
$ sysctl --system
```

> This needs to be automated but I haven't decided from a gem installed puppet or hacked ansible for managing my synology yet.

# Enable IP Tables Rules

```
sudo -s
/sbin/iptables -t nat -A POSTROUTING -o bond0 -j MASQUERADE
/sbin/iptables -A FORWARD -i bond0 -o eth50 -m state --state RELATED,ESTABLISHED -j ACCEPT
/sbin/iptables -A FORWARD -i eth50 -o bond0 -j ACCEPT
```

Where is `bond0` is your LAN interface and `eth50` is your zerotier interface

> I have a bond as I use LACP to bond my 4 Ethernet ports together on my DS1515+


# Persistance

These need to be set at reboot to stay persistent so you should add them to `/etc/rc.local`


# Configure Bridging

![ZeroTier](/images/posts/zerotier.png)

Add your synology to your network and Allow Ethernet bridging. Create a managed route as shown above.


# Zero Tier on Phone

![PhoneInstall](/images/posts/zerophonejoin.png)

Now that you have an ethernet bridge you can install the Zero Tier app on your phone and join the network.

![CarRemote](/images/posts/carremote.png)

As you can see I can connect to my arduino project that allows me to lock my car doors at night. 

If you followed my [DNS Post](http://127.0.0.1:4000/2020/06/06/DNS-With-Puppet/) then you can actually use split DNS here as well by allowing this subnet to make calls to your named service.
