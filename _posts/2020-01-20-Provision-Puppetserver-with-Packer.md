---
layout: post
title:  "Provision Puppetserver with Packer"
date:   2020-01-01
image: /images/posts/puppet.jpg
tags: [puppet, devops,centos,packer,kickstart,openssl,puppetserver,esxi,how to]
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

I started with a Centos 7 base for my installation as Puppet needs to be realtivly stable given most of the configurations flow from it. Centos is in my expierence the most common installation candidate in the puppet ecosystem.

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

```shell
[ default ]
ca                      = root-ca

[ ca ]
default_ca              = root_ca

[ root_ca ]
dir               = /etc/puppetlabs/puppet/ssl/ca
certs             = $dir/certs
serial            = $dir/serial
database          = $dir/inventory.txt
private_key       = $dir/ca_key.pem
certificate       = $dir/ca_crt.pem
crl               = $dir/ca_crl.pem
unique_subject    = no
default_md        = sha1
default_days    = 365
default_crl_days= 365 
preserve        = no

[req]
default_bits = 2048
prompt = no
distinguished_name = req_distinguished_name
req_extensions     = req_ext

[ req_distinguished_name ]
CN = "Puppet CA: puppet.homeops.tech"

[v3_req]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ req_ext ]
subjectAltName = @alt_names

[alt_names]
DNS.1   = puppet.homeops.tech
DNS.2   = puppet

```

  
`ca.sh`
  

```shell
#!/bin/bash -x
rm -rf ca
rm -rf certs
mkdir -p ca
openssl genrsa -out ca/ca_key.pem 2048
openssl rsa -in ca/ca_key.pem -pubout -out ca/ca_pub.pem
openssl req \
  -x509 \
  -new \
  -nodes \
  -key ca/ca_key.pem \
  -sha256 \
  -days 3000 \
  -out ca/ca_crt.pem \
  -config openssl.conf
touch ca/inventory.txt
echo "03" > ca/serial
openssl ca \
  -create_serial \
  -config openssl.conf \
  -crldays 1460 \
  -gencrl \
  -out ca/ca_crl.pem
```

I increment the CA serial as the CA ad puppet server will both be in the first few slots.My long term plans are to make this an intermediate CA with some PKC11 automation and a Nitro Key I'm planning on using for Vault. check back here for updates.

> This script is also useful for regenerating your CA when you decide to reset it for testing. Please note the `rm -rf` commands in the begining of the script 

## Install Puppetserver

Once the certificate directory and files exists, all we need is to install Puppet server.

  
  `puppetserver.sh`
  

```
#!/bin/bash -x
wget https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
rpm -Uvh puppet6-release-el-7.noarch.rpm
yum install puppet -y
yum install puppetserver -y
chown -R  puppet:puppet /etc/puppetlabs/puppet/ssl/ca
puppetserver ca setup
systemctl start puppetserver
```

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

```shell
#!/bin/bash -x
yum install git -y
yum update -y nss curl libcurl
/opt/puppetlabs/puppet/bin/gem install r10k
mkdir -p ~/.ssh
ssh-keyscan github.com > ~/.ssh/known_hosts
cp /etc/puppetlabs/puppet/ssl/id_rsa ~/.ssh/id_rsa
/opt/puppetlabs/puppet/bin/r10k \
  deploy environment \
  -p \
  -v debug2 \
  --color \
  -c /etc/puppetlabs/puppet/ssl/r10k.yaml
/opt/puppetlabs/bin/puppet agent -t
```

And your done. From here all additional management can flow from your version controlled puppet code rather then bash scripts. If you want to build an Image from your Puppet server you can use the above examples with something like `packer`


## Deploy Server with Packer

Here is an example packer file that can be used with esxi. I normally leave the build running for at the end:


`puppet.json`
  
```json
{
   "_comment": "Build with `ESXI_PASSWORD=foo packer build puppet.json`",
   "variables": {
     "esxi_password": "{{env `ESXI_PASSWORD`}}"
   },
   "builders": [
     {
       "vm_name": "puppet.homeops.tech",
       "type": "vmware-iso",
       "iso_url": "http://ftp.iij.ad.jp/pub/linux/centos-vault/7.2.1511/isos/x86_64/CentOS-7-x86_64-DVD-1511.iso",
       "iso_checksum": "4c6c65b5a70a1142dadb3c65238e9e97253c0d3a",
       "iso_checksum_type": "sha1",
       "ssh_username": "packer",
       "ssh_password": "packer",
       "ssh_wait_timeout": "10m",
       "disk_size": "100000",
       "tools_upload_flavor": "linux",
       "guest_os_type": "centos-64",
       "remote_type": "esx5",
       "remote_username": "root",
       "remote_password": "{{user `esxi_password`}}",
       "remote_datastore": "synology.homeops.tech",
       "remote_cache_datastore": "datastore1",
       "remote_host": "esxi.homeops.tech",
       "ssh_wait_timeout": "1000s",
       "keep_registered": true,
       "headless": "false",
       "shutdown_command": "sudo /sbin/halt -p",
       "floppy_files": [
           "floppy/puppet.cfg"
         ],
       "boot_command": [
         "<tab> inst.text inst.ks=hd:fd0:/puppet.cfg <enter><wait>"
       ],
       "vmx_data": {
         "ethernet0.networkName": "VM Net",
         "config.version": 8,
         "virtualHW.version": 8,
         "ethernet0.present": "TRUE",
         "ethernet0.virtualDev": "e1000",
         "ethernet0.startConnected": "TRUE",
         "ethernet0.addressType": "generated",
         "ethernet0.generatedAddressOffset": "0",
         "ethernet0.wakeOnPcktRcv": "FALSE",
         "memsize": "3096",
         "cpuid.coresPerSocket": "1",
         "numvcpus": "4",
         "vhv.enable": "TRUE",
         "RemoteDisplay.vnc.enabled": "TRUE",
         "RemoteDisplay.vnc.port":  "5900"
       }
     }
   ]
 }
```

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

  
`
puppet.cfg
`
  

