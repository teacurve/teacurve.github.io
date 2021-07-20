---
layout: post
title:  "Proxmox Automation Part 2: Automate Salt master creation with cloud-init and Terraform"
date:   2021-07-20 11:50:00 +0000
date:   2021-07-20 11:50:00 +0000
tags:   virtualization proxmox cloud-init terraform salt infrastructure automation
author: teacurve
toc: true
---

## Introduction

In [part 1](proxmox-automation-part-1-terraform) of this Proxmox Automation series, I showed how to configure a Ubuntu Server cloud-ready template, and create a new instance on Proxmox using cloud-init and Terraform.

The first post was the simple way - directly using Proxmox cloud-init functionality to customize the bare minimum parameters required to start a new instance; namely setting an authorized ssh key, and setting the network IP (or in our case, setting it to use DHCP).

However, the real power of cloud-init happens when you define your own configuration file. Proxmox can generate the [magical cd-rom image](https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html) for cloud-init based on the contents of this file, which lets you specify lots more parameters to customize the new instance.

In this post, I'll take advantage of this extra functionality to configure a salt master node.

SaltStack (Salt) is a configuration management system, like Puppet, Chef or Ansible. It is again part of the 'infrastructure as code' revolution, but focusses on software installation and configuration. This is as compared to something like Terraform, which concerns itself with creation and configuration of the (physical or virtual) hardware itself.

Simply put; I'll use Terraform to create VMs on Proxmox, and Salt to install software, and configure services.

Cloud-init does let you install software, and there is some overlap here. But cloud-init is really for initial-state configuration, Salt lets you manage the instance through its full lifecycle, and has more flexible and powerful tools to help you do that. Of course, the chicken-egg situation presents itself here; we need to somehow install Salt so that we can install further software. That's where cloud-init comes in; the plan is to install salt with cloud-init, then let Salt take care of the rest.

## Salt Basics 


### Why salt

There are four big names in configuration management;

- Chef
- Ansible
- Puppet
- Salt

Of these, I previously used ansible. I wanted to try out Salt because A) it's something new to try, and B) it claims to be faster, and Ansible has been quite slow for me (probably due to it operating over SSH).

There are multiple good articles comparing the different systems, all of which of course have their own pros and cons. After doing some research, I settled on Salt, for a bunch of reasons.

