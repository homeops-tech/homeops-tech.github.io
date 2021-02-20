---
layout: post
title:  "Docker SSH Pass through links"
date:   2020-01-15
image: /images/posts/docker.jpg
tags: [puppet,devops,docker,vagrant,virtualbox,ubuntu,gdm,ssh,how to]
---

I have always liked links. When I used to make Cocoa Apps for enterprise customers I liked adding things like Webkit to load cool HTML links and work with custom URLs. There is something about making something pop-up on screen from an otherwise static application that seems fancy to me. Sometimes I need to setup an environment (often linux) that needs a user to interact with a docker container. While docker exec is very similar to ssh, it lacks a
URL structure to integrate a website and docker together. In this how-to I will show you how to add `ssh://` links to Ubuntu and pass through those connections to docker.

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
        <tr>
            <td>docker</td>
            <td>18.0.9</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>vagrant</td>
            <td>2.2.14</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>virtualbox</td>
            <td>6.1.16</td>
            <td>Ubuntu</td>
        </tr>
        <tr>
            <td>Ubuntu</td>
            <td>18.04</td>
            <td>Ubuntu</td>
        </tr>
    </tbody>
</table>

I'll show how to do this with  Ubuntu here but the user side of this is essentially platform agnostic. The first thing to understand is that when a user logs into ssh they have a shell thats executed. You can configure that shell to be a wrapper script.
Here is an example that drops a user into a docker shell

  
`/usr/local/bin/scenario1.sh"`
  

```shell
#!/bin/bash
/usr/bin/docker exec -it scenario1 bash
```

This exec will create a new shell in a running container. However if you want to dynamically spin up or start the container you can do that via something like the `AuthorizedKeysCommand` directive in the sshd_config. This directive is meant to dynamically return a users public key when using something like ldap to store the keys. However you can run any arbitrary commands such as running the container


  
`/usr/local/bin/authorized_key_command`
   

```shell
  #!/bin/bash
  [ $# -ne 1 ] && { echo "Usage: $0 userid" >&2; exit 1; }
  /usr/bin/docker start $1 &>/dev/null
  cat /etc/authorized_keys 
```

> The first argument to this script is the username attempting to authenticate. This example assumes the container is named the same.

## Enabling SSH Links in Ubuntu (GDM)

Unlike macOS which ships with `ssh://` urls out of the box, this handler application is needed for Ubuntu. This simple wrapper is just taking the url and stripping the progID off the begining. It then passes it to a new shell. The result is the command `ssh foo@bar` instead of `ssh ssh://foo@bar`. This is ripe for arbitrary code execution so I suggest you only set this up for controller environments or do more validation here.

```shell
#Write code for new handler
cat << EOF > ~/.local/share/applications/ssh.desktop
[Desktop Entry]
Version=1.0
Name=SSH Launcher
Exec=bash -c '(URL="%U" ; ssh "\${URL##ssh://}"); bash'
Terminal=true
Type=Application
Icon=utilities-terminal
EOF

#Apply New handler for SSH
xdg-mime default ssh.desktop x-scheme-handler/ssh

#Confirm new handler has been applied
xdg-mime query default x-scheme-handler/ssh
```

## Putting it all together in vagrant 

![Anitmated Gif](/images/posts/ssh_links.gif)

{% gist 1778440802dcc250b9560be55fff5f44 example.html %}

I used a vagrant file here for testing, you can use it via the following commands.

> Note I tested with vagrant and virtualbox, see your OS's results on installing them.

```shell
git clone git@gist.github.com:1778440802dcc250b9560be55fff5f44.git docker_ssh
cd docker_ssh
vagrant up
```

> Note: This box is from the UK, you need to change the region if you want to type special commands like '@' on a US keyboard.

## Configuring GDM & Docker

I have included some example scripts that will configure the system via Puppet. The gist of these scripts is that they will configure users that have pass through scripts as shells. These pass through scripts will log them into docker via `docker exec`. This along with the URL handler allows the user to simply click on the right link to get the right container. It should work with any app that works with the xdg style apis e.g. from the commandline `xdg-open ssh://scenario1@localhost`. I have
included an example HTML file with the links.

1. Login to the vagrant box (password `vagrant`)
2. Open a terminal and type `cd /vagrant`
3. As the current user (vagrant) type `./setup.sh`
4. Open the `example.html` and click on the links.

This will install Puppet and run the configuration (with sudo) as needed for all the global configurations and the per-user configurations.

This example code configures a set of 3 users that have corrisponding docker containers and pass through shell scripts.
 
  
`users.pp`
  

{% gist 1778440802dcc250b9560be55fff5f44 users.pp %}

Once the users are setup you can configure the keys and the `AuthorizedKeysCommand` script mentioned above.

  
`ssh.pp`
  

{% gist 1778440802dcc250b9560be55fff5f44 ssh.pp %}

> You could fully automate this [setup.sh](https://gist.githubusercontent.com/acidprime/1778440802dcc250b9560be55fff5f44/raw/2de97e9899474a5cfe379692d454a26dade4f51d/setup.sh) by adding some of these scripts to vagrant provisioners however that wasn't my need in using this vagrant box.
