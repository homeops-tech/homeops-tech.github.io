---
layout: post
title:  "Docker SSH Pass through links in Linux"
date:   2020-01-01
image: /images/posts/docker.jpg
tags: [puppet,devops,docker,vagrant,virtualbox,ubuntu,gdm,ssh,how to]
---

I have always liked links. When I used to make [Cocoa Apps](https://github.com/acidprime/winnebago) for enterprise customers I liked adding things like Webkit to load cool HTML links and work with custom URLs. There is something about making something pop-up on screen from an otherwise static application that seems fancy to me. Sometimes I need to setup an environment (often linux) that needs a user to interact with a docker container. While docker exec is very similar to ssh, it lacks a
URL structure to integrate a website and docker together. In this how-to I will show you how to add `ssh://` links to Ubuntu and pass through those connections to docker.

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
            <td>puppet</td>
            <td>6.21.0</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>docker</td>
            <td>18.0.9</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>vagrant</td>
            <td>2.2.14</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>virtualbox</td>
            <td>6.1.16</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>Ubuntu</td>
            <td>18.04</td>
            <td>Ubuntu</td>
        </tr>
    </tbody>
</table>

I start with Ubuntu here but the user side of this is essentially platform agnostic. The first thing to understand is that when a user logs into ssh they have a shell thats executed. You can configure that shell to be a wrapper script. There are ways to secure this

> Given the recent Centos news I may revisit this choice in the future. But I sometimes resolve to let things be somewhat static when they are this central especially in a home lab.

I will cover migrating certificate authorities to [Vault](https://www.hashicorp.com/products/vault) in a future article, however to get started with we need to generate a Certificate Authority PEM puppet server will use to sign its own and agent certificates.


## Pre-Generating the Certificate Authority 

While this process is normally automatic, I like to build my Puppet server multiple times from scratch to get the automation as dialed in as possible. This means I want to preserve the CA files between installations once I get it right so agents don't need their certificates recreated each time I rebuild Puppet. As such I put these certificates on shared storage like an NFS export on my synology. Scripting the intial CA deployment is essentially the same as what happens the second build where these files pre-exist before installation. This also has some huge advantages for moving from physical to virtual and back again.

I run the following which mounts a SSL directory only to my puppet host.

```shell
mkdir -p /etc/puppetlabs/puppet/ssl/
mount -t nfs synology.homeops.tech:/volume1/ssl /etc/puppetlabs/puppet/ssl/
```

> If this was production you should would want to consider the security implications e.g. what machines can access your private keys. I do want to migrate to Kerberos in the future as well, but most of the core infrastructure I use builds the next infrasturcture and thus needs to be solid and simple.

Once your NFS export is mounted at the default "ssldir" for puppetserver you can change directories and generate a new certificate authority. First though I need to generate an openssl configuration.

  
`openssl.conf`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 openssl.conf %}


  
`ca.sh`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 ca.sh %}

I increment the CA serial as the CA ad puppet server will both be in the first few slots.My long term plans are to make this an intermediate CA with some PKC11 automation and a [Nitro Key](https://shop.nitrokey.com/shop/product/nk-pro-2-nitrokey-pro-2-3) I'm planning on using for Vault. check back here for updates.

> This script is also useful for regenerating your CA when you decide to reset it for testing. Please note the `rm -rf` commands in the begining of the script 

## Install Puppetserver

Once the certificate directory and files exists, all we need is to install Puppet server.

  
`puppetserver.sh`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 puppetserver.sh %}

> I'm not installing PuppetDB here as frankly I just don't need it at this scale, I can still use flat files to use storeconfigs out of the box.

At this point you should have a PuppetServer running but it wont have any code to deploy. While I could keep the code on a NFS mount, I normally like to have it clone fresh to determine if any repos are out of date. I do this via [`r10k`](https://github.com/puppetlabs/r10k). Given we have an NFS mount that will persist across installations we can use it to host both the Github Deploy key and the r10k.yaml configuration itself (or at least a bootstrap copy if you prefer to later manage it with puppet).

  
`r10k.yaml`
  

```yaml
---
sources:
  wallcity:
    remote: 'git@github.com:homeops-tech/control-repo.git'
    basedir: '/etc/puppetlabs/code/environments'
git:
  provider: shellgit
```

Using the `r10k.yaml` above you can bootstrap the inital control repo on your puppet server. This will download all environments (branches) and modules specified in your Puppetfile. I normally have Puppet managed by Puppet in most case and so I also perform a puppet run at the end. There are some chicken before the egg issues you might encounter like hiera, so testing this from scratch is a good idea.

  
`code.sh`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 code.sh %}

And your done. From here all additional management can flow from your version controlled puppet code rather then bash scripts. If you want to build an Image from your Puppet server you can use the above examples with something like `packer`


## Deploy Server with Packer

Here is an example packer file that can be used with esxi. I normally leave the build running for at the end:

  
`puppet.json`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 puppet.json %}

I use the following kickstart file to tie every thing together

```shell
mkdir floppy && cd floppy
vim puppet.cfg
```

For testing, you can fire up a local http server temporarily.
cd to the directory where this ks.cfg file resides and run the following:
`python -m SimpleHTTPServer`
You don't have to restart the server every time you make changes.  Python
will reload the file from disk every time.  As long as you save your changes
they will be reflected in the next HTTP download.  Then to test with
a PXE boot server, enter the following on the PXE boot prompt:
`   > linux text ks=http://<your_ip>:8000/ks.cfg`

Once your kickstart file is ready you can load it into the virtual floppy drive.

  
`puppet.cfg`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 puppet.cfg %}

And there you have it after running `ESXI_PASSWORD='pa$$w0rd' packer build puppet.json` , packer will build your puppetserver and leave it running. You can then make a template or use this as a bare bones provisioning system.

In future articles I'll show you how to take this Puppet server and have it build a Razor server to provision the rest of your infrastructure.