I do value the ability of Ansible to operate over SSH (agentless), but you have to create the image in the right way for it to happen (for example a sudo NOPASSWD user). Since we can automate  the creation of the Salt Master with Terraform anyway, it seemed like it was worth biting the bullet and trying it out. Additionally, it is possible to configure Salt to run [masterless](https://docs.saltproject.io/en/latest/topics/tutorials/quickstart.html){:target="_blank"}, if you so choose.

Additionally, I already use a single server (VM) to do most of my administration from - which I ssh into. Doing it from Making this a salt server  

### Salt Master/Minion

As previously mentioned, Salt can operate in a masterless mode, but we'll be automating the creation of a salt master node in this article.

The salt master node is responsible for handing out commands to all the minions. Communication happens over a custom (encrypted, ZeroMQ) protocol.

Salt execution modules are python modules that are run on a salt minion. There are many built in, many third party, and you can [write your own](https://www.linode.com/docs/guides/create-a-salt-execution-module/){:target="_blank"}, which makes this a really powerful configuration mechanism. Especially since it's not Ruby...

Salt states are configuration files that specify what a node should have installed on it. Salt will then 'make it so'. I think of this as saying 'I wish I had a server that looked like this' and then Salt makes the wish come true. This is also the same concept as Terraform - you describe how you wish the system to look, instead of describing the actions it should take (kind of).


## Terraform cloud-init revisited

We have previously used the special proxmox variables to configure basic-cloud-init settings. However, cloud-init works with a powerful config file that has many modules that allow for specifying all sorts of initial configuration, including package installation, and service configuration. This is all documented [here](https://cloudinit.readthedocs.io/en/latest/topics/modules.html){:target="_blank"}. Proxmox allows you to specify this file (which is passed to the vm via the magic cd-rom) as a 'snippet', which is what we shall use.

First, make a new terraform module to work from. You can copy the `tf` files from the previous post:

```
manager@linux-manager:~/terraform$ mkdir saltmaster-tf
manager@linux-manager:~/terraform$ cp terraform/main.tf saltmaster-tf/
manager@linux-manager:~/terraform$ cp terraform/vm.tf saltmaster-tf/
manager@linux-manager:~/terraform$ cd saltmaster-tf/
manager@linux-manager:~/terraform/saltmaster-tf$ terraform init
```

### Reproduce post1 with a cloud config file

I like to do things iteratively, so the first part of this will be just getting an instance up and running as in [Part 1](proxmox-automation-part-1-terraform), but passing the image customizations in a config file instead of using Proxmox's special variables. Namely, we want to configure an authorized SSH key (we'll continue to set the network ip as dhcp via the `ipconfig0` parameter).

#### Enable snippets

The Proxmox cloud-init config file works by having a local file on the proxmox server, which is a 'snippet'. You first need to enable snippets on the local directory of the proxmox server. Go to Database -> Storage -> Local -> Edit, click on Content and add 'Snippets'. You may also need to `mkdir snippets` in the `/var/lib/vz/` directory on the proxmox server.


#### Terraform module

For convenience, here is the unchanged `main.tf` file that contains the provider configuration:"

```
# PM_API_TOKEN_SECRET 
# PM_API_TOKEN_ID 
# (in .bashrc)

terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
      version = "2.7.4"
    }
  }
}


provider "proxmox" {
  pm_api_url = "https://192.168.1.10:8006/api2/json"
  pm_tls_insecure = "true"
  pm_log_enable = true
  pm_log_file = "terraform-plugin-proxmox.log"
  pm_log_levels = {
    _default = "debug"
    _capturelog = ""
  }
}
```

Here is the full `vm.tf` file:

```

locals {
  cloud_init_name = "ubuntu-server-focal-saltmaster-cloud-init"
  cloud_init_template_fn = "${local.cloud_init_name}.tpl"
  cloud_init_fn = "${local.cloud_init_name}.yml" 
  proxmox_ip = "192.168.1.10"
  template_file_init = templatefile("${path.module}/files/${local.cloud_init_template_fn}", {
    ssh_key = file("~/.ssh/id_rsa.pub")
  })
}

resource "local_file" "saltmaster-cloud-init-local" {
  content = local.template_file_init
  filename = "${path.module}/files/${local.cloud_init_fn}"
}


# this is the dirty bit; we need to be able to put the file to the proxmox server, 
# so we use SSH
resource "null_resource" "cloud_init_salt_master_config" {
  connection {
    type    = "ssh"
    user    = "root"
    private_key = file("~/.ssh/id_rsa")
    host    = "${local.proxmox_ip}"
  }

  provisioner "file" {
    source       = local_file.saltmaster-cloud-init-local.filename
    destination  = "/var/lib/vz/snippets/${local.cloud_init_fn}"
  }
}


resource "proxmox_vm_qemu" "salt-master-01" {

  depends_on = [
    null_resource.cloud_init_salt_master_config
  ]

  name = "salt-master-01"
  target_node = "hades"
  clone = "ubuntu-cloudready-template"
  os_type = "cloud-init"
  balloon = 1024
  boot = "order=scsi0"

  agent = 1

  cores = 2
  sockets = 1
  memory = 2560
  
  disk {  
      size            = "10G"
      type            = "scsi"
      storage         = "local-lvm"
  }
  
  vga {
    type = "std"
  }


  # Set the network
  network {
    model = "virtio"
    bridge = "vmbr0"
  }

  # We record which user to SSH in as
  ssh_user = "ubuntu"


  # The cloud init variables
  ipconfig0 = "ip=dhcp"
  cicustom = "user=local:snippets/${local.cloud_init_fn}"

  # Ignore changes to the network
  ## MAC address is generated on every apply, causing
  ## TF to think this needs to be rebuilt on every apply
  lifecycle {
      ignore_changes = [
          network
      ]
  }

  connection {
      type = "ssh"
      user = "${self.ssh_user}"
      private_key = "${file("~/.ssh/id_rsa")}"
      host = "${self.ssh_host}"
      port = "${self.ssh_port}"
  }

  provisioner "remote-exec" {
    inline = [  
      // here you will see the actual ip address        
      "/sbin/ip a"  
    ]      
  }

}

```

We also need a place to store the config file template, so:

`mkdir files`

And inside there, I have `ubuntu-server-focal-saltmaster-cloud-init.tpl`:

```
#cloud-config
users:
- name: ubuntu
  ssh_authorized_keys:
    - ${ssh_key}
```


#### Terraform module explained

Most of this is the same as before. At the start of the file, we have a locals block:

```
locals {
  cloud_init_name = "ubuntu-server-focal-saltmaster-cloud-init"
  cloud_init_template_fn = "${local.cloud_init_name}.tpl"
  cloud_init_fn = "${local.cloud_init_name}.yml" 
  proxmox_ip = "192.168.1.10"
  template_file_init = templatefile("${path.module}/files/${local.cloud_init_template_fn}", {
    ssh_key = file("~/.ssh/id_rsa.pub")
  })
}
```

Here I define some local variables, including the names of the files that I'm using. Additionally, I use the `templatefile` function to render my template cloud-init config. The config is very minimal, but shows how to use a templated argument (${ssh_key}). This is populated in the `templatefile` function wih the parameter passed to it, which is my ssh public key file. The result is stored in `template_file_init`.

Now, since we have to put the config file as a snippet on the proxmox server, I write the result of the template expansion into a local file with the next block:

```
resource "local_file" "saltmaster-cloud-init-local" {
  content = local.template_file_init
  filename = "${path.module}/files/${local.cloud_init_fn}"
}
```

This will then get put on the Proxmox server via SSH:

```
resource "null_resource" "cloud_init_salt_master_config" {
  connection {
    type    = "ssh"
    user    = "root"
    private_key = file("~/.ssh/id_rsa")
    host    = "${local.proxmox_ip}"
  }

  provisioner "file" {
    source       = local_file.saltmaster-cloud-init-local.filename
    destination  = "/var/lib/vz/snippets/${local.cloud_init_fn}"
  }
}
```

We use a null resource to do the SSH transfer with the file provisioner. We define the parameters for the SSH connection - and for this I have already put my public key into the proxmox authorized_keys file. You can do that with a one-liner:

```
ssh root@proxmoxip "mkdir ~/.ssh; echo \"`cat ~/.ssh/^C_rsa.pub`\" >> .ssh/authorized_keys"
```

Finally we declare our `proxmox_vm_qemu` resource as before. There are a few differences, but it's mostly the same. We call it salt-master-01, and set the name to be the same.

The first block:

```
depends_on = [
null_resource.cloud_init_salt_master_config
]
```

tells terraform to make sure the config file has been copied over before doing this resource, which is important.

The final change is to the cloud-init parameters:

```
# The cloud init variables
ipconfig0 = "ip=dhcp"
cicustom = "user=local:snippets/${local.cloud_init_fn}"
```

We no longer set the ssh_key here, but instead set the name of the cloud-init config file (which contains the ssh key)


#### Terraform apply

Finally, run `terraform plan` to make sure the file is valid and doesn't contain any typos. Once you've ensured it's correct you can run `terraform apply` to create the instance. Once complete, you should be able to ssh in via public key as before.

At this point, it might not seem like we have done much, since we have just replicated what we were able to do before, with added complexity. However, we've laid the foundations for much more powerful cloud-init configuration, and now we will take advantage of that to set up our salt master node.


## Create the Salt master

### Terraform module

The main difference here is the cloud-init configuration file; the terraform module at this point is almost exactly the same. The only difference is that we have more templated arguments to patch in to our more complicated cloud-init config. The full new `vm.tf` is:


```

locals {
  cloud_init_name = "ubuntu-server-focal-saltmaster-cloud-init"
  cloud_init_template_fn = "${local.cloud_init_name}.tpl"
  cloud_init_fn = "${local.cloud_init_name}.yml" 
  proxmox_ip = "192.168.1.10"
  salt_master_address = "192.168.1.18"
  template_file_init = templatefile("${path.module}/files/${local.cloud_init_template_fn}", {
    ssh_key = file("~/.ssh/id_rsa.pub")
    hostname = "salt-master-01"
    salt_master_address = "${local.salt_master_address}"
  })
}

resource "local_file" "saltmaster-cloud-init-local" {
  content = local.template_file_init
  filename = "${path.module}/files/${local.cloud_init_fn}"
}


# this is the dirty bit; we need to be able to put the file to the proxmox server, 
# so we use SSH
resource "null_resource" "cloud_init_salt_master_config" {
  connection {
    type    = "ssh"
    user    = "root"
    private_key = file("~/.ssh/id_rsa")
    host    = "${local.proxmox_ip}"
  }

  provisioner "file" {
    source       = local_file.saltmaster-cloud-init-local.filename
    destination  = "/var/lib/vz/snippets/${local.cloud_init_fn}"
  }
}


resource "proxmox_vm_qemu" "salt-master-01" {

  depends_on = [
    null_resource.cloud_init_salt_master_config
  ]

  name = "salt-master-01"
  target_node = "hades"
  clone = "ubuntu-cloudready-template"
  os_type = "cloud-init"
  balloon = 1024
  boot = "order=scsi0"

  agent = 1

  cores = 2
  sockets = 1
  memory = 2560
  
  disk {  
      size            = "10G"
      type            = "scsi"
      storage         = "local-lvm"
  }
  
  vga {
    type = "std"
  }


  # Set the network
  network {
    model = "virtio"
    bridge = "vmbr0"
  }

  # We record which user to SSH in as
  ssh_user = "teacurve"


  # The cloud init variables
  ipconfig0 = "ip=${local.salt_master_address}/24,gw=192.168.1.1"
  nameserver = "192.168.1.8"
  searchdomain = "teanet.local"
  cicustom = "user=local:snippets/${local.cloud_init_fn}"

  # Ignore changes to the network
  ## MAC address is generated on every apply, causing
  ## TF to think this needs to be rebuilt on every apply
  lifecycle {
      ignore_changes = [
          network
      ]
  }

  connection {
      type = "ssh"
      user = "${self.ssh_user}"
      private_key = "${file("~/.ssh/id_rsa")}"
      host = "${self.ssh_host}"
      port = "${self.ssh_port}"
  }

  provisioner "remote-exec" {
    inline = [  
      // here you will see the actual ip address        
      "/sbin/ip a"
    ]      
  }

}

```

And the full cloud-init config file is here:

```
#cloud-config
# Add/configure the main user
users:
- name: teacurve
  groups: sudo
  shell: /bin/bash
  sudo: ['ALL=(ALL) NOPASSWD:ALL']
  ssh_authorized_keys:
    - ${ssh_key}

# set the hostname
fqdn: ${hostname}.teanet.local

# set some qol stuff
locale: en_US.UTF-8
timezone: America/New_York

# update machine
package_update: true
package_upgrade: true

apt:
  sources:
    saltstack.list:
      source: "deb https://repo.saltproject.io/py3/ubuntu/20.04/amd64/latest focal main"
      filename: saltstack.list
      key: |
        -----BEGIN PGP PUBLIC KEY BLOCK-----
        Version: GnuPG v2

        mQENBFOpvpgBCADkP656H41i8fpplEEB8IeLhugyC2rTEwwSclb8tQNYtUiGdna9
        m38kb0OS2DDrEdtdQb2hWCnswxaAkUunb2qq18vd3dBvlnI+C4/xu5ksZZkRj+fW
        tArNR18V+2jkwcG26m8AxIrT+m4M6/bgnSfHTBtT5adNfVcTHqiT1JtCbQcXmwVw
        WbqS6v/LhcsBE//SHne4uBCK/GHxZHhQ5jz5h+3vWeV4gvxS3Xu6v1IlIpLDwUts
        kT1DumfynYnnZmWTGc6SYyIFXTPJLtnoWDb9OBdWgZxXfHEcBsKGha+bXO+m2tHA
        gNneN9i5f8oNxo5njrL8jkCckOpNpng18BKXABEBAAG0MlNhbHRTdGFjayBQYWNr
        YWdpbmcgVGVhbSA8cGFja2FnaW5nQHNhbHRzdGFjay5jb20+iQE4BBMBAgAiBQJT
        qb6YAhsDBgsJCAcDAgYVCAIJCgsEFgIDAQIeAQIXgAAKCRAOCKFJ3le/vhkqB/0Q
        WzELZf4d87WApzolLG+zpsJKtt/ueXL1W1KA7JILhXB1uyvVORt8uA9FjmE083o1
        yE66wCya7V8hjNn2lkLXboOUd1UTErlRg1GYbIt++VPscTxHxwpjDGxDB1/fiX2o
        nK5SEpuj4IeIPJVE/uLNAwZyfX8DArLVJ5h8lknwiHlQLGlnOu9ulEAejwAKt9CU
        4oYTszYM4xrbtjB/fR+mPnYh2fBoQO4d/NQiejIEyd9IEEMd/03AJQBuMux62tjA
        /NwvQ9eqNgLw9NisFNHRWtP4jhAOsshv1WW+zPzu3ozoO+lLHixUIz7fqRk38q8Q
        9oNR31KvrkSNrFbA3D89uQENBFOpvpgBCADJ79iH10AfAfpTBEQwa6vzUI3Eltqb
        9aZ0xbZV8V/8pnuU7rqM7Z+nJgldibFk4gFG2bHCG1C5aEH/FmcOMvTKDhJSFQUx
        uhgxttMArXm2c22OSy1hpsnVG68G32Nag/QFEJ++3hNnbyGZpHnPiYgej3FrerQJ
        zv456wIsxRDMvJ1NZQB3twoCqwapC6FJE2hukSdWB5yCYpWlZJXBKzlYz/gwD/Fr
        GL578WrLhKw3UvnJmlpqQaDKwmV2s7MsoZogC6wkHE92kGPG2GmoRD3ALjmCvN1E
        PsIsQGnwpcXsRpYVCoW7e2nW4wUf7IkFZ94yOCmUq6WreWI4NggRcFC5ABEBAAGJ
        AR8EGAECAAkFAlOpvpgCGwwACgkQDgihSd5Xv74/NggA08kEdBkiWWwJZUZEy7cK
        WWcgjnRuOHd4rPeT+vQbOWGu6x4bxuVf9aTiYkf7ZjVF2lPn97EXOEGFWPZeZbH4
        vdRFH9jMtP+rrLt6+3c9j0M8SIJYwBL1+CNpEC/BuHj/Ra/cmnG5ZNhYebm76h5f
        T9iPW9fFww36FzFka4VPlvA4oB7ebBtquFg3sdQNU/MmTVV4jPFWXxh4oRDDR+8N
        1bcPnbB11b5ary99F/mqr7RgQ+YFF0uKRE3SKa7a+6cIuHEZ7Za+zhPaQlzAOZlx
        fuBmScum8uQTrEF5+Um5zkwC7EXTdH1co/+/V/fpOtxIg4XO4kcugZefVm5ERfVS
        MA==
        =dtMN
        -----END PGP PUBLIC KEY BLOCK-----

packages:
  - salt-master
  - tmux
  - vim


salt_minion:
  conf: 
    master: ${salt_master_address}
    id: ${hostname}
```

### Terraform module explained

The only small changes to the terraform are that firstly we changed the ssh_user to be `teacurve` instead of `ubuntu`, since we're now pushing our public ssh key to the `teacurve` user instead. We've added a couple of new local variables useful for the new salt configuration.

Additionally, we change the ip configuration from dhcp to a static ip, since we'll need to know where this server is. Alternatively, we could rely on DNS. We use the same variable as in the config templatefile. Since we are setting a static IP, we also provide a nameserver and searchdomain which it would normally get via DHCP.

```
# The cloud init variables
ipconfig0 = "ip=${local.salt_master_address}/24,gw=192.168.1.1"
nameserver = "192.168.1.8"
searchdomain = "teanet.local"
cicustom = "user=local:snippets/${local.cloud_init_fn}"
```

Moving on to the new cloud-init config file; first it sets up my real user (`teacurve`), sets its ssh key, and gives it passwordless sudo:

```
users:
- name: teacurve
  groups: sudo
  shell: /bin/bash
  sudo: ['ALL=(ALL) NOPASSWD:ALL']
  ssh_authorized_keys:
    - ${ssh_key}
```

It should be noted that since I'm not pushing the ssh key for the `ubuntu` default user, I will not be able to SSH in as that user.

Next, we set the hostname of the new machine with:

```
fqdn: ${hostname}.teanet.local
```

This is a variable, so it is actually set by the templatefile function in the terraform script.

After that we set the locale/timezone and update the packages on the system. We also add a new repository source for apt to pull down the latest salt pacakges, complete with the public key required.

After that, we install some required packages:

```
packages:
  - salt-master
  - tmux
  - vim
```

`salt-master` will install the salt master on this node (which will be started automatically by default). I also install tmux and vim so that we can more easily interact with the server, though future packages should be installed by salt itself.

Finally, we want to install and configure the master to also be its own minion, so it can manage itself. `cloud-init` has special support for setting up a salt minion, documentation [here](https://cloudinit.readthedocs.io/en/latest/topics/modules.html#salt-minion){:target="_blank"}. The main thing we want to do is direct the minion to the master (in this case, itself).

The salt minion configuration is here:

```
salt_minion:
  conf: 
    master: ${salt_master_address}
    id: ${hostname}
```

Having the `salt_minion` module entry present will ensure the salt minion software is installed; we don't need to explicitly install it in `packages`. 

We set the master target (which happens to be the same host). We also explicitly set the minion_id as the hostname. Normally, salt will generate the minion_id from the hostname, and cache it in `/etc/salt/minion_id`. I found that it was picking up an old hostname ('ubuntu' - from the template), so I needed to explicitly set it.



### Instantiate the salt master

Run `terraform apply -auto-approve` to ignore the prompt, and once that's finished, you can SSH in with your own user, and see that salt has indeed been installed:


```
teacurve@salt-master-01:~$ apt list --installed | grep salt

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

salt-common/unknown,now 3003.1+ds-1 all [installed,automatic]
salt-master/unknown,now 3003.1+ds-1 all [installed]
salt-minion/unknown,now 3003.1+ds-1 all [installed]
``` 


## Create the salt minion

Make a new directory for a new terraform module for the minion. Copy the tf files from the salt master, and also copy the .tpl file, but rename it to saltminion. You should end up with a directory structure like this:

```
manager@linux-manager:~/terraform/saltminion-tf$ ls -R
.:
files  main.tf  vm.tf

./files:
ubuntu-server-focal-saltminion-cloud-init.tpl
```

### Terraform module

Here's the new `vm.tf`:

```

locals {
  cloud_init_name = "ubuntu-server-focal-saltminion-cloud-init"
  cloud_init_template_fn = "${local.cloud_init_name}.tpl"
  cloud_init_fn = "${local.cloud_init_name}.yml" 
  proxmox_ip = "192.168.1.10"
  salt_master_address = "192.168.1.18"
  template_file_init = templatefile("${path.module}/files/${local.cloud_init_template_fn}", {
    ssh_key = file("~/.ssh/id_rsa.pub")
    hostname = "salt-minion-01"
    salt_master_address = "${local.salt_master_address}"
  })
}

resource "local_file" "saltminion-cloud-init-local" {
  content = local.template_file_init
  filename = "${path.module}/files/${local.cloud_init_fn}"
}


# this is the dirty bit; we need to be able to put the file to the proxmox server, 
# so we use SSH
resource "null_resource" "cloud_init_salt_master_config" {
  connection {
    type    = "ssh"
    user    = "root"
    private_key = file("~/.ssh/id_rsa")
    host    = "${local.proxmox_ip}"
  }

  provisioner "file" {
    source       = local_file.saltminion-cloud-init-local.filename
    destination  = "/var/lib/vz/snippets/${local.cloud_init_fn}"
  }
}


resource "proxmox_vm_qemu" "salt-minion-01" {

  depends_on = [
    null_resource.cloud_init_salt_master_config
  ]

  name = "salt-minion-01"
  target_node = "hades"
  clone = "ubuntu-cloudready-template"
  os_type = "cloud-init"
  balloon = 1024
  boot = "order=scsi0"

  agent = 1

  cores = 2
  sockets = 1
  memory = 2560
  
  disk {  
      size            = "10G"
      type            = "scsi"
      storage         = "local-lvm"
  }
  
  vga {
    type = "std"
  }


  # Set the network
  network {
    model = "virtio"
    bridge = "vmbr0"
  }

  # We record which user to SSH in as
  ssh_user = "teacurve"


  # The cloud init variables
  ipconfig0 = "ip=dhcp"
  cicustom = "user=local:snippets/${local.cloud_init_fn}"

  # Ignore changes to the network
  ## MAC address is generated on every apply, causing
  ## TF to think this needs to be rebuilt on every apply
  lifecycle {
      ignore_changes = [
          network
      ]
  }

  connection {
      type = "ssh"
      user = "${self.ssh_user}"
      private_key = "${file("~/.ssh/id_rsa")}"
      host = "${self.ssh_host}"
      port = "${self.ssh_port}"
  }

  provisioner "remote-exec" {
    inline = [  
      // here you will see the actual ip address        
      "/sbin/ip a"      
    ]      
  }

}

```

And the new `ubuntu-server-focal-saltminion-cloud-init.tpl`:

```
#cloud-config
# Add/configure the main user
users:
- name: teacurve
  groups: sudo
  shell: /bin/bash
  sudo: ['ALL=(ALL) NOPASSWD:ALL']
  ssh_authorized_keys:
    - ${ssh_key}

# set the hostname
prefer_fqdn_over_hostname: true
fqdn: ${hostname}.teanet.local

# set some qol stuff
locale: en_US.UTF-8
timezone: America/New_York

# update machine
package_update: true
package_upgrade: true

apt:
  sources:
    saltstack.list:
      source: "deb https://repo.saltproject.io/py3/ubuntu/20.04/amd64/latest focal main"
      filename: saltstack.list
      key: |
        -----BEGIN PGP PUBLIC KEY BLOCK-----
        Version: GnuPG v2

        mQENBFOpvpgBCADkP656H41i8fpplEEB8IeLhugyC2rTEwwSclb8tQNYtUiGdna9
        m38kb0OS2DDrEdtdQb2hWCnswxaAkUunb2qq18vd3dBvlnI+C4/xu5ksZZkRj+fW
        tArNR18V+2jkwcG26m8AxIrT+m4M6/bgnSfHTBtT5adNfVcTHqiT1JtCbQcXmwVw
        WbqS6v/LhcsBE//SHne4uBCK/GHxZHhQ5jz5h+3vWeV4gvxS3Xu6v1IlIpLDwUts
        kT1DumfynYnnZmWTGc6SYyIFXTPJLtnoWDb9OBdWgZxXfHEcBsKGha+bXO+m2tHA
        gNneN9i5f8oNxo5njrL8jkCckOpNpng18BKXABEBAAG0MlNhbHRTdGFjayBQYWNr
        YWdpbmcgVGVhbSA8cGFja2FnaW5nQHNhbHRzdGFjay5jb20+iQE4BBMBAgAiBQJT
        qb6YAhsDBgsJCAcDAgYVCAIJCgsEFgIDAQIeAQIXgAAKCRAOCKFJ3le/vhkqB/0Q
        WzELZf4d87WApzolLG+zpsJKtt/ueXL1W1KA7JILhXB1uyvVORt8uA9FjmE083o1
        yE66wCya7V8hjNn2lkLXboOUd1UTErlRg1GYbIt++VPscTxHxwpjDGxDB1/fiX2o
        nK5SEpuj4IeIPJVE/uLNAwZyfX8DArLVJ5h8lknwiHlQLGlnOu9ulEAejwAKt9CU
        4oYTszYM4xrbtjB/fR+mPnYh2fBoQO4d/NQiejIEyd9IEEMd/03AJQBuMux62tjA
        /NwvQ9eqNgLw9NisFNHRWtP4jhAOsshv1WW+zPzu3ozoO+lLHixUIz7fqRk38q8Q
        9oNR31KvrkSNrFbA3D89uQENBFOpvpgBCADJ79iH10AfAfpTBEQwa6vzUI3Eltqb
        9aZ0xbZV8V/8pnuU7rqM7Z+nJgldibFk4gFG2bHCG1C5aEH/FmcOMvTKDhJSFQUx
        uhgxttMArXm2c22OSy1hpsnVG68G32Nag/QFEJ++3hNnbyGZpHnPiYgej3FrerQJ
        zv456wIsxRDMvJ1NZQB3twoCqwapC6FJE2hukSdWB5yCYpWlZJXBKzlYz/gwD/Fr
        GL578WrLhKw3UvnJmlpqQaDKwmV2s7MsoZogC6wkHE92kGPG2GmoRD3ALjmCvN1E
        PsIsQGnwpcXsRpYVCoW7e2nW4wUf7IkFZ94yOCmUq6WreWI4NggRcFC5ABEBAAGJ
        AR8EGAECAAkFAlOpvpgCGwwACgkQDgihSd5Xv74/NggA08kEdBkiWWwJZUZEy7cK
        WWcgjnRuOHd4rPeT+vQbOWGu6x4bxuVf9aTiYkf7ZjVF2lPn97EXOEGFWPZeZbH4
        vdRFH9jMtP+rrLt6+3c9j0M8SIJYwBL1+CNpEC/BuHj/Ra/cmnG5ZNhYebm76h5f
        T9iPW9fFww36FzFka4VPlvA4oB7ebBtquFg3sdQNU/MmTVV4jPFWXxh4oRDDR+8N
        1bcPnbB11b5ary99F/mqr7RgQ+YFF0uKRE3SKa7a+6cIuHEZ7Za+zhPaQlzAOZlx
        fuBmScum8uQTrEF5+Um5zkwC7EXTdH1co/+/V/fpOtxIg4XO4kcugZefVm5ERfVS
        MA==
        =dtMN
        -----END PGP PUBLIC KEY BLOCK-----

packages:
  - tmux
  - vim


salt_minion:
  conf: 
    master: ${salt_master_address}
    id: ${hostname}
```

### Terraform module explained

Apart from the name, the `vm.tf` is only changed in the networking area - we set it back to dhcp.

Additionally, the cloud config filename changed, but it is almost identical except that it doesn't install the `salt-master` package.

### Instantiate the salt minion

Now, `terraform init` to bring in the providers, and `terraform apply -auto-approve` to bring it up.


### Accept the keys on the salt master

At this stage we have salt master (`salt-master-01`), which is also a minion, and another minon `salt-minion-01`. If we ssh in to the salt master, we can (as root) check the keys pending authorization:

```
teacurve@salt-master-01:~$ sudo -i
root@salt-master-01:~# salt-key -F
Local Keys:
master.pem:  23:30:c2:cf:4e:73:a0:db:59:d5:2f:8e:f3:37:fe:a8:54:4b:33:f6:13:8a:05:c9:50:87:e4:ee:42:c5:c6:99
master.pub:  05:f2:ce:6f:22:62:a9:b0:0e:1b:2f:8b:8f:d2:ef:9a:fd:49:c3:0c:9e:a6:6b:4f:b1:38:ec:e2:12:a3:c8:ab
Unaccepted Keys:
salt-master-01:  40:7b:42:3e:c5:42:00:b4:81:8b:b1:75:bd:e7:68:10:9e:c3:a9:08:d0:bc:8f:31:ca:73:84:39:70:ea:fe:09
salt-minion-01:  d5:ee:ee:21:f8:c5:19:10:2b:e9:06:39:3c:87:57:1b:54:22:05:15:cb:48:97:e6:eb:47:98:56:2c:05:68:2b
```

As part of the security model, salt requires you to acknowledge newly connected minions before it will manage them. They are identified by their minion ID and their public key.

We can now accept these two pending keys with:

```
root@salt-master-01:~# salt-key -A
The following keys are going to be accepted:
Unaccepted Keys:
salt-master-01
salt-minion-01
Proceed? [n/Y] y
Key for minion salt-master-01 accepted.
Key for minion salt-minion-01 accepted.
```

And we can run:

```
root@salt-master-01:~# salt-run manage.status
down:
up:
    - salt-master-01
    - salt-minion-01
```

To check the nodes are up and available to be managed by salt.


## Issues

Unfortunately, proxmox requires us to put the cloud init file on the proxmox server, we cannot pass it through terraform. This means we need to scp it to the right place on the server. I'm not sure if that's a limitation of the Proxmox API or the telmate/proxmox provider.

## References

- [Beginners guide to Salt (linode)](https://www.linode.com/docs/guides/beginners-guide-to-salt/){:target="_blank"}
- [Salt in 10 minutes (SaltStack documentation)](https://docs.saltproject.io/en/latest/topics/tutorials/walkthrough.html){:target="_blank"}
- [Using Terraform and Cloud-Init to deploy and automatically monitor Proxmox instances (yetiops)](https://yetiops.net/posts/proxmox-terraform-cloudinit-saltstack-prometheus/){:target="_blank"}
- [Cloud config examples (cloudinit documentation)](https://cloudinit.readthedocs.io/en/latest/topics/examples.html){:target="_blank"}

## Troubleshooting

### Debugging cloud-init

The previous post showed how to debug Terraform and the telmate/proxmox proivder, but now we also need to be able to debug cloud-init, since we're giving it a more complicated configuration. 

By default, cloud-init logs to `/var/log/cloud-init-output.log`, and you can review this log after terraform has finished. You can also use cloud-init itself to parse this log more effectively:

```
/usr/bin/cloud-init analyze show
```

### Multipart warning in cloud-init error logs

If you get a warning like this:

```
__init__.py[WARNING]: Unhandled non-multipart (text/x-not-multipart) userdata: ...
```

Check to make sure you cloud-config file starts with the line:

```
#cloud-config
```

which is required.

## Conclusion

Thanks for reading, I hope you found this post useful. In this post I described how to automate the creation of a salt master node via Terraform and cloud-init. We take advantage of the more powerful way to customize cloud-init; by passing our own cloud-init config file that lets us specify more parameters for cloud-init to configure. 

Salt is the configuration management system I have chosen to use to automate my infrastructure. Whilst it is a server-based and agent-based solution, the system is trivial to set up. Additionally, salt can be alternatively run in a ['masterless'](https://docs.saltproject.io/en/latest/topics/tutorials/quickstart.html){:target="_blank"} configuration (where the minion is its own master).

In the next post I will describe how I used salt to configure a reproducable development environment, and some other infrastructure.
