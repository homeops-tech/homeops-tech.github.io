---
layout: post
title:  "Using Terraform with Proxmox"
date:   2020-10-16
image: /images/posts/terraform.jpg
tags: [terraform,proxmox,cloud-init]
---

In my previous post on using [Razor with Cloud Init](http://www.homeops.tech/2020/07/15/Razor-and-Cloud-Init/) I was able to get newly created VMs to PXE boot to razor and automatically install debian. This allows me to quickly deploy VMs for testing and hosting services. However to create these VMs still required clicking though multiple pages of settings in the Proxmox UI. Given I use terraform so much as work I wanted to use it in my home lab. I this quick post I will show you some example
code to provision a VM in proxmox with terraform.

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
            <td>terraform</td>
            <td>v0.14.5</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>Ubuntu</td>
            <td>20.04</td>
            <td>Ubuntu</td>
        </tr>
    </tbody>
</table>

One great things about the terraform ecosystem is like other tools such as Puppet it has a  [provider](https://github.com/Telmate/terraform-provider-proxmox) architecture. This plugin system allows users to add resources to terraform that can then be use along side built-in ones. You could for instance use terraform to spin up a vault node and then use the vault node to provision the configuration. The same could happen with the Gitlab provider.


# Installing the Provider

  
`versions.tf`
  


```hcl
 terraform {
   required_providers {
     proxmox = {
       source  = "Telmate/proxmox"
       version = "2.6.7"
     }
   }
 }
```

In the most recent version of terraform you specify the provider version by convention in the `versions.tf` file.
This is akin to something like a Puppetfile or a Git Submodule. It allows you to pin a version of this provider so that you and your team don't have compatibility issues across invocations. 

# Provisioning a Bare Metal Node

  
`main.tf`
  

```hcl
resource "proxmox_vm_qemu" "metrics" {
   name = "metrics"
   desc = "Graphite, Grafana and Collectd"
 
   network {
     model    = "virtio"
     bridge   = "vmbr0"
     firewall = true
   }
 
   onboot = true
 
   os_type = "cloud-init"
 
   target_node = "proxmox"
   boot        = "order=net0;scsi0"
   iso = "none"
 
   cores   = "2"
   sockets = "1"
   vcpus   = "0"
   cpu     = "host"
   memory  = "2048"
   scsihw  = "virtio-scsi-pci"
 
 
   disk {
     type    = "scsi"
     size    = "64G"
     format  = "qcow2"
     storage = "mute"
   }
 
}

```

 Here is an example of Provisioning the Graphana node covered in this [previous post](http://www.homeops.tech/2020/05/20/Graphite-And-Grafana-With-Puppet/). Lets break down some of the important bits here.

```hcl
...
  boot        = "order=net0;scsi0"
... 
```

As I mentioned in the [Razor with Cloud Init](http://www.homeops.tech/2020/07/15/Razor-and-Cloud-Init/) ,razor server normally expects network booting to always happen. That includes after a node has been provisioned. This allows razor to centrally reinstall the node.

```hcl
...
    disk {
      type    = "scsi"
      size    = "64G"
      format  = "qcow2"
      storage = "mute"
    }
...
```

This configures the virtual scsi interface that will correspond to the `/dev/sda` entry in your preseed.

> At the time of this writing the format ends up being raw instead of qcow.  [Known Bug](https://github.com/Telmate/terraform-provider-proxmox/issues/291) 

If you wanted to simply boot from an iso here you would add:

```hcl
...
  boot        = "legacy=ncd"
  iso         = "synology:iso/debian-10.7.0-amd64-netinst.iso"
...
```

This is an exmample of an iso placed in the templates directory of the proxmox share.


# Cloud-init

```
...
os_type = "cloud-init"
...
```

We specify cloud-init here as in the razor example we are not creating this system from a packer image. However that is possible to do with templates in proxmox.


Ideally we would be able to add the following cloud-init config information

```
     sshkeys = <<EOF
       ssh-rsa AAAA...
   EOF

```

However the provider has a heavy handed error message that you cant use cloud-init without templates. Once this is resolved we should be able to use this as a cloud-drive to have proxmox merge configurations with our Razor Cloud Init configuration. This is the [bug](https://github.com/Telmate/terraform-provider-proxmox/issues/274) that covers that issue. I might expierement with forking the provider to get it working in the mean time.

As a side note, this should eventually allow us to tie into Terrform's  ability to generate cloud-init multi part files.


```hcl
 Source the Cloud Init Config file
  template  = file("${path.module}/templates/cloud.cfg")

  vars = {
    ssh_key = file("/home/acidprime/.ssh/id_rsa.pub")
    domain = "homeops.tech"
  }
}

data "cloudinit_config" "homeops" {
  gzip = false
  base64_encode = false

  part {
    content_type = "text/x-shellscript"
    content = "baz"
    filename = "init.sh"
  }
  # Main cloud-config configuration file.
  part {
    filename     = "init.cfg"
    content_type = "text/cloud-config"
    content      = "${data.template_file.script.rendered}"
  }
}

```

I will update this post once that bug is updated. 
