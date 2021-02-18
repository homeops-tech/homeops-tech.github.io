---
layout: post
title:  "Managing DHCP with Puppet"
date:   2020-07-15
image: /images/posts/glass.png
tags: [puppet,isc-dhcpd,hiera,iot,how to,ha,raspberry pi,monit,systemd]
---

In my previous post on managing [DNS with Puppet](http://www.homeops.tech/2020/06/06/DNS-With-Puppet/) I walked you through the code to configure an internal DNS domain with puppet. In this post I will cover setting up a DHCP server that supports IP reservations and has failover configured. I use this with two raspberry pis that also provide my DNS. While DHCP might often be ran off your router, in a home lab you might want to run DHCP in a separate VLAN for testing things out like PXE booting.

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
            <td>dhcpd</td>
            <td>isc-dhcpd-4.4.1</td>
            <td>ubuntu</td>
        </tr>
        <tr>
            <td>ubuntu</td>
            <td>20.04</td>
            <td>server</td>
        </tr>
    </tbody>
</table>


> I run this all on a set of two raspberry pis'

  
  `Puppetfile`
  


```ruby
mod 'dhcp',
  :git => 'git@github.com:puppetlabs/puppetlabs-dhcp.git',
  :ref => 'b5925938188787faad99fdb52f294796e527a3d1'

mod 'nodejs',
  :git => 'https://github.com/voxpupuli/puppet-nodejs.git',
  :tag => 'v8.1.0'

mod 'staging',
  :git    => 'https://github.com/nanliu/puppet-staging.git',
  :commit => 'b466d93f8deb0ed4d9762a17c3c38f356aa833ee'
```


# Installing DHCPD

```puppet
  class { '::dhcp':
    nameservers   => ['192.168.53.60','192.168.53.70'],
    ntpservers    => ['time.homeops.tech'],
    interfaces    => ['eth0'],
    dnsdomain     => ['homeops.tech'],
    extra_config  => @("EOS")
      option space APC;
      option APC.cookie code 1 = string;
      class "vendor-id" {
           match option vendor-class-identifier;
      }
      subclass "vendor-id" "APC" {
           vendor-option-space APC;
           option APC.cookie "1APC";
      }
      set vendor-string = option vendor-class-identifier;
      if exists user-class and option user-class = "iPXE" {
          filename "bootstrap.ipxe";
      } else {
          filename "undionly.kpxe";
      }
      next-server $::ipaddress;
      | EOS
  }

```

Here we are installing isc-dhcpd , which is the most common dhcp deamon on embedded systems. If your wireless gear is based on openwrt or if your networking gear is based on linux you might have this service already running. Given I have DNS running on my raspberry pis it only makes sense to add DHCP. This allows us in this example to configure DHCP options like PXE boot servers and the APC cookie configuration on my batter backups.

# Setting up failover.

```puppet
 # Hiera call is needed here as they are two different servers
 class { 'dhcp::failover':
   role         => hiera('dhcp::failover::role'),
   peer_address => hiera('dhcp::failover::peer_address'),
 }

```

I have two rasperry pis and unlike DNS that has its own client side failure abilities DHCP requires a server side configuration for failure protection. This allows me to reboot,reimage or reconfigure my running DHCP servers without bumping Netflix offline. 

> in my experience DHCP failover works best with reservations. In the event of a failure the pool is cut in half , however those nodes with reservations will continue to receives their IPs. This keeps me honest in updating the yaml I use to manage this.

This hiera lookup is essentially just a per node configuration. In the example hierarchy we setup in configuring DNS we added this line:

```yaml
'nodes/%{::trusted.certname}'
```

I have two DHCP servers `hank.homeops.tech` and `dean.homeops.tech` So my yaml configuration is stored in my `data` directory as follows


```yaml
nodes/
├── dean.wallcity.org.yaml
├── hank.wallcity.org.yaml
└── puppet.wallcity.org.yaml
```

Hank is the primary so that servers configuration is:

```yaml
---
dhcp::failover::role: 'primary'
dhcp::failover::peer_address: '192.168.53.70'
```

Dean is secondary/failover node and its configuration is:

```yaml
---
dhcp::failover::role: 'secondary'
dhcp::failover::peer_address: '192.168.53.60'
```

While there are many ways of making this information dynamic, you likely want to keep things simple given these servers are some of the first needed to get things rolling in your lab.


## Configuring an IP Pool


```puppet
dhcp::pool{ 'homeops.tech':
  network => '192.168.53.0',
  mask    => '255.255.255.0',
  range   => ['192.168.53.165 192.168.53.200'],
  gateway => '192.168.53.1',
}
```

Here we create a range of IPs form `165` to `200`. The DHCP reservations should be outside this range.

> Keep in mind this pool is cut in half during failover.

## Load Static Reservations


```puppet
  $dhcp_hosts = hiera('dhcp_hosts')

  create_resources('dhcp::host',$dhcp_hosts)
```

Here we load in the DHCP hosts from hiera. This allows us (like DNS) to manage this with YAML files. In my case I create a file named `dhcp.yaml` in the hierarchy to contain these entries. This data can then be consumed by many other service , like icinga and razor-server.

> Note: I need to upgrade this for completeness to the .each syntax in Puppet.


Adding static entries is as simple as adding entries to the hash:

  
`dhcp.yaml`
  


```yaml
---
dhcp_hosts:
  aurora:
    mac: '14:B3:2F:0B:F1:15'
    ip: '192.168.53.16' 
  echo_dot:
    mac: '44:65:0D:49:F9:47'
    ip: '192.168.53.51'
```

## Monitoring with Monit

![Monit](/images/posts/monit.png)

```puppet
  class { 'monit':
    httpd          => true,
    httpd_address  => $::ipaddress,
    httpd_password => '$uper$secret',
  }

  monit::check { 'dhcpd':
    content => 'check process isc-dhcp-server with pidfile /run/dhcp-server/dhcpd.pid 
    start program = "/etc/init.d/isc-dhcp-server start"
    stop  program = "/etc/init.d/isc-dhcp-server stop"
    if failed host 127.0.0.1 port 67 type udp then alert
  ',
  }
```

Because of the criticality of this service I want to make sure it is always running, as we saw with DNS we can add monit. We can run this on the local box as well to have status page for the services.

# Installing Glass (DHCP Web interface)

```puppet
  class { '::nodejs':
    manage_package_repo       => true,
    nodejs_dev_package_ensure => 'present',
    npm_package_ensure        => 'present',
  }

  package { 'forever':
    ensure   => 'present',
    provider => 'npm',
  }

  vcsrepo { '/opt/glass-isc-dhcp':
    ensure   => present,
    provider => git,
    source   => 'https://github.com/Akkadius/glass-isc-dhcp.git',
    revision => '10102a670909f30f7a471837826cf863b18ba0b0',
    notify   => Exec['install'],
  }

  exec {'install':
    command =>  'chmod u+x ./bin/ -R && chmod u+x *.sh && mkdir -p logs && npm install',
    cwd     => '/opt/glass-isc-dhcp',
    refreshonly => true,
    path        => '/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
  }

  cron { 'glass_startup':
    ensure  => 'present',
    command => 'cd /opt/glass-isc-dhcp && /usr/local/bin/forever --minUptime 10000 --spinSleepTime 10000 -a -o ./logs/glass-process.log -e ./logs/glass-error.log ./bin/www',
    special => 'reboot',
    target  => 'root',
    user    => 'root',
    require => Package['forever'],
  }
```

In configuring the raspberry pis you might miss having the standard web interface that comes with most home routers version of DHCP. This can be helpful for finding MAC addresses on opaque machines, like smart speakers. I Have been using the Glass DHCP web interface for a while now and while not every feature is useful,given I'm managing the configuration with puppet, I find the overview page useful for viewing the lease files. There is no official installer for this tool but a little puppet code goes along way here.


# Optional: Enable PXE & TFTP Services

```puppet
package {'tftp':}

  file { '/var/lib/tftpboot':
    owner  => 'tftp',
    group  =>  'tftp',
    ensure => directory,
    mode   => '0755'
  }

  class { 'tftp':
    directory => '/var/lib/tftpboot',
    address   => $::ipaddress,
    options   => '--verbose --permissive --secure --timeout 120',
    inetd     =>  false,
  }

  # The iPXE kernel will fetch a bootstrap.ipxe file, that then aims a chain
  # loader at the Razor server to determine how to proceed.  For instance, new
  # machines are simply told to load a microkernel.
  # The bootstrap is static, but Razor likes to be the one to craft it.
  staging::file { 'bootstrap.ipxe':
    target      => '/var/lib/tftpboot/bootstrap.ipxe',
    source      => "http://razor.homeops.tech:8150/api/microkernel/bootstrap?nic_max=1&http_port=8150",
    curl_option => '--insecure',
    require     => [ File['/var/lib/tftpboot'] ],
  } ->

  tftp::file { 'bootstrap.ipxe':
    ensure => file,
    mode   =>  '0755',
    source => "/var/lib/tftpboot/bootstrap.ipxe",
  }

  # undionly.kpxe
  wget::fetch { 'http://boot.ipxe.org/undionly.kpxe':
    destination => "/var/lib/tftpboot/undionly.kpxe",
  } ->

  tftp::file { 'undionly.kpxe':
    ensure => file,
    mode   =>  '0755',
    source => "/var/lib/tftpboot/undionly.kpxe",
  }
```

While this not strictly needed in every environment, I find the inclusion of TFT and PXE to be generally connected to my DHCP ecosystem. 

```shell
...
      if exists user-class and option user-class = "iPXE" {
          filename "bootstrap.ipxe";
      } else {
          filename "undionly.kpxe";
      }
      next-server $::ipaddress;
...
```

As we saw earlier we setup the DHCP options to point to the needed files to PXE boot a `razor-server` implementation. 

> Note that you need to comment out this section unless you have setup razor-server. Watch here for a future razor article
