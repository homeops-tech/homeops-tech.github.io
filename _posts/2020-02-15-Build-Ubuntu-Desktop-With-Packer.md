---
layout: post
title:  "Build Ubuntu Desktop With Packer"
date:   2020-02-15
image: /images/posts/packer.jpg
tags: [packer,vagrant,devops]
---

In one of my [previous posts](http://www.homeops.tech/2020/01/01/SSH-Pass-Through-Docker/) I used a third party box. This box was configured to load the window manager and came with things like docker pre-installed. However as with most things in system imaging I needed to customize my images due to requirements like size. In addtion building a bare metal iso installed version of the base image allows me to more easily upgrade the installation over time. This is sort of best of both worlds approache to system imaging as you have the ability to use a image in production but also to upgrade the system using the standard OS installer.

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

Installing using the ubuntu desktop iso is an interesting expierence to say the least. The little practical examples I found essentially rewrite the boot command using a series of backspaces. This is pretty interesting to watch:

![Animated-Gif](/images/posts/packer.gif)

```json
                "<esc><esc><f6><esc><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
```

> I recently bought a [CHG90](https://amzn.to/3dcAAfE) gaming monitor and it was actually able to not line wrap this in vim for me. Stay tuned for a review on said monitor - lol

I'd be interested to hear if anyone has a better way of doing this, my first instinct is to just built a custom iso. This is easier in PXE booting in my expierence. 

I also had to to lower the boot wait on my box from 10 to 6 seconds e.g. `"boot_wait": "6s"`. This makes sense as its essentially just spamming the boot loaded like we all do when on hardware boxen.

  
`ubuntu-1804-desktop.json`
  

```json
{
    "variables": {
        "iso_checksum": "c0d025e560d54434a925b3707f8686a7f588c42a5fbc609b8ea2447f88847041",
        "iso_checksum_type": "sha256",
        "non_gui": "false",
        "vboxversion": "04.3",
        "osdetails": "ubuntu-18.04.3-amd64",
        "ssh_fullname": "vagrant",
        "ssh_password": "vagrant",
        "ssh_username": "vagrant",
        "hostname":   "ubuntu.vagrant.vm",
        "ubuntu_images_url": "https://releases.ubuntu.com/18.04",
        "version": "{{ timestamp }}",
        "version_desc": "Latest kernel build of Ubuntu Vagrant images based on Ubuntu 18.04.3 (With Desktop)"
    },
    "builders": [
        {
            "type": "virtualbox-iso",
            "guest_os_type": "Ubuntu_64",
            "iso_url": "{{ user `ubuntu_images_url` }}/ubuntu-18.04.5-desktop-amd64.iso",
            "iso_checksum": "file:{{ user `ubuntu_images_url` }}/SHA256SUMS",
            "boot_command": [
                "<esc><esc><f6><esc><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
                " auto=true priority=critical noprompt ",
                " hostname={{user `hostname`}} ",
                " url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg ",
                " automatic-ubiquity ",
                " ubiquity/reboot=true ",
                " keyboard-configuration/layoutcode=us ",
                " debian-installer=en_US auto locale=en_US kbd-chooser/method=us ",
                " languagechooser/language-name=English ",
                " countrychooser/shortlist=IN ",
                " localechooser/supported-locales=en_US.UTF-8 ",
                " netcfg/choose_interface=auto ",
                " boot=casper ",
                " initrd=/casper/initrd ",
                " quiet splash noprompt noshell ",
                " --- <wait>",
                "<enter><wait>"
            ],
            "boot_wait": "6s",
            "disk_size": "40000",
            "headless": false,
            "http_directory": "http",
            "ssh_password": "{{user `ssh_password`}}",
            "ssh_username": "{{user `ssh_username`}}",
            "ssh_port": 22,
            "ssh_wait_timeout": "10000s",
            "shutdown_command": "echo 'vagrant'|sudo -S shutdown -P now",
            "guest_additions_path": "VBoxGuestAdditions_{{.Version}}.iso",
            "virtualbox_version_file": ".vbox_version",
            "vm_name": "packer-ubuntu-18.04-amd64",
            "vboxmanage": [
                [
                    "modifyvm",
                    "{{.Name}}",
                    "--memory",
                    "1024"
                ],
                [
                    "modifyvm",
                    "{{.Name}}",
                    "--cpus",
                    "1"
                ],
                [
                    "modifyvm",
                    "{{.Name}}",
                    "--audio",
                    "none"
                ],
                [
                    "modifyvm",
                    "{{.Name}}",
                    "--usb",
                    "off"
                ]
            ]
        }
    ],
    "provisioners": [
        {
            "type": "shell",
            "scripts": [
                "scripts/init.sh",
                "scripts/cleanup.sh"
            ]
        }
    ],
    "post-processors": [
        {
            "type": "vagrant",
            "compression_level": "8",
            "keep_input_artifact": true,
            "output": "ubuntu-18.04-{{.Provider}}.box"
        }
    ]
}
```
