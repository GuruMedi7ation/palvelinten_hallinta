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

Leveraging **Vagrant -h** I discovered I can use cmd **vagrant init** for creating a sample Vagrantfile. Let's do this.


    
