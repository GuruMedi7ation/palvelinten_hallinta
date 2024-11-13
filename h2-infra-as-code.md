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
![eka_vagrantfile](https://github.com/user-attachments/assets/cd1a237f-2c09-457e-94aa-78f1ca2e1202)  

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
about the **Vagrant.configure("2")**-line and **config.vm.synced_folder**-line.  
The former is used for Vagrant's version control so that it's backward compatible. It's interesting to note that we can mix and match  
Vagrant configuration versions in a single Vagrantfile if necessary.  
**config.vm.synced_folder** lets user set a shared folder between the host and the guest VMs.  

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
Oh boy, I'm excited to get to see if this works! Notice that we haven't touched Salt's configuration files in our installation script.  
Let's go create a folder for this attempt to keep things tidy..  After quick-fixing a minor case-sensitive issue with my script with ChatGPT..    

...IT'S UP!   

![provisioning](https://github.com/user-attachments/assets/f5f5e245-b9d8-44da-8c74-1e757bf945af)  

Everything seems to be provisioned as per our vagrantfile.  

![vagrant_ping](https://github.com/user-attachments/assets/09399691-d6ee-4c35-a4a1-075a1e0c911a)  

And network seems to be working as intended, both VMs have the IPs set correctly.  

## D) Master-minion architecture on the same network

Now we'll demonstrate Master-Minion architecture on the two VMs we just created.
Since we already installed Salt-Master and Salt-Minion on our VMs, this will be very easy and quick.

First, we will cmd **vagrant ssh slave** to go configure salt minion with cmd **sudoedit /etc/salt/minion**  

We'll configure minion to call back to our Salt-Master on 192.168.1.10. We will also specify the correct port to use.  
![minion_cfg](https://github.com/user-attachments/assets/8948f819-f3bf-4582-beda-08a0b2d2887d)
  

Then we will proceed to kick the daemon for the cfg changes to take effect with cmd **sudo systemctl restart salt-minion.service**  

We log into our Salt-Master with cmd **vagrant ssh master** and then see if we have pending keys to acccept with cmd **sudo salt-key -L**  
Everything is in order, so we proceed to accept all pending keys with cmd **sudo salt-key -A**  
![slave01_key](https://github.com/user-attachments/assets/ac8e8e7d-bde6-4ee6-a372-1bdf8274300f)  

We will test our system by using cmd **sudo salt 'slave01' cmd.run 'whoami'**

![i_am_root](https://github.com/user-attachments/assets/33194b5d-f91d-4957-bd7f-ea76fabef281)

I am (G)root.  

## E) Hej, It's code as infra!

Our next task is to create a .SLS-file that creates a file at /TMP-folder.  
I don't think we'll need a Top.sls for this quite yet, so we'll proceed to **/srv/salt/** on our Salt master  
and **touch touched.sls**. The contents of the file look as following:  

```touched:
  file.managed:
    - name: '/tmp/cogitoergosum.txt'
```  

Let's test our .sls file with cmd **sudo salt-call --local state.apply touched**

![cogitoergosum](https://github.com/user-attachments/assets/c8e5b3a7-b195-4488-9d8b-653a493df924)

It works beautifully! Salt gave us a warning about creating an empty file.  

## F) Over the Internet and far away

Next we will invoke our **touched.sls** over the internet on our minion, slave01. 
For this, we cmd **sudo salt 'slave01' state.apply touched** on our master VM.

![touched_slave](https://github.com/user-attachments/assets/2dca3800-a840-4d8e-a89a-5c08a0252c97)  

Since it's so easy to hop into our minion with cmd **vagrant ssh slave**, we'll do just that to verify the file really is there.  
![IthinkIlive](https://github.com/user-attachments/assets/b18f2da0-0818-4ad1-8d5d-63de981256ec)  

## G) Freestyle .SLS with two state-describing funtions

Our next task is to create a .sls-file using two state describing functions. I'll try to create a sls-file that upgrades
all system packages to latest versions, and I want also to create a text file that tells when the updates have been ran.

For this, we will use **pkg.uptodate** to update our system packages, and **file.managed** to create the text file.

At first, our code did not compile, but thankfully the error message was clear: **"State 'twostates' in SLS 'twostates' is not formed as a list"**  
I used a YAML-validator to double check at **https://www.yamllint.com/** - but to my surprise, the code still did not compile! The same error message again  

I made small adjustments, and I went from  
![yaml_error_twostates](https://github.com/user-attachments/assets/fa85f1a5-b91a-4b67-b441-cef3d923bbd0)

To this:
```
twostates:
  file.managed:
    - name: /home/vagrant/lastupdated.txt
    - contents: The system has been last updated X.X.X
system_packages:
  pkg.uptodate
```
The new format works and the .SLS compiles nicely. Now that we could use something more descriptive than **twostates:** on the first line  
For example **creating_a_log:** would be better. Same for goes for **system_packages:**

Here's the result. There's only 1 change that was made to the system, because I tested **file.managed** before running the full .SLS including
the system package updates, and I thus created our **lastupdated.txt** already.

![twostates](https://github.com/user-attachments/assets/3d3dc5d6-25cf-4ecf-9d76-2016f4fb17e4)

Next we'll prove idempotency by invoking **twostates.sls** again  
![two_states_idempotent](https://github.com/user-attachments/assets/273b1900-b8bc-4000-a2f4-2aae54348a72)  

Notice how quick the run time was compared to the previous run! The desired end state has been achieveed, so no changes  
were done. Our .SLS file is thus idempotent!

## H) The Top File / Two modules at the same time  

I'm a lucky boy! We implemented a top file last week, so now that we're actually tasked with building a top file this week, we're ahead a little!

The top file will be located at **/srv/salt/**. The top file is accompanied with two state files/modules, **twostates.sls** and **touched.sls**,
which we created earlier. I also wanted to create a timestamp on the **lastupdated.txt**. I asked ChatGPT for some ideas how to implement this without using  
Jinja, which could be used to create dynamic content for our Salt state files. In the end, we went with a following **cmd.run** implementation

```timestamp:
  cmd.run:  
    - name: echo "Timestamp: $(date '+%Y-%m-%d %H:%M:%S')" >> /home/vagrant/lastupdated.txt```

However, it did not work. The culprit seemed to be what I interpret as a syntax error.
![syntax_error_1](https://github.com/user-attachments/assets/bbe150e8-e808-4048-bbec-58c53684f71c)

I asked ChatGPT again for some guidance, and we gave our timestamp another attempt at life. Here's our full code for our **twostates.sls**

```
twostates:
  file.managed:
    - name: /home/vagrant/lastupdated.txt
    - contents: The system has been last updated X.X.X
system_packages:
  pkg.uptodate
timestamp:
  cmd.run:
    - name: "echo \"Timestamp: $(date '+%Y-%m-%d %H:%M:%S')\" >> /home/vagrant/lastupdated.txt"
    - require:
      - file: twostates```  

For my surprise this our state file compiled nicely!  

![Great_success](https://github.com/user-attachments/assets/4d21e18d-ca55-4134-90ea-77978ddfaa92)  

Let's go see how it looks in the target file @slave01! 

![great_success_2](https://github.com/user-attachments/assets/488ba357-40f5-48ec-973a-1d4e74406d94)  

I'm so happy! ChatGPT also told us we could use a following method to ensure our more complex **cmd.run**-commands run properly.

```timestamp:
  cmd.run:
    - name: "bash -c 'echo \"Timestamp: $(date \"+%Y-%m-%d %H:%M:%S\")\" >> /home/vagrant/lastupdated.txt'"```

According to ChatGPT, this method allows us to:  
"bash -c: This tells Salt to execute the command in bash, which supports the $(...) syntax.
Escaped Quotes: The inner double quotes are escaped with \ to ensure YAML interprets them correctly and allows for proper string expansion within bash."

Let's go and test that method!

It works! It's interesting to note the end result:

![bash_works_too](https://github.com/user-attachments/assets/31fa07a6-7324-4b1c-9a84-48e2d82c62bf)

It seems like the **bash -c**-method ends up with salt interpreting this as two changes have been made, which is indeed correct!
 


  







































 









   

   

     
   
    
    
   
 














    
