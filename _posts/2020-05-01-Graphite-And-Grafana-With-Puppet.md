---
layout: post
title:  "Graphite, Grafana and Collectd"
date:   2020-05-20
image: /images/posts/graphite.png
tags: [graphite,grafana,puppet,collectd]
---

I recently moved my existing esxi hypervisors to [ProxMox](https://www.proxmox.com/en/) to have more control then the esxi shell allowed. This allows me to run puppet natively on the hyper visor host. As part of this upgrade I installed an old SSD and ram into my 2011 Mac Mini. While doing so I broke the small traces that attach the cooling fan. After some research I found many people have done the same and just leave it fanless. This presented me with a great opportunity to graph the
the various temp sensors as I tried different cooling methods.


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
            <td>Debian</td>
        </tr>
        <tr>
            <td>Debian</td>
            <td>10.7</td>
            <td>buster</td>
        </tr>
    </tbody>
</table>

  
`Puppetfile`
  

```ruby
mod 'graphite',
  :git => 'git@github.com:echocat/puppet-graphite.git',
  :ref => 'v8.0.0'

mod 'grafana',
  :git => 'git@github.com:echocat/puppet-grafana.git',
  :ref => 'v1.2.0'

mod 'elasticsearch',
  :git => 'https://github.com/elasticsearch/puppet-elasticsearch.git',
  :ref => '7.0.0'

mod 'elastic_stack',
  :git => 'https://github.com/elastic/puppet-elastic-stack.git',
  :ref => '7.0.0'

mod 'apache',
  :git => 'https://github.com/puppetlabs/puppetlabs-apache',
  :ref => '1.5.0'

mod 'collectd',
  :git => 'git@github.com:pdxcat/puppet-module-collectd.git',
  :ref => 'v12.2.0'
```

> New to puppet? Checkout my how to on installing a [`puppetserver`](http://www.homeops.tech/2020/02/01/Provision-Puppetserver-with-Packer/)

We can install the packages needed for graphite to configure and store data.

## Installing Graphite

```puppet
package{ 'graphite-carbon':
  ensure => 'present',
}

class { 'graphite':
    gr_apache_24               => true,
    gr_web_cors_allow_from_all => true,
    secret_key                 => '$uper$ecret?',
    gr_web_server           => 'none',
    gr_web_user             => 'www-data',
    gr_web_group             => 'www-data',
    gr_disable_webapp_cache => true,
    gr_storage_schemas      => [
    {
      name       => 'carbon',
      pattern    => '^carbon\.',
      retentions => '1m:90d'
    },
    {
      name       => 'default',
      pattern    => '.*',
      retentions => '1m:30m,1m:1d,5m:2y'
    },
    {
      name       => 'collectd',
      pattern    => '^collectd.*',
      retentions => '1m:30m,1m:1d,5m:2y'
    }
  ],
  }
```

Here we can install Graphite and configure it with a storage scheme for collectd. The important bits here are `gr_web_server => 'none',` which allows us to configure graphite to be hosted with apache.

# Setting up Webserver

As both Graphite and Grafana can use apache lets start with a default installation.

```puppet
include '::apache'

```

```puppet

package {'libffi-dev':
  ensure => 'present',
}
```
## Setting up Graphite Vhost

Once apache is running we can setup a vhost for Graphite as follows

```puppet
apache::vhost { "graphite.${::domain}":
    port    => '80',
    docroot => '/opt/graphite/webapp',
    wsgi_application_group      => '%{GLOBAL}',
    wsgi_daemon_process         => 'graphite',
    wsgi_daemon_process_options => {
      processes          => '5',
      threads            => '5',
      display-name       => '%{GROUP}',
      inactivity-timeout => '120',
    },
    wsgi_import_script          => '/opt/graphite/conf/graphite.wsgi',
    wsgi_import_script_options  => {
      process-group     => 'graphite',
      application-group => '%{GLOBAL}'
    },
    aliases => [
         { 
           alias            => '/content/',
           path             => '/opt/graphite/webapp/content/',
         },
         { 
           alias            => '/static/',
           path             => '/opt/graphite/static/',
         },
   ],
    wsgi_process_group          => 'graphite',
    wsgi_script_aliases         => {
      '/' => '/opt/graphite/conf/graphite.wsgi'
    },
    headers => [
      'set Access-Control-Allow-Origin "*"',
      'set Access-Control-Allow-Methods "GET, OPTIONS, POST"',
      'set Access-Control-Allow-Headers "origin, authorization, accept"',
    ],
    directories => [
      {
        path => '/media/',
        order => 'deny,allow',
        allow => 'from all'
      },
      {
        path => '/opt/graphite/static/',
        order => 'deny,allow',
        allow => 'from all'
      },
      {
        path => '/opt/graphite/webapp/content/',
        order => 'deny,allow',
        allow => 'from all'
      },
    ]
  }
```

This is more or less the example in the modules README with a small tweak:

```
...
{ 
  alias            => '/static/',
  path             => '/opt/graphite/static/',
},
...
```

This undocumented line is the most important as otherwise you are greeted with a blank site with only headers.

```puppet
  file {"/opt/graphite/conf/graphite.wsgi":
    ensure => 'present',
    source => "/opt/graphite/conf/graphite.wsgi.example"
  }
```

We now can copy the example .wsgi file


## Setup

We now need to init the graphite django configuration. I should show you how to do this with an exec here, but honestly I often do this manually on first build with the following commands

```shell
sudo -H PYTHONPATH=/opt/graphite/webapp django-admin.py migrate --settings=graphite.settings --run-syncdb
sudo -H PYTHONPATH=/opt/graphite/webapp django-admin.py createsuperuser --settings=graphite.settings
sudo -H PYTHONPATH=/opt/graphite/webapp django-admin.py collectstatic --noinput --settings=graphite.settings
```


### Installing ElasticSearch 

To store the graphs in grafana we need an elasticsearch deployment. I actually don't use this as I normally export the json and have puppet manage it so it can be redeployed as will. The following however is an example of getting elasticsearch up and running.

```puppet
class {'elasticsearch':
  config       => {
    'cluster.name'           => 'elk',
    'node.name'              => $::fqdn,
    'network.host'           => '0.0.0.0',
    'discovery.type'         => 'single-node',
    'http.cors.allow-origin' => '*',
    'http.cors.enabled'      => 'true',
    'xpack.security.enabled' => 'false',
    'xpack.ml.enabled'       => 'false',
    'node.master' => 'true',
    'node.data' => true,
    'node.ingest' => true,
    'action.auto_create_index' => true,
  },
  restart_on_change => true,
  manage_repo  => true,
  init_defaults => {
      'ES_USER' => 'elasticsearch',
      'ES_GROUP' => 'elasticsearch',
      'MAX_OPEN_FILES' => '65535',
      'MAX_LOCKED_MEMORY' => 'unlimited',
      'JAVA_HOME' => '/usr',
      },
}

es_instance_conn_validator { 'elk' :
  server => 'localhost',
  port   => '9200',
}
```

I use this elasticsearch for an ELK stack (look forward a future article!) so thats why I call it elk here.



## Installing Grafana

```puppet
class {'grafana':
  graphite_host      => "graphite.${::domain}",
  elasticsearch_host => "localhost",
  elasticsearch_port => 9200,
}
```


> I'm using an older version of grafana as I have some older dashboards I don't want to update.

### Dashboards

```puppet
# Collectd Dashboard
file {'/opt/grafana/app/dashboards/':
  ensure  =>  directory,
  source  => 'puppet:///modules/profile/grafana/',
  recurse => true,
}
```

As mentioned, while its neat that grafana can store its data in ElasticSearch. I don't actually use that. When I build a dashboard I export it and save it in the `profile/files/grafana` directory of my module. This the is recusively copied into grafana's web directory. This means when I build or rebuild my grafana deployments I don't have to import elasticsearch data. It also means I can use templates to build dynamic dashboards that are copied in to `/opt/grafana/app/dashboards/`. These are then available using the following URL e.g. for `collectd.json` : `http://grafana.homeops.tech/#/dashboard/file/collectd.json`


## Setting up collectd

As I mentioned I wanted to monitor the temperature of the 2011 mac mini server I use as the Proxmox virtualisation box. Its my backup box and so I wanted to see if any of the USB fans I had laying around the house would actually work to cool it. To start graphing the temperature I needed to use the `sensors` package on debian/proxmox.

This is as simple as appling this configuration to that node:

```puppet
class {'collectd::plugin::sensors':
  manage_package => true,
}
```

This installs the sensors package. I also for good measure (and maybe required) ran the following to detect all sensors on the system.

```shell
sudo sensors-detect
```

![Sensors](/images/posts/sensors.png)

I was pleasantly surprised to find most supported by the package.

> While collectd doesn't resolve them , a long term plan is to rewrite the json using [this table](https://logi.wiki/index.php/SMC_Sensor_Codes). 

![Graphite Config](/images/posts/graphite-config.png)

Create the dashboard is simple as creating a dynamic entry for the node as show above.

![SadMini](/images/posts/sad-mini.jpg)

As luck would have it the only USB fans I had was a gift that someone gave me years ago. It was a LED fan that I had since lost the windows-only install CD for. _Given the state of the mini it seemed appropriate_.

![USBFan](/images/posts/usb_fan.png) 

Interestingly enough even at high load the fan managed to cool the SMC temp sensors ( and the CPU sensors) below the high/crit values.

Suffice to say I have replacement server (intel NUC) on order , however I may keep this mini around given its working. Proxmox supports a cluster with my APC PDU where it would only power on in the event of failure.
  
I'll write that up if I get it working.
