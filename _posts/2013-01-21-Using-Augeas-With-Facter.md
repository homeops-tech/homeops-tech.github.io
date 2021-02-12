---
layout: post
title:  "Using Augeas With Facter"
date:   2013-01-21
image: /images/posts/ruby.jpg
tags: [puppet,facter,augeas,devops]
---

Augeas is one of those little know but hugely powerful tools in the configuration management world. It allows a unified configuration language to view files in different formats. It does so with different lens'. The lens files are really useful when you use augeas as a library to parse and edit files.

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
    </tbody>
</table>

I recently needed to demo how to use augeas with facter. This needs to be optimised to not just be `Augeas::NONE` but it allows you to have augeas parse the file rather then ruby . This is useful in cutting down the code needed and also the development needed over time. Its also likely to handle corner cases better then just reading a file in line by line.


```ruby
Facter.add(:default_realm) do
  setcode do
    begin
      require 'augeas'
      aug = Augeas::open('/', nil, Augeas::NONE)
      default_realm = aug.get('/files/etc/krb5.conf/libdefaults/default_realm')
      aug.close
      default_realm
    rescue Exception
      Facter.debug('ruby-augeas not available')
    end
  end
end
```


