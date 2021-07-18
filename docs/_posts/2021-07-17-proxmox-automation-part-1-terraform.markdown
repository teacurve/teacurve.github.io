---
layout: post
title:  "Proxmox Automation Part 1: Terraform with Proxmox and cloud-init"
date:   2021-07-17 21:10:00 +0000
tags:   virtualization proxmox cloud-init terraform infrastructure automation
author: teacurve
toc: true
---

## Introduction

[Terraform](https://www.terraform.io/){:target="_blank"} is an infrastructure-as-code system that allows you to define an environment in a text file, and repeatably bring that environment to life on a variety of platforms (AWS, Azure, etc). 

I've tried out several different virtualization providers, but recently I have been running [Proxmox](https://pve.proxmox.com/wiki/Main_Page){:target="_blank"} (based on QEMU/KVM) in my home lab. I think Proxmox is fantastic for a homelab; it has a LOT of really advanced features (you can send a vm between clustered nodes, and it actually works), a usable web interface, and even a REST API. For enterprise use it's a little rough around the edges (at least the community version is), but for me it's great.

I don't really do much 'devops' but I do like repeatable environments. Normally I will just manually create VMs for research (or hosting services at home), but I document here how I got Terraform working with Proxmox to automate the creation of a `cloud-init` Ubuntu Server VM.


## Overview

Just so we're on the same page, here are the main components:

- Terraform - An infrastructure-as-code system
- Proxmox - A virtualization server (like ESXi)
- cloud-init - A standardized way of customizing cloud instances (basically how we specify some things about a new instance from a template)

In our case, we'll use Ubuntu as the OS for the new instance. I also happened to use Ubuntu as the machine from which I used Terraform, but Terraform runs on other operating systems, including Windows.

I have looked at various tutorials and sources of documentation, and provided a list in [References](#references) of some that might be helpful. I've also compiled a list of issues that I ran in to in the [Troubleshooting](#troubleshooting) Section, which you should check if you have problems following.



## Details



### Step 1 - Create a cloud-init template

There are essentially two ways of doing this. The first is to create a 'golden image' template of the OS you want, configure it how you like, and then convert to a template. Not only is this laborious for multiple templates, but you can run into issues if you do not prep the image correctly.


#### Get the image

Luckily, since everyone uses the cloud now, many operating systems provide 'cloud-ready' images for us to use. `cloud-init` is the standard way of doing this. In my case, I wanted a Ubuntu Server template, and they do provide cloud-ready images, which you can browse [here](https://cloud-images.ubuntu.co){:target="_blank"}. So, on the Proxmox server, download the chosen image:

```
wget wget https://cloud-images.ubuntu.com/focal/current/focal-server=cloudimg-amd64.img
```

#### Prep the image

Since proxmox is KVM/QEMU based, it's very helpful to have the QEMU guest agent installed in the image. In fact, I found not having it caused issues with the telmate/proxmox provider. Of course, the Proxmox wiki has a page on it [here](https://pve.proxmox.com/wiki/Qemu-guest-agent){:target="_blank"}. 

There's a really cool project called `libguestfs` that contains tools for accessing and modifying disk images like ours. Install it with:

```
apt install libguestfs-tools
```

Then make a copy of the image file (just in case...):

```
cp focal-server-cloudimg-amd64.img focal-server-cloudimg-amd64-agent.img
```

And then slipstream the agent into it:

```
virt-customize -a focal-server-cloudimg-amd64-agent.img --install qemu-guest-agent
```

We now have a ubuntu server cloud-ready image with qemu-guest-agent present.


#### Create the VM Template

First, get the next available VM ID:

```
pvesh get /cluster/nextid
```

Now, create a new VM with that ID:

```
qm create 138 --name "ubuntu-cloudready-template" --memory 2048 --net0 virtio,bridge=vmbr0
```

You want to have minimum specs, but that will depend on the OS. You're also free to change the bridge, or any other settings. But all of it is modifiable for new instances, so it doesn't matter too much. 

Import the image we modified, and set it as the vm disk:

```
qm importdisk 138 focal-server-cloudimg-amd64-agent.img local-lvm
qm set 138 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-138-disk-0
```

Next, attach the cloud init disk (a magical auto-generated cd-rom device):

```
qm set 138 --ide2 local-lvm:cloudinit
```

Set the boot disk to be the new one:

```
qm set 138 -boot c --bootdisk scsi0
```

Enable qemu-agent:

```
qm set 138 --agent enabled=1
```

Finally, make the VM as a template. Whilst not strictly necessary, it speeds up clone times.

```
qm template 138
```


If you look in your Proxmox GUI, you'll find a new template VM in there.



### Step 2 - Basic Terraform Configuration

Now, you'll need to [install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli){:target="_blank"} according to their docs. I installed the autocomplete too.

Once you have the Terraform CLI working, create a folder for your terraform 'code':

`mkdir terraform && cd terraform`


#### Secrets

You'll need to specify credentials for Terraform to use to interact with the Proxmox API. You can either use a username/password, or an API token. It's simple to do either way, but I used a token. 

One of the great things about Proxmox is it's useful wiki. The relevant pages are [Proxmox VE API](https://pve.proxmox.com/wiki/Proxmox_VE_API){:target="_blank"} and [User Management](https://pve.proxmox.com/wiki/User_Management#pveum_tokens){:target="_blank"}.

To create an API token, go to Datacenter -> Permissions -> API Tokens -> Add. Because I'm lazy, I created it for the root user and *unchecked Privilege Separation*. This means the token has access to everything th root user does. In the future, I may go back and try and figure out what the least privilege that I really need is, but for now, since I'm the only user of the server, it's fine.

Like all code automation, secrets are difficult to handle. You can do some fancier things with Vaults, but for my purposes it is enough to have them in environment variables. To do this, export the variables in your `.bashrc`:

```
export PM_API_TOKEN_ID="root@pam!terraform"
export PM_API_TOKEN_SECRET="<secret>"
```

And don't forget to `source ~/.bashrc` afterwards.


#### Providers

Terraform has a plugin-architecture where 'Providers' can be added to interact with different backends. Most people use Terraform with things like Heroku or AWS, but there is a third party provider for Proxmox; [`telmate/proxmox`](https://github.com/Telmate/terraform-provider-proxmox). Terraform makes installing new providers easy - if you declare a provider is required, it will install it for you when running the `Terraform init` command.

Create a file named `main.tf` with the following contents:

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
    pm_api_url = "https://<proxmox_ip>:8006/api2/json"
    pm_tls_insecure = "true"
}
```

The version is the latest as of writing, I don't know how to make this automatically update (using the string 'latest' didn't work).

This file specifies that a provider `telmate/proxmox` should be installed, and points it at our Proxmox server. If you have proper SSL certificate trust set up, by all means change `pm_tls_insecure`, but I need it since it's just a self-signed certificate.

Now, from this directory run

```
terraform init
``` 

And terraform will download and install the plugin for you.


#### Test Proxmox works

There doesn't seem to be a good way to test that the proxmox provider can actually access the Proxmox server. But, if you enable debug logging and have a bare minimum proxmox resource, you can check the debug log and ensure you could get to it.

First, enable debug as per the [Add Logging](#add-logging) section of troubleshooting.

Then, create a `vm.tf` file, and populate with the following:

```
resource "proxmox_vm_qemu" "terravm01" {
        name = "terravm"
        target_node = "hades"
}
```

Note; you'll need to change the target_node to be whatever Proxmox node you want to use.

Now, run terraform with: 

```
terraform apply
```

This will fail, but if you look inside the `terraform-plugin-proxmox.log` file, if it was successful you should see requests and replies from the server, at which point you know it's working, and your credentials are correct.


### Step 3 - Proxmox Terraform time


At this stage we have a proxmox VM Template available to clone from, and we have a basic terraform template. Now we need to expand it to define the new instance we want to create.


#### Describe the instance


Create `vm.tf` in the terraform folder (replace the contents if you created it in a previous step). This is the file in which we will define our instance. Terraform basically concatenates all the `.tf` files in a directory together (without recursing), so you can actually call this file whatever you want. 

Here's my complete `vm.tf` file, which I'll go through in detail below:

```
resource "proxmox_vm_qemu" "terravm01" {
  
    name = "terravm01"
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

    ipconfig0 = "ip=dhcp"
    sshkeys = "${file("~/.ssh/id_rsa.pub")}"

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

```
resource "proxmox_vm_qemu" "terravm01" {
```

Defines the resource named 'terravm01' which is of type 'proxmox_vm_qemu' - this type is defined by the telmate/proxmox provider we use. The name is irrelevant outside of the tf script.

The first big chunk define some basic properties about the VM:

```
name = "terravm01"
target_node = "hades"
clone = "ubuntu-cloudready-template"
os_type = "cloud-init"
balloon = 1024
boot = "order=scsi0"

agent = 1

cores = 2
sockets = 1
memory = 2560
```

Most importantly we define the name (terravm01), the target node, and the vm we are cloning from (The 'ubuntu-cloudready-template' template we created previously).

The os_type is special (cloud-init), balloon defines the minimum memory to give to the VM, and we specify the boot order just incase we forget to do it properly in the template.

We also specified the agent is enabled, and give the VM 2 cores (1 socket) and 2.5 gigs of max memory (for it to balloon into).


Next comes the disk block:

```
disk {  
        size            = "10G"
        type            = "scsi"
        storage         = "local-lvm"
}
```

Where we define the disk for the VM. We give it 10 gigs, because it doesn't need to be big.

The next blog 'vga' seems to be required, and caused an error if I didn't include it. I just set it as the standard.

```
vga {
  type = "std"
}
```

The network block defines the network adapter for the instance. You can modify this as you wish (for example, if you're creating an entire isolated network of machines).

```
network {
    model = "virtio"
    bridge = "vmbr0"
}
```


In the next section we set some cloud-init properties for the proxmox VM. There are a few ways in which you can pass cloud-init properties through, I chose the easiest way. There is a more powerful method, in which you send over an actual cloud-init config file to the proxmox server for the instance. For now I really only want to configure the network and sshkey for the user.

Here, I read the ssh public key from a local file with some dynamic code, but you can put the actual key in there too, if you prefer. The public key will be put into the ubuntu user's `~/.ssh/authorized_keys` file.

```
ssh_user = "ubuntu"
ipconfig0 = "ip=dhcp"
sshkeys = "${file("~/.ssh/id_rsa.pub")}"
```

The lifecycle block is as the comment describes - a workaround to stop the recreation of the instance every time you do `terraform apply`.

The next block describes how Terraform can connect to the instance after creation:

```
connection {
        type = "ssh"
        user = "${self.ssh_user}"
        private_key = "${file("~/.ssh/id_rsa")}"
        host = "${self.ssh_host}"
        port = "${self.ssh_port}"
}
```

Mostly self explanatory, the private_key corresponds to the public key from the earlier `sshkey` variable.

Finally, the remote exec provisioner block:

```
provisioner "remote-exec" {
    inline = [  
      // here you will see the actual ip address        
      "/sbin/ip a"  
    ]   
```

This connects to the instance and dumps its ip address, which makes sure everything is working (and displays the ip we need to connect to)


#### Instantiate!

Run `terraform apply` to start up the instance. If you have errors in your `.tf` you can correct them and try again, if you have issues consult the [Troublshooting](#troubleshooting) section below.

If all goes to plan, you should see output like this:

```
  Enter a value: yes

proxmox_vm_qemu.terravm01: Destroying... [id=hades/qemu/137]
proxmox_vm_qemu.terravm01: Destruction complete after 4s
proxmox_vm_qemu.terravm01: Creating...
proxmox_vm_qemu.terravm01: Still creating... [10s elapsed]
proxmox_vm_qemu.terravm01: Still creating... [20s elapsed]
proxmox_vm_qemu.terravm01: Still creating... [30s elapsed]
proxmox_vm_qemu.terravm01: Still creating... [40s elapsed]
proxmox_vm_qemu.terravm01: Still creating... [50s elapsed]
proxmox_vm_qemu.terravm01: Still creating... [1m0s elapsed]
proxmox_vm_qemu.terravm01: Provisioning with 'remote-exec'...
proxmox_vm_qemu.terravm01 (remote-exec): Connecting to remote host via SSH...
proxmox_vm_qemu.terravm01 (remote-exec):   Host: 192.168.1.116
proxmox_vm_qemu.terravm01 (remote-exec):   User: ubuntu
proxmox_vm_qemu.terravm01 (remote-exec):   Password: false
proxmox_vm_qemu.terravm01 (remote-exec):   Private key: true
proxmox_vm_qemu.terravm01 (remote-exec):   Certificate: false
proxmox_vm_qemu.terravm01 (remote-exec):   SSH Agent: false
proxmox_vm_qemu.terravm01 (remote-exec):   Checking Host Key: false
proxmox_vm_qemu.terravm01 (remote-exec):   Target Platform: unix
proxmox_vm_qemu.terravm01 (remote-exec): Connected!
proxmox_vm_qemu.terravm01 (remote-exec): 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
proxmox_vm_qemu.terravm01 (remote-exec):     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
proxmox_vm_qemu.terravm01 (remote-exec):     inet 127.0.0.1/8 scope host lo
proxmox_vm_qemu.terravm01 (remote-exec):        valid_lft forever preferred_lft forever
proxmox_vm_qemu.terravm01 (remote-exec):     inet6 ::1/128 scope host
proxmox_vm_qemu.terravm01 (remote-exec):        valid_lft forever preferred_lft forever
proxmox_vm_qemu.terravm01 (remote-exec): 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
proxmox_vm_qemu.terravm01 (remote-exec):     link/ether be:3e:70:fc:eb:28 brd ff:ff:ff:ff:ff:ff
proxmox_vm_qemu.terravm01 (remote-exec):     inet 192.168.1.116/24 brd 192.168.1.255 scope global dynamic eth0
proxmox_vm_qemu.terravm01 (remote-exec):        valid_lft 86394sec preferred_lft 86394sec
proxmox_vm_qemu.terravm01 (remote-exec):     inet6 fe80::bc3e:70ff:fefc:eb28/64 scope link
proxmox_vm_qemu.terravm01 (remote-exec):        valid_lft forever preferred_lft forever
proxmox_vm_qemu.terravm01: Creation complete after 1m5s [id=hades/qemu/137]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
manager@linux-manager:~/terraform$ 
```

And be able to log in to the new machine:

```
manager@linux-manager:~/terraform$ ssh ubuntu@192.168.1.116
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Jul 18 00:48:37 UTC 2021

  System load:  0.18              Processes:             122
  Usage of /:   15.1% of 9.52GB   Users logged in:       0
  Memory usage: 14%               IPv4 address for eth0: 192.168.1.116
  Swap usage:   0%


0 updates can be applied immediately.


Last login: Sun Jul 18 00:47:16 2021 from 192.168.1.136
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@terravm01:~$ 
```

Cool!


## References

- [Cloud-init support (Proxmox Wiki)](https://pve.proxmox.com/wiki/Cloud-Init_Support){:target="_blank"}
- [Telmate/Proxmox Cloud-init guide (telmate)](https://github.com/Telmate/terraform-provider-proxmox/blob/master/docs/guides/cloud_init.md){:target="_blank"}
- [Provision Proxmox VMs with Terraform Quick and Easy (vectops)](https://vectops.com/2020/05/provision-proxmox-vms-with-terraform-quick-and-easy/){:target="_blank"}




## Troubleshooting

### Add Logging

You can increase logging output for both Terraform and the Proxmox provider. To debug Terraform issues, export the log level before running the terraform command:

```
export TF_LOG=trace
```

See the available log levels [here](https://www.terraform.io/docs/cli/config/environment-variables.html){:target="_blank"}

To enable debug logging in the provider, use the following provider block in your `main.tf` which specifies a log file:

```
provider "proxmox" {
  pm_api_url = "https://<proxmox ip>:8006/api2/json"
  pm_tls_insecure = "true"
  pm_log_enable = true
  pm_log_file = "terraform-plugin-proxmox.log"
  pm_log_levels = {
    _default = "debug"
    _capturelog = ""
  }
}
```

### Invalid boot disk

Make sure you a) get the right img file (not 'disk'), and b) set the disk image to the template after importing it like:

```
qm set $next_id --scsihw virtio-scsi-pci --scsi0 "local-lvm:vm-138-disk-0"     
```

### Slow vm creation (10+ minutes) or don't get dhcp IP output

Make sure you add the qemu agent to the template with virt-customize, and make sure you enable it in the template (and in the `.tf` file might be necessary too, not sure).


### Terraform plugin crash

I had the plugin crash in:

```
goroutine 38 [running]:
github.com/Telmate/proxmox-api-go/proxmox.(*Client).GetVmRefsByName(0xc0004b0280, 0xc00032eb80, 0x9, 0xb9c6a0, 0xd82630, 0x0, 0x0, 0xc000209600)
```

Check your logs for something like:

```
2021/07/18 00:52:43 [DEBUG] checking for duplicate name
2021/07/18 00:52:43 >>>>>>>>>> REQUEST:
2021/07/18 00:52:48 >>>>>>>>>> REQUEST:
2021/07/18 00:52:53 >>>>>>>>>> REQUEST:
```

In my case, it was because I had a typo in the proxmox url. Sure, the plugin have a better error, but we should probably give it proper urls :)



## Conclusion

I hope you found this blog post useful. This is the first in a series on Proxmox automation. Here I described how to get up and running with Terraform and Proxmox, and we created a cloud-init Ubuntu Server templated and instantiated a clone of it automatically with different vm settings.

In subsequent posts I plan extend this to include installing and configuring software on the new instances, and also to create more advanced networks of machines (like a kubernetes cluster, or a windows Active Directory domain).
