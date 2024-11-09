## INFRA AS CODE

Welcome back, Traveller.

One of you asked me last week about the hardware I'm running these exercises on.  
CPU: R7 5700X@stock clocks  
RAM: Kingston DDR4-3200 32GB CL16  
GPU: RX6600@stock clocks  
SSD: WD Black 1TB + 0.5TB legacy SSD + 1TB legact HDD  
OS: Latest Win 11  

A) Vagrant-installation  

I installed Vagrant on my host OS  
To verify the installation, we proceed with a command:    
    **vagrant -v**  
![vagrant_v](https://github.com/user-attachments/assets/d78baf09-ebd9-4b18-b6db-c02f57a7a208)

B) Creating a new Linux VM using Vagrant

Next we will create a new Linux VM using Vagrant  
For this, I first explored **https://developer.hashicorp.com/vagrant/docs/vagrantfile** for information about vagrantfile, as well as 
**https://terokarvinen.com/2023/salt-vagrant/#infra-as-code---your-wishes-as-a-text-file** for general information about Vagrant usage.

Leveraging cmd **Vagrant -h** I discovered I can use cmd **vagrant init** for creating a sample Vagrantfile. Let's do this.

The end result looks as following. A bit messy, but instructive.  
![eka_vagrantfile](https://github.com/user-attachments/assets/7ad07024-eaa6-4f7b-95a3-300366a70ca6)

Before we clean this up, let's try creating a single Linux VM. For this we need to find a VM provider using
a ready made Vagrant Box. We can search for these at **https://portal.cloud.hashicorp.com/vagrant/discover?query=**

It looks like many of these boxes are ancient, and I'm a little bit hesitant using 3rd party software, so we'll use Hashicorps's own
standard Ubuntu 18.04 LTS 64-bit box.  

Let's edit Vagrantfile's **config.vm.box = "base"**-line to match **config.vm.box = "hashicorp/bionic64"**  
I'm curious to try if it works without enforcing version control.  

Let's try starting it with cmd **vagrant up**  

![eka_boksi](https://github.com/user-attachments/assets/6bb11b0b-d843-45ca-b49d-a3035987d224)  

Now that it's up, let's verify by logging in with SSH. We can do this with cmd **vagrant ssh**  
![vagrant_eka_in](https://github.com/user-attachments/assets/2f9a4171-ebf9-4466-93bc-8a681adf453f)  

Finally, we will stop the VM with cmd **vagrant halt** and proceed to destroy the VM with cmd **vagrant destroy**  







    
