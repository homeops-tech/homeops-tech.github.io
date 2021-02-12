---
layout: post
title:  "Provision Puppetserver with Packer"
date:   2020-02-01
image: /images/posts/puppet.jpg
tags: [puppet,devops,packer,kickstart,openssl,puppetserver,esxi,how to]
---

When I left Puppet a year ago I decided to move off of the [Puppet Enterprise](https://puppet.com/products/puppet-enterprise/) Install I had been upgrading throughout my [7 year tenure](https://www.linkedin.com/in/acidprime/) at Puppet.  I realized I didn't need to keep up to date with the latest commercial features anymore. It gave me a long over due opportunity to setup a bare bones puppetserver. I try to use the best tools for the job and Puppet is normally one of the best configuration management solutions when you use it right. Puppet is thus one of the first servers I provision. Given its central role Puppet itself needs to be easily automated and upgraded over time. Frankly this is something most admins dont do enough of. In this how to, I will walk you through how I provision puppet server with [Packer](https://learn.hashicorp.com/packer).

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
            <td>6.19.1</td>
            <td>Centos</td>
        </tr>
        <tr>
            <td>puppetserver</td>
            <td>6.14.1</td>
            <td>Centos</td>
        </tr>
        <tr>
            <td>Centos</td>
            <td>7.9.2009</td>
            <td>Centos</td>
        </tr>
    </tbody>
</table>

I started with a Centos 7 base for my installation as Puppet needs to be relatively stable given most of the configurations flow from it. Centos is in my experience the most common installation candidate in the puppet ecosystem.

> Given the recent Centos news I may revisit this choice in the future. But I sometimes resolve to let things be somewhat static when they are this central especially in a home lab.

I will cover migrating certificate authorities to [Vault](https://www.hashicorp.com/products/vault) in a future article, however to get started with we need to generate a Certificate Authority PEM puppet server will use to sign its own and agent certificates.


## Pre-Generating the Certificate Authority 

While this process is normally automatic, I like to build my Puppet server multiple times from scratch to get the automation as dialed in as possible. This means I want to preserve the CA files between installations once I get it right so agents don't need their certificates recreated each time I rebuild Puppet. As such I put these certificates on shared storage like an NFS export on my [Synology](https://amzn.to/3tU1CP6). Scripting the initial CA deployment is essentially the same as what happens the second build where these files pre-exist before installation. This also has some huge advantages for moving from physical to virtual and back again.

I run the following which mounts a SSL directory only to my puppet host.

```shell
mkdir -p /etc/puppetlabs/puppet/ssl/
mount -t nfs synology.homeops.tech:/volume1/ssl /etc/puppetlabs/puppet/ssl/
```

> If this was production you should would want to consider the security implications e.g. what machines can access your private keys. I do want to migrate to Kerberos in the future as well, but most of the core infrastructure I use builds the next infrastructure and thus needs to be solid and simple.

Once your NFS export is mounted at the default "ssldir" for puppetserver you can change directories and generate a new certificate authority. First though I need to generate an openssl configuration.

  
`openssl.conf`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 openssl.conf %}


  
`ca.sh`
  

{% gist 3c673dfebd5a0b309cbc22095da348e7 ca.sh %}

I increment the CA serial as the CA and puppet server certs will both be in the first few slots. My long term plans are to make this an intermediate CA with some PKC11 automation and a [Nitro Key](https://shop.nitrokey.com/shop/product/nk-pro-2-nitrokey-pro-2-3) I'm planning on using for Vault. check back here for updates.

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
  homeops:
    remote: 'git@github.com:homeops-tech/control-repo.git'
    basedir: '/etc/puppetlabs/code/environments'
git:
  provider: shellgit
```

Using the `r10k.yaml` above you can bootstrap the initial control repo on your puppet server. This will download all environments (branches) and modules specified in your Puppetfile. I normally have Puppet managed by Puppet in most case and so I also perform a puppet run at the end. There are some chicken before the egg issues you might encounter like hiera, so testing this from scratch is a good idea.

  
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
