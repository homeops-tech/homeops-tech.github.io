---
layout: post
title:  "Sync Unifi with Hiera"
date:   2020-12-3
image: /images/posts/unifi.jpg
tags: [uniFi,ubiquiti,wifi,monogodb,synology,puppet,hiera,docker]
---

In my previous [post](http://www.homeops.tech/2020/07/15/DHCP-With-Puppet/) I walked you through configuring DHCP with hiera yaml files. These entries allow you to reserve IPs for nodes. These IPs can then also be tied to DNS entries. However I always strive to integrate all services I run. One of those core services is a [Unifi Controller](https://evanmccann.net/blog/unifi-ecosystem-overview). Unifi's web interface is great for viewing current Wifi clients and also to troubleshoot and track down connectivity issues. However when a system doesn't
advertise its name, you are presented with a MAC address or in the case of some IOT devices a random string. Vendor lookups don't always help given the same chipset being used on multiple platforms. In this post I'll show you how to sync Unifi with a yaml file. 

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
            <td>ruby</td>
            <td>2.7.0p0</td>
            <td>ubuntu</td>
        </tr>
        <tr>
            <td>Unifi</td>
            <td>5.6.40</td>
            <td>ubuntu</td>
        </tr>
    </tbody>
</table>

> I have some older Unifi hardware mixed in with newer. This older version supports both.

![Mongo In Docker](/images/posts/mongo-docker.png)

I run my unifi controller on my synology. I split the controller (Web interface) and its database into separate containers. This is useful for upgrades and migrating the container to different hosts and platforms over time. This also makes it easier for me to connect to the mongo api and rewrite aliases that unifi uses. Lets look at an example of blank node.

![Unifi Blank](/images/posts/unifi-mac.png)

This node doesn't add its hostname to DHCP however I know what it is when looking though my DHCP reseverations. It is the dog treat dispenser that doubles as one of the security cameras for the Synology.

```yaml
   furbo:
     mac: '38:a2:8c:xx:xx:xx'
     ip: '192.168.53.167'
```

The DHCP reservation is working due to the puppet code, but the synology is unable to lookup this data on its own. While its possible to import this via CSVs I want this to happen automatically do this as I add new nodes. To do this first I write a script that will read a yaml file on disk and update the names based on the format above. 

# Push aliases to Unifi

  
`sync_unif_names.rb`
  

```ruby
#!/usr/bin/env ruby
 require 'mongo'
 require 'yaml'

 begin
   dhcp_hosts = YAML.load_file('/etc/dhcp.yaml')['dhcp_hosts']
   puts dhcp_hosts.inspect
   mac_to_name = Hash.new
   dhcp_hosts.each do |k,h|
     puts "Processing #{k}"
     mac_to_name[h['mac'].downcase] = k
   end
   db = Mongo::Client.new(['synology.homeops.tech:27117'],:database => 'ace')
   db.collections.each do |collection|
     if collection.name == 'user'
       collection.find().each do |doc|
         unless doc['mac'].nil?
           if mac_to_name.keys.include?(doc['mac'].downcase)
             puts "Found: #{mac_to_name[doc['mac']]}"
             puts doc['hostname'] unless doc['hostname'].nil?
             puts doc['oui'] unless doc['oui'].nil?
             collection.update_one({ _id: doc['_id'] }, { '$set' => { name: mac_to_name[doc['mac']] }})
           end
         end
       end
     end
   end
 rescue StandardError => err
   puts('Error: ')
   puts(err)
 end
```

I can copy my `dhcp.yaml` to `/etc` and run the script and it will sync the names.

![Unifi Name](/images/posts/unifi-name.png)


# Automating the Sync


While its nice to be able to update the Unifi interface en masse, Ideally we should be able to this automatically as the YAML file in hiera is updated.
We can do that with an `refresh` in Puppet. On the DHCP server (or any node) we apply the following code.


```puppet
   package {'mongo':
     ensure   => 'present',
     provider => 'gem',
   }

   file {'/usr/local/bin/sync_names.rb':
     ensure => 'present',
     owner  => 'root',
     mode   => '0700',
     source => 'puppet:///modules/profile/sync_names.rb',
     require => Package['monogo'],
   }

   $hosts = hiera('dhcp_hosts')

   file {'/etc/dhcp.yaml':
     ensure  => 'present',
     owner   => 'root',
     mode    => '0700',
     content => inline_template('<%= @{ dhcp_hosts: hosts }.to_yaml %>'),
     notify  => Exec['/usr/local/bin/sync_names.rb'],
   }

   exec { '/usr/local/bin/sync_names.rb':
     refreshonly => true,
     path        => $::path,
   }
```

Now when I add a new node to my DHCP server, the same puppet run will trigger the script to run. This happens because the local YAML file's content will change and puppet will issue a refresh notification to the exec.
