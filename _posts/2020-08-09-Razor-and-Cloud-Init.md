---
layout: post
title:  "Using Razor Server with Cloud-Init"
date:   2020-07-15
image: /images/posts/razor.jpg
tags: [puppet,isc-dhcpd,hiera,how to,razor,terraform]
---

Bare metal imaging is certainly not the rage anymore. Tools like packer allow us to walk the line between golden images and infrastructure as code. I really like packer but I often want more tools in my belt. In a home lab I want to be able to install physical machines with ease the way I install virtual ones at work. Looking at the commit history of razor you can see that its not something being actively updated that much anymore. That said, its actually very versatile software when you get know it. Its light weight and adding
modern tools to the mix makes it work quite well. Its lack of use can also be attributed to it being difficult to get started with. Hopefully this post can help.

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
            <td>centos</td>
        </tr>
        <tr>
            <td>centos</td>
            <td>7</td>
            <td>centos</td>
        </tr>
    </tbody>
</table>

  
  `Puppetfile`
  


```ruby
mod 'razor',
  :git => 'https://github.com/acidprime/puppet-razor.git',
  :ref => 'master'

```


# Installing Razor Server 

```puppet
   include java
   class { 'postgresql::server':
     ip_mask_allow_all_users    => '0.0.0.0/0',
     postgres_password          => 'razor',
   }
 
   include 'wget'
 
   postgresql::server::db { 'razor':
     user     => 'razor',
     password => postgresql_password('razor', 'secret'),
   }
 
   file {'/etc/razor':
     owner  => 'razor',
     ensure => 'directory',
   }
 
   file {'/opt/razor':
     owner  => 'razor',
     ensure => 'directory',
   }
 
   file {'/var/lib/razor/repo-store':
     ensure => 'directory',
     owner  => 'razor',
   }
 
   file {'/etc/puppetlabs/razor-server/config.yaml':
     ensure => 'link',
     source => '/etc/razor/config.yaml',
   }
 
   class { 'razor':
     database_hostname   => '127.0.0.1',
     compile_microkernel => false,
     database_password   => 'secret',
     enable_tftp         => false,
   }
```
I chose razor server, because its actually light weight. Systems like foreman want to take over multiple parts of your infrastructure for their narrative to work. In my case I had two main goals.

1. PXE boot new VMs in proxmox
2. PXE boot my Intel NUC 

Its very possible you have never heard of razor before. The basic concept is very intuitive once you start using it. Once you have a razor server, the PXE booted server first boots into an intermediary "micro" kernel which is essentially a tiny Centos7 box. This micro kernel, reports back on the various details of the machine its booted from. These "facts" are collected using Puppet's `facter` tool. The razor server then can set the next boot to be one of installation or pass through. In other
words on say my Intel NUC I configure to always network boot. This allows razor to check if the box should be installed or perhaps reinstalled. Razor even has IPMI support for proper server hardware to reboot a new box.

>I tried to get the IPMI libvirt tools to work with Proxmox only to find that proxmox doesnt support them. Not a huge deal in my case as I often can reboot a node easily. 

To reinstall a node type

```shell
razor reinstall-node node30

```


In my previous post I walked you through installing ISC-DHCPD. Those DHCP/DNS/TFTP servers allow me to PXE boot machines in my network with razor. This exposes the Web URLs that razor uses to advance the node on to the next step in the process of being installed.


# Node Protection


```puppet
razor::razor_yaml_setting { "all/checkin_interval":
  value => "15"
}

razor::razor_yaml_setting { "all/protect_new_nodes":
  value => false
}
```

By default Razor protects new nodes (as to not erase them!). I actually don't want that protection here as I want to be able to spin a new node up with `terraform`. I'm also setting the `checkin` interval here as its required in the template.

# Razor Repo


```puppet
   razor_repo { 'debian-buster-amd64-netboot':
     ensure  => 'present',
     iso_url => "https://deb.debian.org/debian/dists/buster/main/installer-amd64/current/images/netboot/mini.iso",
     task    => 'debian',
   }
```

In this example we configure a repo for the debian netboot mini iso. One of the first things I found confusing about razor was new terms it uses for common imaging steps. Some like repo more or less make sense over time.


# Razor Tags

```puppet
   razor_tag { 'virtual':
     ensure => 'present',
     rule   => ['=', ['fact', 'is_virtual'], true]
   }
 
   razor_tag { 'proxmox':
     ensure => 'present',
     rule   => ['=', ['fact', 'macaddress'], "54:b2:03:1c:0c:bd"]
   }
```