```shell
# CentOS 7.x kickstart file - puppet.cfg
# Required settings
lang en_US.UTF-8
keyboard us
rootpw packer
authconfig --enableshadow --enablemd5
timezone UTC

# Optional settings
install
cdrom
user --name=packer --plaintext --password packer
services --disabled=NetworkManager --enabled=network,sshd
unsupported_hardware
network --bootproto=dhcp
firewall --disabled
selinux --permissive
bootloader --location=mbr
text
skipx
zerombr
clearpart --all --initlabel
autopart --type=lvm
firstboot --disabled
selinux --permissive

reboot
network --onboot yes --device ens33 \
  --bootproto=static \
  --ip=192.168.53.53 \
  --netmask=255.255.255.0 \
  --gateway=192.168.53.1 \
  --nameserver=192.168.53.60 \
  --nameserver=192.168.53.70 \
  --noipv6 \
  --hostname=puppet.homeops.tech

%packages --nobase --ignoremissing --excludedocs
# packer needs this to copy initial files via scp
openssh-clients
@base
kernel-headers
kernel-devel
gcc
make
perl
curl
wget
bzip2
dkms
patch
net-tools
git
sudo
nfs-utils
%end

%post --log=/var/log/post-install.log
# Disable 'consistent network device naming' and make things act more or less reasonable in a VM-oriented context.
echo > /etc/udev/rules.d/70-persistent-net.rules
echo > /etc/udev/rules.d/75-persistent-net-generator.rules

sed -i'' -e '/UUID=/d' /etc/sysconfig/network-scripts/ifcfg-ens33
sed -i'' -e '/HWADDR=/d' /etc/sysconfig/network-scripts/ifcfg-ens33
sed -i'' -e '/DHCP_HOSTNAME=/d' /etc/sysconfig/network-scripts/ifcfg-ens33
sed -i'' -e 's/NM_CONTROLLED=.*/NM_CONTROLLED="no"/' /etc/sysconfig/network-scripts/ifcfg-ens33

echo "Setting up ifcfg-ens33"

for nic in /etc/sysconfig/network-scripts/ifcfg-eth*; do sed -i /HWADDR/d $nic; done
sed -i -e '/#UseDNS yes/a UseDNS no' /etc/ssh/sshd_config
yum -y remove networkmanager

# Configure Synology LDAP
authconfig --kickstart --enableshadow --enablemd5 --enableldap --enableldapauth --ldapserver synology.homeops.tech --ldapbasedn dc=homeops,dc=tech

# configure packer user in sudoers
echo "%packer ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/packer
chmod 0440 /etc/sudoers.d/packer
cp /etc/sudoers /etc/sudoers.orig
sed -i "s/^\(.*requiretty\)$/#\1/" /etc/sudoers

# Configure Puppet
mkdir -p /etc/puppetlabs/puppet/ssl

echo '#!/bin/sh' > /etc/rc.d/rc.local
chmod 0755 /etc/rc.d/rc.local
echo 'mkdir -p /etc/puppetlabs/puppet/ssl' >> /etc/rc.d/rc.local
echo 'mount -t nfs synology.homeops.tech:/volume1/ssl /etc/puppetlabs/puppet/ssl/' >> /etc/rc.d/rc.local

echo 'yum clean all' >> /etc/rc.d/rc.local
echo 'yum update' >> /etc/rc.d/rc.local

echo '!!!!!Replace wirh your ca.sh!!!!!'>> /etc/rc.d/rc.local
echo '!!!!!Replace wirh your puppetserver.sh!!!!!'>> /etc/rc.d/rc.local

echo '/usr/bin/rm -rf /etc/rc.d/rc.local' >> /etc/rc.d/rc.local
%end

```

And there you have it after running `ESXI_PASSWORD='pa$$w0rd' packer build puppet.json` , packer will build your puppetserver and leave it running. You can then make a template or use this as a bare bones provisioning system.

In future articles I'll show you how to take this Puppet server and have it build a Razor server to provision the rest of your infrastructure.
