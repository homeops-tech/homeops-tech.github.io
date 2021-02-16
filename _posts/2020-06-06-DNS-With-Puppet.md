---
layout: post
title:  "Managing DNS with Puppet"
date:   2020-06-06
image: /images/posts/dns.jpg
tags: [puppet,bind,dns,hiera,iot,how to,ha,raspberry pi,monit]
---

One of the more important things in configuration management is DNS. In home labs we often don't have DNS out of box. Some folks do use pi-holes but often don't configure custom domains.
I often use static reservations for all my IOT devices. This means all devices on my network can use DNS names to configure each other. In this how-to I will show you how I use Puppet to setup an internal DNS domain.


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
            <td>ubuntu</td>
        </tr>
        <tr>
            <td>bind</td>
            <td>9</td>
            <td>ubuntu</td>
        </tr>
        <tr>
            <td>ubuntu</td>
            <td>20.04</td>
            <td>server</td>
        </tr>
    </tbody>
</table>


> I run this all on a set of raspberry pis'

```Puppetfile
mod 'dns',
  :git => 'git://github.com/amateo/puppet-dns.git',
  :commit => '15805a8a6577bea6b3dfab1d8951369c925b5e6a'

mod 'monit',
  :git => 'https://github.com/acidprime/puppet-monit.git',
  :commit => 'f94712677271ccab0fc990478e31a7f37cb9791d'

mod 'collectd',
  :git => 'git@github.com:pdxcat/puppet-module-collectd.git',
  :tag => 'v12.2.0'

mod 'hiera',
  :git => 'https://github.com/hunner/puppet-hiera.git'
  :tag => 'v2.1.1'

mod 'resolv_conf',
  :git => 'git@github.com:saz/puppet-resolv_conf.git',
  :tag => 'v3.0.5'
```

> Using a `monit` fork here as the original author doesn't update OS version quickly enough for me.

# Installing Bind 

```puppet

  $dns_domain = 'homeops.tech'


  class {'dns::server':
   enable_default_zones => false,
  }

  dns::server::options { '/etc/bind/named.conf.options':
    check_names_master     => 'fail',
    check_names_slave      => 'warn',
    dnssec_enable          => false,
    forwarders             => [ '205.171.3.65', '205.171.2.65' ],
    statistic_channel_ip   => '127.0.0.1',
    statistic_channel_port => 8053,
  }
```

The code above is an example of configuring bind9 without the default zones.
I'm setting this server up with forwarders as I find that media streaming and such seems to work best with the content distro networks when I use my ISP's DNS.

## Adding DNS Zones


```puppet
 dns::zone { $dns_domain:
    soa => $::fqdn,
    soa_email => "root.${::fqdn}",
    nameservers => [$::hostname],
  }

  # Reverse Zone
  dns::zone { '53.168.192.IN-ADDR.ARPA':
    soa => $::fqdn,
    soa_email => "root.${::fqdn}",
    nameservers => [$::hostname],
  }

  # Apex record e.g. homeops.tech vs www.homeops.tech
  dns::record { "apex_alias${dns_domain}":
    zone   => $dns_domain,
    host   => '@',
    record => 'A',
    data   => '192.168.53.222'
  }
```

Here we create a new DNS zone , and set the nameservers to the hostname of the machine.


> 192.168.22.222 is the address of a jekell server that runs the rendered for my website.

## Configuring records

Now that we have out DNS zones configured we can add record. While you could do this in code, I find the amount of DNS records over time mean something like yaml is better. Further it also means you can load this data set into other tools e.g. add automatic testing with icinga. Stay tuned for a future article on icingaweb2.


First lets setup Hiera. Hiera is a data ingestion engine builtin to puppet. If you just setup your Puppet Server using my [previous article](http://www.homeops.tech/2020/02/01/Provision-Puppetserver-with-Packer/) then you need to manage hieras configuration.

```puppet
  class { '::hiera':
    backends     => [
      'yaml',
    ],
    datadir      => '/etc/puppetlabs/code/environments/%{environment}/data',
    hierarchy    => [
      'nodes/%{::trusted.certname}',
      'environments/%{environment}',
      'dhcp',
      'dns',
      'common',
    ],
  }
```

> Bootstrapping puppet can have some chicken before the egg scenarios.

This is an example configuration you can apply to your Puppet Master to configure hiera with support for a file called `dns.yaml` in your `data` directory of your control repo.
The most important idea here is that hiera is using yaml files that are dynamically name e.g. `%{::trusted.certname}` or statically named e.g. `dns`. For our purposes we will mostly be using static entries for our DNS records.

Lets add code to pull DNS records in from hiera:


```puppet
  # Resource Defaults
  Dns::Record::A {
    zone => $dns_domain,
    ptr  => true,
  }

  Dns::Record::Cname {
    zone => $dns_domain,
  }

  hiera('a_records').each |$key, $value| {
    dns::record::a { $key:
      data => "${value['data']}",
    }
  }

  hiera('cname_records').each |$key, $value| {
    dns::record::cname { $key:
      data => "${value['data']}"
    }
  }
```


This code snippet will iterate over the keys in hiera. You can create more then A and CNAME but for my purposes, these and the automatically created PTR records normally are enough.
Lets look at what our dns.yaml file looks like in this configuration:

  
`dns.yaml`
  


```yaml
---
# DNS Records
cname_records:
  grafana:
    data: 'proxmox.homeops.tech'
  www:
    data: 'proxmox.homeops.tech'
a_records:
  proxmox:
    data: '192.168.53.38'
    tag: ['linux']
  alexa:
    data: '192.168.53.206'
    tag: ['iot']
```

The nice thing about this configuration is that to add DNS records you now simply edit the yaml. 

> You can safely ignore the `tag` key here, it will be used in future articles to classify these entries for monitoring.

## Installing useful tools

```puppet
  package {'dnstop':
    ensure => present,
  }
```

## Monitoring & Reporting

```puppet
  monit::check { 'bind9':
    content => 'check process named with pidfile /var/run/named/named.pid 
    start program = "/etc/init.d/named start"
    stop  program = "/etc/init.d/named stop"
    if failed host 127.0.0.1 port 53 type tcp protocol dns then restart
    if failed host 127.0.0.1 port 53 type udp protocol dns then restart
  ',
  }

```

While systemd is all the rage, I like pants and suspenders when it comes to critical services.


```puppet
class { 'collectd::plugin::bind':
  url    => 'http://localhost:8053/',
}
```


We can also load the metrics into collect/graphite/graphana. This is configured above in the options:

```puppet
...
   statistic_channel_ip   => '127.0.0.1',
   statistic_channel_port => 8053,
...
```

> You can checkout my article on setting up [Graphite/Grafana/Collectd here](http://www.homeops.tech/2020/05/20/Graphite-And-Grafana-With-Puppet/)

## Split DNS on VPN

```puppet
  dns::acl { 'trusted':
    ensure => present,
    data   => [ '192.168.53.221/32', '192.168.54.0/24', ]
  }
```

I use zerotier for my "vpn" in addition to swan. I need to allow these clients to access the DNS server. in this configuration I setup the hiera range of my subnet (vpn) and the .54 address space I use with zerotier. 


## Configure resolv.conf

```puppet
class { 'resolv_conf':
   nameservers => ['127.0.0.1', '192.168.53.60', '192.168.53.70'],
   domainname  => $dns_domain,
}
```

Once we have tested the DNS server we can set it to perform resolutions with itself.
