## INFRA AS CODE

Welcome back, Traveller.

One of you asked me last week about the hardware I'm running these exercises on.  
CPU: R7 5700X@stock clocks  
RAM: Kingston DDR4-3200 32GB CL16  
GPU: RX6600@stock clocks  
SSD: WD Black 1TB + 0.5TB legacy SSD + 1TB legact HDD  
OS: Latest Win 11  

## A) Vagrant-installation  

I installed Vagrant on my host OS  
To verify the installation, we proceed with a command:    
    **vagrant -v**  
![vagrant_v](https://github.com/user-attachments/assets/d78baf09-ebd9-4b18-b6db-c02f57a7a208)

## B) Creating a new Linux VM using Vagrant

Next we will create a new Linux VM using Vagrant  
For this, I first explored **https://developer.hashicorp.com/vagrant/docs/vagrantfile** for information about vagrantfile, as well as 
**https://terokarvinen.com/2023/salt-vagrant/#infra-as-code---your-wishes-as-a-text-file** for general information about Vagrant usage.

Leveraging cmd **Vagrant -h** I discovered I can use cmd **vagrant init** for creating a sample Vagrantfile. Let's do this.

The end result looks as following. A bit messy, but instructive.  
![eka_vagrantfile](https://github.com/user-attachments/assets/7ad07024-eaa6-4f7b-95a3-300366a70ca6)

Before we clean this up, let's try creating a single Linux VM. For this we need to find a VM provider using
a ready made Vagrant Box. We can search for these at **https://portal.cloud.hashicorp.com/vagrant/discover?query=**

It looks like many of these boxes are ancient, and I'm a little bit hesitant to use 3rd party software, so we'll use Hashicorps's own
standard Ubuntu 18.04 LTS 64-bit box.  

Let's edit Vagrantfile's **config.vm.box = "base"**-line to match **config.vm.box = "hashicorp/bionic64"**  
I'm curious to try if it works without enforcing version control.  

Let's try starting it with cmd **vagrant up**  

![eka_boksi](https://github.com/user-attachments/assets/6bb11b0b-d843-45ca-b49d-a3035987d224)  

Now that it's up, let's verify by logging in with SSH. We can do this with cmd **vagrant ssh**  
![vagrant_eka_in](https://github.com/user-attachments/assets/2f9a4171-ebf9-4466-93bc-8a681adf453f)  

Finally, we will stop the VM with cmd **vagrant halt** and proceed to destroy the VM with cmd **vagrant destroy** 

## C) Two VM network with Vagrant

Next we will build a public network with two VMs using Vagrant and infrastructure as code.  
I want to create a public network so that my VMs can call home to our Salt Master in the future.  

To get a reference about how a cleaned up vagrantfile might look like, I explored a vagrantfile at  
https://terokarvinen.com/2021/two-machine-virtual-network-with-debian-11-bullseye-and-vagrant/  

![two_VMs_vagrantfile_tero](https://github.com/user-attachments/assets/53d12128-4dab-42d6-b89f-5b4b82db78a5)  

Next, I looked at https://developer.hashicorp.com/vagrant/docs/vagrantfile/machine_settings to get more information  
about the **Vagrant.configure("2")**-line and **config.vm.synced_folder**-line. The former is used for Vagrant's version
control so that it's backward compatible. It's interesting to note that we can mix and match Vagrant configuration versions  
in a same Vagrantfile if necessary. **config.vm.synced_folder** lets user set a shared folder between the host and the guest VMs.  

Let's try to create a vagrantfile of our own by shamelessly scavenging Tero's work! Thanks, Tero! Since Tero was using a script
to install software for his Vagrant VM's, let's try to do the same by installing Salt minion and Salt master on our new VMs.  

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

$master_script = <<-MASTER_SCRIPT
set -o verbose
mkdir /etc/apt/keyrings
sudo curl -fsSL -o /etc/apt/keyrings/salt-archive-keyring-2023.gpg https://repo.saltproject.io/salt/py3/debian/12/amd64/SALT-PROJECT-GPG-PUBKEY-2023.gpg
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.gpg arch=amd64] https://repo.saltproject.io/salt/py3/debian/12/amd64/latest bookworm main" | sudo tee /etc/apt/sources.list.d/salt.list
sudo apt-get update
sudo apt-get install -y salt-master
sudo systemctl restart salt-master.service
MASTER_SCRIPT

$minion_script = <<-MINION_SCRIPT
set -o verbose
mkdir /etc/apt/keyrings
sudo curl -fsSL -o /etc/apt/keyrings/salt-archive-keyring-2023.gpg https://repo.saltproject.io/salt/py3/debian/12/amd64/SALT-PROJECT-GPG-PUBKEY-2023.gpg
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.gpg arch=amd64] https://repo.saltproject.io/salt/py3/debian/12/amd64/latest bookworm main" | sudo tee /etc/apt/sources.list.d/salt.list
sudo apt-get update
sudo apt-get install -y salt-minion
sudo systemctl restart salt-minion.service
MINION_SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/bionic64"

  config.vm.define "master" do |master|
    master.vm.provision :shell, inline: $master_script
    master.vm.network "public_network", ip: "192.168.1.10"
    master.vm.hostname = "master"
  end

  config.vm.define "slave" do |slave|
    slave.vm.provision :shell, inline: $minion_script
    slave.vm.network "public_network", ip: "192.168.1.11"
    slave.vm.hostname = "slave01"
  end
end
```   
Oh boy, I'm excited to get to see if this works! It's pays to note that Salt configs for both VMs have not been touched yet.  
Let's go create a folder for this attempt to keep things tidy..  After quick-fixing a minor case-sensitive issue with my script with ChatGPT..    

...IT'S ON!!!  

![provisioning](https://github.com/user-attachments/assets/f5f5e245-b9d8-44da-8c74-1e757bf945af)  

Everything seems to be provisioned as per our vagrantfile.  

![vagrant_ping](https://github.com/user-attachments/assets/09399691-d6ee-4c35-a4a1-075a1e0c911a)  

And network seems to be working as intended, both VMs have the IPs set correctly.  

## D) Master-minion architecture on the same network

Now we'll demonstrate Master-Minion architecture on the two VMs we just created.
Since we already installed Salt-Master and Salt-Minion on our VMs, this will be very easy and quick.

First, we will **vagrant ssh slave** to go configure salt minion with **sudoedit /etc/salt/minion**  

We'll configure minion to call back to our Salt-Master on 192.168.1.10.  









   

   

     
   
    
    
   
 














    
