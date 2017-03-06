---
#Categories:
#- Development
#- Configuration Management
Description: ""
Tags:
- ansible 
date: 2017-03-06T13:06:32+01:00
title: Using Ansible to automate my Macbook setup 
draft: false
---

#### Introduction

I am soon going to get a new Macbook, and have been thinking about how to setup it quickly and easily.
There is [Boxen](https://github.com/boxen/boxen), and while it is awesome at what is does, it does
too much for my needs. A quick Google shows there is plenty of other tools and solutions out there
like `Boxen` that automate a Mac setup. While they all looked mostly good, I want something a little more
personal, so, let's reinvent the wheel :D

#### Configuration management

I noticed that `Boxen` is build on top of [Puppet](https://puppet.com/), nice, however I prefer
[Ansible](https://www.ansible.com/), which led me the idea of using Ansible to automate my Macbook setup.

First step, how to install applications. I normally use [Homebrew](https://brew.sh/) as much as possible to
install command line utilities. But normally install GUI apps either through the Mac store or from the
developer's website. Which will not be so nice to try and maintain through `Ansible`. Looking through brew
and [Cask](https://caskroom.github.io/) however I realized that I will be able to install almost everything
I want using just brew and git. awesome!

Browsing through the Ansible module's, I saw that Homebrew is [supported](http://docs.ansible.com/ansible/homebrew_module.html) :D
So no need to use `Ansible` command module to call brew! And in normal `Ansible` fashion, it's incredibly easy
to use.

Example snippet of how to install vim using the homebrew module : 

```
- name: install vim 
  homebrew:
    name: vim 
    state: latest
```

and using cask

```
- name: install chrome 
  homebrew_cask:
    name: google-chrome 
    state: installed 
```

How simple is that! So now we have all the building blocks to automate the setup of my macbook.

#### Setupmac

So this is what I build : 
https://github.com/daemonza/setupmac

Yes, I know, super original name :D

What is does is, it bootstraps getting `Ansible` by installing [pip](https://pypi.python.org/pypi/pip) with `easy_install` which in
turns then install `Ansible`. From there, the `Ansible` playbook runs a `setup` role against localhost that installs `homebrew` and update it.
It then installs the applications listed [here](https://raw.githubusercontent.com/daemonza/setupmac/master/roles/setup/vars/main.yml). 
Unfortunately two of the applications is not available through Homebrew, so I download those to my `$HOME/Downloads` directory. Luckily 
this is simple to do with `Ansible` :

Example snippet of how to download a app([Zwift](http://zwift.com/)) to $HOME/Downloads :

```
   general:
     local_home: "{{ lookup('env','HOME') }}"

   zwift:
     name: Zwift
     url: http://cdn.zwift.com/app/ZwiftOSX.dmg
     dest: "{{general.local_home}}/Downloads/ZwiftOSX.dmg"
```

The `Ansible` role then starts the configuration, which involves getting [oh-my-zsh](http://ohmyz.sh/) with the [theme](https://draculatheme.com/)
that I use, and [spacemacs](http://spacemacs.org/) from git, clones my [dotfiles](https://github.com/daemonza/dotfiles) from my github repository,
and copies the respective files to the correct locations.

To make it as easy as possible to install, you can execute the start script locally on your mac with curl as follows : 

```
curl -s https://raw.githubusercontent.com/daemonza/setupmac/master/start.sh | /bin/bash
```

The [script](https://raw.githubusercontent.com/daemonza/setupmac/master/start.sh) that bootstraps `ansible`

#### Conclusion

Ansible is awesome! It provides just the right level of power vs complexity to quickly enable you
to build useful automation. While `setupmac` is really setup for how I like my Macbook, I hope it will be useful
to someone who wants to do something similar.

