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

#### Prerequsites

* Go
To start using it make sure you have [Go](https://golang.org) installed and your GOPATH and GOROOT
is configured. See the following [https://golang.org/doc/install](https://golang.org/doc/install) for
more information on how to setup Go.


* Terraform

#### Setting up infrastructure

Starting this blog post, I was very tempted to setup the infrastructure with salt-cloud using the 
[Vultr](https://docs.saltstack.com/en/latest/ref/clouds/all/salt.cloud.clouds.vultrpy.html) module.
However, you then need to have a `Salt` master, up and running. I wanted to get get a cluster up on 
`vultr` with the lowest setup barrier possible. Having used [Terraform](https://www.terraform.io/) 
before, and knowing how well it works, I decided to use it create a server on `Vultr`, and then 
bootstrap that server as a `Salt master`. 

With that said, let's create some resources. Unfornatly, `Terraform` does not come with `Vultr` 
as a build in [provider](https://www.terraform.io/docs/providers/index.html). There is however
a `Vultr` provder written by [RGL](https://github.com/rgl/terraform-provider-vultr).
Unfornatly, the repository is not maintained and more. So use the following one :
https://github.com/elricsfate/terraform-provider-vultr
and follow the instructions to install it.

As you might have noticed one of the dependecies we installed was the excellent `vultr` command line tool
written by [James Clonk](https://github.com/JamesClonk). We will use the tool to keep track of what's
created on Vultr.
For more information about the tool look [here](https://jamesclonk.github.io/vultr/)

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

Time, to setup your `Salt master`. Git clone my [Vultr Kubernetes stack](https://github.com/daemonza/vultr_kubernetes_stack) project :
```
git clone https://github.com/daemonza/vultr_kubernetes_stack.git
```
First thing we need to do is setup ssh keys to use. You can either copy exsisting ssh key to the `infrastructure/keys` directory, or
generate a new key pair and saving it there. The `saltmaster.tf` file expected a public_key with the name of `id_rsa.pub` if yours
is different, you need to edit `saltmaster.tf` pointing to the correct files.

Open the variables.tf.examples files, add your `Vultr` API key, and save the files as variables.tf
Open the `saltmaster.tf` file and edit as needed. I commented on what everything is, so tweak to your needs. For sake of this
blog post, I chose the cheapst server plan. However you will probarly want a better server for any type of production.

Now run `terraform plan` to see what will get created. You should get something simialer back :
```

+ vultr_server.k8s
    default_password:     "<computed>"
    ipv4_address:         "<computed>"
    ipv4_private_address: "<computed>"
    ipv6:                 "true"
    name:                 "master"
    os_id:                "167"
    plan_id:              "87"
    power_status:         "<computed>"
    private_networking:   "true"
    region_id:            "7"
    ssh_key_ids.#:        "<computed>"
    status:               "<computed>"

+ vultr_ssh_key.k8s
    name:       "master node ssh key"
    public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5HbVXsqFDbLWoQyP8K6kVgIl0u8IVgU4CL076P4ZPaoWe96eEU0vAHkALMfX4yfvcygK4fTyIFfZ3nwJANp/HolQNWYaTtB9UYIYIkEaEWJa4aEn5vYUILZNjY6/EHWXpH9TRzC00TaaIUdUf7AjFOSoIe7UGv8IDuC52UJO5ehRzb1O3rPSvXBJJtW4PaHCgXUB8C69dhVEpxpMLvZo6XHzbcTaSPuIWcdtYbVGAu9d94aaqBKmJHEBpyx+kB8A59MctDNJ6eqLmBF+2wQZZj/IhJy4wqKhmNiYpkjfNLO+aim9vQv9xB8YUxNdKhqrEeWOReThlTrfv6FY+rC4F wgillmer@Werners-MacBook-Pro.local"


Plan: 2 to add, 0 to change, 0 to destroy.
```

From this we can see a new server with the parameters we specified in saltmaster.tf will be created, and it will use the ssh key that
I specified.
If your  happy with what will be created run `terraform apply` which will create the above listed resources, and start the
`Salt master` service.

While `terraform apply` is running and busy created the server on `Vultr`, you can confirm it's being created by
using the `vultr` command line tool :
```
vultr servers
```
Expected output should look simialer to :
```
SUBID		STATUS	IP		NAME	OS		LOCATION	VCPU	RAM	DISK		BANDWIDTH     	COST
7298303		active	45.32.233.75	master	CentOS 7 x64	Amsterdam	1	512 MB	Virtual 125 GB	1000	      	5.00
```

Looking at the output of `terraform apply` your will see the following happens :
* key upload 
* server resource
* remote-exec over ssh 
* download bootstrap script with curl
* execute bootstrap script

The finaly last bit of output from the above `terraform apply` command should look simialer to :
```
vultr_server.k8s (remote-exec):  *  INFO: Running install_centos_git_post()
vultr_server.k8s (remote-exec):  *  INFO: Running install_centos_check_services()
vultr_server.k8s (remote-exec):  *  INFO: Running install_centos_restart_daemons()
vultr_server.k8s: Still creating... (5m50s elapsed)
vultr_server.k8s (remote-exec):  *  INFO: Running daemons_running()
vultr_server.k8s (remote-exec):  *  INFO: Salt installed!
vultr_server.k8s: Still creating... (6m0s elapsed)
vultr_server.k8s: Creation complete (ID: 7298382)

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.
```
We now have a `Salt master` on `Vultr` :-D
The whole process from the moment I executed `terraform apply` took roughly 6mins for me. I am based in Amsterdam
and deployed a server to `Vultr` Amsterdam region.

Let's verify. Get the IP of your `Vultr` server by running `vultr servers`.
```
SUBID		STATUS	IP		NAME	OS		LOCATION	VCPU	RAM		DISK		BANDWIDTH	COST
7371948		active	185.92.222.77	master	CentOS 7 x64	Amsterdam	1	1024 MB		Virtual 25 GB	1000		5.00
```
Using the IP, ssh to the server
using the key in the `infrastructure/keys` directory.
```
ssh -i keys/id_rsa root@108.61.198.37
```
When your on the server run `ps aux | grep salt-master`. You should have output simialer to :
```
[root@vultr ~]# ps aux | grep salt
root      9839  1.1  3.4 377372 35520 ?        Ss   10:51   0:01 /usr/bin/python2 /usr/bin/salt-master
root      9852  0.0  1.8 307440 18928 ?        S    10:51   0:00 /usr/bin/python2 /usr/bin/salt-master
root      9858  0.0  2.9 458748 29728 ?        Sl   10:51   0:00 /usr/bin/python2 /usr/bin/salt-master
root      9859  0.0  2.8 376820 29320 ?        S    10:51   0:00 /usr/bin/python2 /usr/bin/salt-master
root      9869  0.8  3.5 383488 36252 ?        S    10:51   0:00 /usr/bin/python2 /usr/bin/salt-master
root      9870  0.0  2.9 377372 29708 ?        S    10:51   0:00 /usr/bin/python2 /usr/bin/salt-master
root      9871  0.2  1.8 282904 19292 ?        Ss   10:51   0:00 /usr/bin/python2 /usr/bin/salt-minion
root      9872  0.0  2.9 754228 30244 ?        Sl   10:51   0:00 /usr/bin/python2 /usr/bin/salt-master
root      9873  1.5  3.8 467256 38856 ?        Sl   10:51   0:01 /usr/bin/python2 /usr/bin/salt-master
root      9874  1.5  3.8 467264 38860 ?        Sl   10:51   0:01 /usr/bin/python2 /usr/bin/salt-master
root      9875  1.5  3.8 467264 38864 ?        Sl   10:51   0:01 /usr/bin/python2 /usr/bin/salt-master
root      9882  1.5  3.8 467268 38868 ?        Sl   10:51   0:01 /usr/bin/python2 /usr/bin/salt-master
root      9883  1.5  3.8 467268 38872 ?        Sl   10:51   0:01 /usr/bin/python2 /usr/bin/salt-master
root      9905  0.5  3.0 450620 31472 ?        Sl   10:51   0:00 /usr/bin/python2 /usr/bin/salt-minion
root      9925  0.0  1.8 398972 18824 ?        S    10:51   0:00 /usr/bin/python2 /usr/bin/salt-minion
```

Nice! Everything is looking good, we can see that the salt master and a minion is up and running. However, if your not
happy with the resource that got created, e.g. you changed your mind about the server OS or vultr plan, you can destroy the
server with `terraform destroy -force`, edit the saltmaster.tf file and run `terraform apply` again to get the server with
new parameters back up again.
Expected output from a `terraform destroy -force` :
```
vultr_ssh_key.k8s: Refreshing state... (ID: 58c28d402e9e4)
vultr_server.k8s: Refreshing state... (ID: 7298303)
vultr_server.k8s: Destroying... (ID: 7298303)
vultr_server.k8s: Destruction complete
vultr_ssh_key.k8s: Destroying... (ID: 58c28d402e9e4)
vultr_ssh_key.k8s: Destruction complete

Destroy complete! Resources: 2 destroy
```

TODO - laat remote minions die master sien en keys accepted 




#### Provision Kubernetes 

Salt


Kyk na : https://ahus1.github.io/saltconsul-examples/tutorial.html


* start k8s container
* create vultr instances
* bootstrap one of the instances as a salt master and the other as minions
    ---- miskien kan die n split wees, en die bootstrapping van kubernetes
    ---- kan n tweede blog post wees?
* setup kubernetes


run salt master fileserver met git backend

https://docs.saltstack.com/en/getstarted/
https://docs.saltstack.com/en/latest/topics/tutorials/walkthrough.html

http://talks.caktusgroup.com/lightning-talks/2013/salt-master/#slide1
https://www.digitalocean.com/community/tutorials/saltstack-infrastructure-installing-the-salt-master
https://www.digitalocean.com/community/tutorials/saltstack-infrastructure-configuring-salt-cloud-to-spin-up-digitalocean-resources

Gebruik terraform om n vultr nodes op te kry, en gebruik dan salt om k8s te 
deploy op die nodes.
