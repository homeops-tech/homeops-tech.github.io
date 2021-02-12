---
layout: post
title:  "Build Ubuntu Desktop With Packer"
date:   2020-02-15
image: /images/posts/packer.jpg
tags: [packer,vagrant,devops,ubuntu,ssh,bare metal,linux]
---

In one of my [previous posts](http://www.homeops.tech/2020/01/01/SSH-Pass-Through-Docker/) I used a third party vagrant box. This box was configured to load the GDM window manager and came with things like docker pre-installed. However as with most things in system imaging I needed to customize my images. This is normally due to requirements like size and configuration. In addition building a bare metal iso installed version of the base image allows me to more easily upgrade the installation over time. This is sort of best of both worlds approach to system imaging as you have the ability to use a image in production but also to upgrade the system using the standard OS installer.

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
            <td>packer</td>
            <td>1.6.6</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>vagrant</td>
            <td>Vagrant 2.2.14</td>
            <td>Ubuntu</td>
        </tr>
    </tbody>
</table>

Installing using the ubuntu desktop iso is an interesting experience to say the least. The little practical examples I found essentially rewrite the boot command using a series of backspaces. This is pretty interesting to watch:

![Animated-Gif](/images/posts/packer.gif)

```json
                "<esc><esc><f6><esc><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
```

> I recently bought a [CHG90](https://amzn.to/3dcAAfE) gaming monitor and it was actually able to not line wrap this in vim for me. Stay tuned for a review on said monitor - lol

I'd be interested to hear if anyone has a better way of doing this, my first instinct is to just built a custom iso. This is easier in PXE booting in my expierence. This reminds me a bit of apple scripts system events framework.

I also had to to lower the boot wait on my box from 10 to 6 seconds e.g. `"boot_wait": "6s"`. Otherwise I would miss the F6 key invocation timeout. This makes sense as its essentially just spamming the boot loaded like we all do when on hardware boxen.

  
`ubuntu-1804-desktop-vagrant.json`
  

{% gist ba02ee0d3c0383bcef92bf772877a197 ubuntu-1804-desktop-vagrant.json %}

You can build this box yourself with 

```shell
packer build ubuntu-1804-desktop-vagrant.json
```

That will build the box and output is as `ubuntu-18.04-virtualbox.box`

You can then create a Vagrantfile like so:

  
`Vagrantfile`
  


```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu-18.04-virtualbox.box"

  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    vb.gui = true
    vb.memory = '2048'
  end

  config.ssh.username = "vagrant"
  config.ssh.password = "vagrant"
end
```

...and finally boot the box up with 


```shell
vagrant up
```

You should be greeted with a Graphical interface that can be logged in with the `vagrant:vagrant` user:password combo.