A tag in razor is a rule set that will be applied to a node. In these examples I am creating a tag `virtual` that will be applied many times to all virtual nodes. I'm also creating a tag `proxmox` that will likely only ever be applied once. This tag is matched only by the mac address. However I can modify it to match multiple mac addresses if I want to install multiple servers.

> This is sort of the tribble style of deployment as my razor server runs on a proxmox node which in turn can create other proxmox nodes in the cluster.

# Razor Broker

```puppet
   razor_broker { 'cloud-init':
     ensure        => 'present',
     broker_type   => 'cloud-init',
     configuration => hiera('cloud-init'),
   }
   razor::broker {'cloud-init':
     module => 'profile',
     root => '/opt/puppetlabs/server/apps/razor-server/share/razor-server/brokers',
   }
```

A razor broker is essentially a script that runs after first boot. While in ye'olden times that was likely `rc.local`, since systemd we would customize our own script. In my case while I am not adverse to maintaining custom scripts, I wanted to use common tools I use in my day job. Cloud-init is the defacto standard for this sort of first boot config. It even supports puppet's advanced install features.

The code above references a custom broker (see below). Lets break it down. 

```puppet
...
configuration => hiera('cloud-init'),
...
```

This line allows us to load out cloud-cfg which is the configuration cloud-init uses to configure its' self on first boot, from hiera. This is nice as most examples on the internet are going to copy and paste-able. Albeit with a little more indentation given its nested under the `cloud-init` key. We don't get many of the other benefits of hiera here as its ability to merge data is not relevant for this use case. That's because this hiera call is evaluated in the context of the razor-server and not he respective node. That shouldn't really be an issue given cloud-init itself can pull in multiple sources of data from multiple types of input.

The `razor::broker` is copying our custom broker from our module, `profile` to its final location on the razor server. Before we disect that broker (below) lets tie everything together.

> Tiny bit hand wavey here as the puppet config needs to be per node, but I do that based on the certname rather then the csr attributes currently.


# Razor Policies

```puppet
 razor_policy { 'install_debian_on_virtual':
     ensure        => 'present',
     repo          => 'debian-buster-amd64-netboot',
     task          => 'buster',
     broker        => 'cloud-init',
     hostname      => 'node${id}.homeops.tech',
     root_password => 'pveroot',
     max_count     =>  100,
     node_metadata => {"os_version" => "buster"},
     tags          => ['virtual'],
   }
 
   razor_policy { 'proxmox_install':
     ensure        => 'present',
     repo          => 'debian-buster-amd64-netboot',
     task          => 'buster',
     broker        => 'cloud-init',
     hostname      => 'proxmox.homeops.tech',
     root_password => 'pveroot',
     max_count     =>  1,
     node_metadata => {"os_version" => "buster"},
     tags          => ['proxmox'],
   }
```

Once you understand the individual building blocks of `tags`,`repo` and `broker` a policy should help solidify how they fit together. Simply put a policy is the only actionable item of the lot. Its code references the tag to line up the matching node with the correct repo and broker.

In the examples above, we are using our two tags to create one dynamic policy and one static policy. The `install_debian_on_virtual` will run 100 times on as many VMs as we through it that are virtual. The `proxmox_install` will run once. The hostnames on the virtual machines will be static with `${id}` being interpolated into the current node number.

And with that we can now boot any new VM in proxmox right to the install screen. Further when I start my Intel NUC up it does the same.

![PXE](/images/posts/pxe.gif)

# Cloud-init Broker.

While its not required to use cloud-init, you might be surprised how under utilized the tool normally is. I use puppet to configure my machines and generally that means I want to keep any initial configuration minimum. However be careful to use the right tools at the right time. Puppet when it works, is great but what happens if you are troubleshooting puppet itself. Having a barebones base configuration with cloud-init sounds like a great plan. Given both puppet and the cloud-init broker can read from hiera, you don't even in practice have to worry about duplication.


So here is what a bare bones cloud-init configuration can look like

