---
layout: post
title:  "Configure Code Manager via Ruby"
date:   2020-03-01
image: /images/posts/ruby.jpg
tags: [puppet,devops,ruby,puppet enterprise]
---

In my previous post about [automating Gitlab with python](http://www.homeops.tech/2020/01/01/Disable-Branch-Protection-Python/) I  showed you how you can add a webhook to Gitlabs repos dynamically. The other side of this configuration in Puppet Enterprise is configuring r10k to point towards that repo. While Gitlab has a nice python library the Puppet Enterprise classifier libraries are written in Ruby. In this quick how to I will show you the code required to configure Puppet Enterprise's
code manager. I actually was the one who wrote this code in Puppet enterprise back in the day and reason was all about automating your PE installations end to end. 

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
            <td>Puppet Enterprise</td>
            <td>2019</td>
            <td>Centos</td>
        </tr>
        <tr>
            <td>Centos</td>
            <td>7.9.2009</td>
            <td>Centos</td>
        </tr>
    </tbody>
</table>

# Requirements

This script requires the `puppetclassify` gem which can be installed via

```ruby
gem install puppetclassify

```

This gem is an API wrapper around the rest calls used with the Puppet Enterprise classifier.

Once installed you can modify this script to add your own Github/Gitlab/etc control repo URL.


  `code_manager_config.rb`
  

```ruby
#!/usr/bin/env ruby
require 'puppetclassify'

OpenSSL::SSL::VERIFY_PEER = OpenSSL::SSL::VERIFY_NONE


@classifier_url =  "https://#{ARGV[0]}:4433/classifier-api"

def load_classifier()
  auth_info = {
    'ca_certificate_path' => '/dev/null',
    'token'               => ARGV[1],
  }
  unless @classifier
    @classifier = PuppetClassify.new(@classifier_url, auth_info)
  end
end

def update_pe_master_r10k_remote()
  load_classifier
  groups    = @classifier.groups
  pe_master = groups.get_groups.select { |group| group['name'] == 'PE Master'}.first
  classes   = pe_master['classes']

  puppet_enterprise_profile_master = classes['puppet_enterprise::profile::master']

  puppet_enterprise_profile_master.update(
    puppet_enterprise_profile_master.merge(
      'code_manager_auto_configure' => true,
      'r10k_remote'      => 'git@gitlab.homeops.tech:homeops-tech/control-repo.git',
      'r10k_private_key' => '/etc/puppetlabs/puppetserver/ssh/id-control_repo.rsa',
      'replication_mode' => 'none'
    )
  )
  # I feel like this composition is overkill if this is truly a delta
  pe_master['classes']['puppet_enterprise::profile::master'] = puppet_enterprise_profile_master
  groups.update_group(pe_master)
end

update_pe_master_r10k_remote

```

You can run this script with the first argument being the node classifier and the second being an RBAC Token. See [automating Gitlab with python](http://www.homeops.tech/2020/01/01/Disable-Branch-Protection-Python/) for an example of using the RBAC API via Python. (it does not currently have a ruby gem)

> Note: that you need to ensure the Puppet server has the private key deployed at the location specified.
> Also: Sorry of the use of `master` here, its what the Puppet classes are named unfortunately in PE
