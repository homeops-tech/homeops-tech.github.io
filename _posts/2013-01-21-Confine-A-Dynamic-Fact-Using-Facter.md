---
layout: post
title:  "Confine a Dynamic Fact Using Facter"
date:   2013-01-21
image: /images/posts/ruby.jpg
tags: [puppet,facter,augeas,devops,windows]
---

While in training this week, a coworker was trying to figure out how to confine a fact using a another fact when using facter. This is quite easy using confine in the Facter.add block however when you add dynamic facts in the mix , that block is always executed after you wanted it confined. For instance if you have some yum based fact that dynamically generates many more facts and wanted to confine it only to RedHat, you can simply do the following:

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


```ruby
if Facter.osfamily == 'RedHat'
  IO.popen('yum list installed').readlines.collect do |line|
      array = line.split
      Facter.add("yum_#{array[0]}_version") do
          setcode do
              "#{array[1]}"
          end
      end
  end
end
```