```yaml
 #cloud-config
 cloud-init:
   # Initial User Configuration
   groups:
     - homeops 
 
   users:
     - default
     - name: homeops 
       gecos: homeops.tech
       shell: /bin/bash
       primary_group: homeops
       sudo: ALL=(ALL) NOPASSWD:ALL
       groups: users, admin
       lock_passwd: false
       ssh_authorized_keys:
         - ssh-rsa AAAAB3Nz...
 
   # If this is set, 'root' will not be able to ssh in and they
   # will get a message to login instead as the above $user (ubuntu)
   disable_root: false
 
   # This will cause the set+update hostname module to not operate (if true)
   preserve_hostname: false
 
   rsyslog:
       remotes:
           log_serv: "@synology.homeops.tech:514"
 
   # Downloads the golang package
   packages:
     - jq
     - curl
     - git
     - qemu-guest-agent
   # Add apt repositories
   apt_sources:
     - source: deb http://apt.puppetlabs.com buster puppet6
       keyid: 'D6811ED3ADEEB8441AF5AA8F4528B6CD9E61EF26'
       filename: puppet6.list
 
   # Run 'apt-get update' on first boot
   apt_update: true
 
   puppet:
     install: true
     package_name: 'puppet-agent'
     conf_file: '/etc/puppetlabs/puppet/puppet.conf'
     csr_attributes_path: '/etc/puppetlabs/puppet/csr_attributes.yaml'
     conf:
       agent:
         server: "puppet.homeops.tech"
         certname: "%f"
     csr_attributes:
         custom_attributes:
             1.2.840.113549.1.9.7: 342thbjkt82094y0uthhor289jnqthpc2290
         extension_requests:
             pp_preshared_key: 342thbjkt82094y0uthhor289jnqthpc2290
 
   timezone: 'UTC'
 
   manage_resolv_conf: true
 
   resolv_conf:
     nameservers: ['192.168.53.60', '192.168.53.70']
     searchdomains:
       - homeops.tech
     domain: homeops.tech
     options:
       rotate: true
       timeout: 1
 
   datasource_list: [ ConfigDrive, None ]
   datasource:
     ConfigDrive:
       dsmode: local
```

So as you can see , this cloud-init handles the the basics of creating a backup account, and then bootstraps puppet. The `csr_attributes` can be use to then allow autosigning and classification of the node in puppet. I enable the ConfigDrive data source here as `proxmox` actually supports cloud-init out of the box. This allows an ISO image to contain extra node specific data such as the IP address of the node. I will be expierementing with using the SD card slow on my servers to bootstrap some of this local data from my servers in the future.

> Again a little hand wavy here with the csr_attributes as they can't be node specific. Thats v2 and can be fixed with cloud-drives and such.

# Cloud-init Broker

{% gist e6e8c674e192ca7fa6d2be7b69ef2998 install.erb %}

You can check out the [gist]({% gist 679c5fd0c32ac40c680d09d5b1aa5b63 Dockerfile %}) for the configuration yaml for the broker. However lets take a look at what the `install.erb` script looks like. This script is essentially the payload for the cloud-init process. It will create a cloud-config and then run the final boot steps for cloud init. Assuming everything was successful it will phone home to razor and mark the node as installed. This by the way showcases one of the powers of razor,
which is to interpolate ruby templates on files by adding metadata and node data dyanmically.



# Buster Task

{% gist a24a415a21c0cdb83622b282827fb10e preseed.erb %}

In my example I am installing debian buster. At the time of this writing the razor server rather predictably only had a "task" for wheezy. Updating for buster was a minor  pain however it also works with the cloud-init broker to enable a new systemd item in liue of the rc.local file. You can find a full buster task in [this gist](https://gist.github.com/acidprime/a24a415a21c0cdb83622b282827fb10e).

A few notes here on this preseed

```shell
<% if node.facts['is_virtual'] == true %>
d-i partman-auto/disk string /dev/sda
<% else %>
d-i partman-auto/disk string /dev/nvme0n1
<% end %>
```

The virtual machines I use in proxmox are scsi emulated storage controllers. The Intel NUC on the other hand is an M.2 Nvme.


```shell
sh -c 'set -- $(vgs --rows --noheadings | head -n 1); for vg in "$@"; do swapoff "/dev/$vg/swap"; vgremove -f "$vg"; done; set -- $(pvs --rows --noheadings | head -n 1); for pv in "$@"; do pvremove -f "$pv"; done'
```

This little script is used to remove the volume group as its name conflicts given I have static DHCP reservations. In other words the hostname is always the same in Netboot or Regular boot.



{% gist a24a415a21c0cdb83622b282827fb10e razor_systemd.erb %}

The new systemd item is simply a runner for the postinstall script that calls cloud-init

> Cloud-init seems well documented in some ways and not in others. My initial assumption was that I would not have to run these commands if I simply enabled the systemd items for cloud-init. However in testing that didn't work and so the script is essentially running cloud-init.
