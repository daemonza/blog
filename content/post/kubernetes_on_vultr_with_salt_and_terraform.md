---
Description: ""
Tags:
- terraform
- saltstack
- kubernetes
- golang
date: 2017-03-06T15:57:14+01:00
title: kubernetes_on_vultr_with_salt_and_terraform
draft: true
---

#### Introduction

Create vultr resources, en deploy kubernetes na dit. n follow post
kan dalk wees hoe om dan Deis workflow na die toe te deploy.

#### Setting up infrastructure

Starting this blog post, I was very tempted to setup the infrastructure with salt-cloud using the 
[Vultr](https://docs.saltstack.com/en/latest/ref/clouds/all/salt.cloud.clouds.vultrpy.html) module.
However, you then need to have a `Salt` master, up and running. I wanted to get get a cluster up on 
`vultr` with the lowest setup barrier possible. Having used [Terraform](https://www.terraform.io/) 
before, and knowing how well it works, I decided to use it create a server on `Vultr`, and then 
bootstrap that server as a `Salt master`. 

With that said, let's create some resources. Unfornatly, `Terraform` does not come with `Vultr` 
as a build in [provider](https://www.terraform.io/docs/providers/index.html). There is however
a `Vultr` provder written by [RGL](https://github.com/rgl/terraform-provider-vultr), which is 
what we will be using.

To start using it make sure you have [Go](https://golang.org) installed and your GOPATH and GOROOT
is configured. See the following [https://golang.org/doc/install](https://golang.org/doc/install) for
more information on how to setup Go.

With Go setup, let's setup `rgl's` vultr provider :
```
mkdir -p $GOPATH/src/github.com/rgl/terraform-provider-vultr
git clone https://github.com/rgl/terraform-provider-vultr $GOPATH/src/github.com/rgl/terraform-provider-vultr
export PATH=$PATH:$GOPATH/bin
```

Now we need to get it's dependencies :

```
go get github.com/hashicorp/terraform
go get github.com/JamesClonk/vultr
```

Build and install the provider :
```
cd $GOPATHsrc/github.com/rgl/terraform-provider-vultr
go build
go test
cp terraform-provider-vultr* $GOPATH/bin
```

For more information about the vultr provider look [here](https://github.com/rgl/terraform-provider-vultr)

As you might have noticed one of the dependecies we installed was the excellent `vultr` command line tool
written by [James Clonk](https://github.com/JamesClonk). For more information about the tool look [here](https://jamesclonk.github.io/vultr/)

Let's confirm what we currently have running on `Vultr`.
Set your `Vultr` API key :
```
export VULTR_API_KEY=YOURVULTRAPIKEY
```
and run 
```
vultr servers
```

I got a empty result back as I have nothing currently running on `Vultr`. You might also get a message back
saying `Your IP is not authorized to use this API key`. You will then have to go to https://my.vultr.com/settings/#settingsapi
and add your IP with subnet size to the white list.


----------
Kyk na : https://ahus1.github.io/saltconsul-examples/tutorial.html


* start k8s container
* create vultr instances
* bootstrap one of the instances as a salt master and the other as minions
    ---- miskien kan die n split wees, en die bootstrapping van kubernetes
    ---- kan n tweede blog post wees?
* setup kubernetes

#### Salt

run salt master fileserver met git backend

https://docs.saltstack.com/en/getstarted/
https://docs.saltstack.com/en/latest/topics/tutorials/walkthrough.html

http://talks.caktusgroup.com/lightning-talks/2013/salt-master/#slide1
https://www.digitalocean.com/community/tutorials/saltstack-infrastructure-installing-the-salt-master
https://www.digitalocean.com/community/tutorials/saltstack-infrastructure-configuring-salt-cloud-to-spin-up-digitalocean-resources

Gebruik terraform om n vultr nodes op te kry, en gebruik dan salt om k8s te 
deploy op die nodes.
