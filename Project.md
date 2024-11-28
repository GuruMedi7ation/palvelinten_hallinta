## Project: Security lab

This week we will set up a vagrantfile for our security lab.  

The laboratory VMs will consist of the following:
1x Debian /w Salt Master  
1x Kali Linux  
1x OWASP Webgoat  
1x Metasploitable  
Perhaps we will also do 1 Windows-based VM, it would be fun.  

I've been reading Vagrant documentation, and it seems that we can perhaps use cmd **vagrant reload**  
to refresh our vagrantfile configuration. I hope this will be useful in switching from a public network  
to a private one. Ultimately we want our lab setup to be an isolated network without internet access,  
but we will want internet access to install software using our Salt magic first.  

It would be better to have all the installed-to-be dependencies locally on my computer, but since we are  
also going to be demonstrating leveraging salt to do some administrative tasks such as updating system files    
to latest available, we'll do it this way. We may also test using Jinja to create a dynamic state file    
to simulate Salt-usage in creating a batch of user accounts for an organization ( And perhaps setting up   
Apache to display the users' web pages, too )  

Salt is going to be installed via a script embedded in our vagrantfile to make things go a bit smoother,  
like I did on our second homework: h2-infra-as-code. However, this time we will use a locally saved copy  
of Salt to install on our newly provisioned VMs, so our vagrantfile and its script will serve us well in the  
future, too. 

## Let's get ready to rumble!

First, we download an up to date Salt from https://packages.broadcom.com/artifactory/saltproject-deb/pool/
Let's try to install it via provisioning script!

cmd **sudo dpkg -i /vagrant/salt-master_3007.1_amd64.deb** should install the package from the **shared VM** folder
that we've set up in our Vagrantfile. Here is our provisioning script:

```
$master_script = <<-MASTER_SCRIPT
set -o verbose
sudo dpkg -i /vagrant/salt-master_3007.1_amd64.deb
sudo systemctl restart salt-master.service
MASTER_SCRIPT

$minion_script = <<-MINION_SCRIPT
set -o verbose
sudo dpkg -i /vagrant/salt-minion_3007.1_amd64.deb
sudo systemctl restart salt-minion.service
MINION_SCRIPT
```
  
Unfortunately we have some problems in the paradise, due to missing dependencies.  
![provisioning_problem](https://github.com/user-attachments/assets/9c8e4523-c84b-461b-9f54-75d93147ca0a)  

Previously I noticed a file called **packages** when I downloaded our **salt-master.deb**  
https://packages.broadcom.com/artifactory/saltproject-deb/dists/stable/main/binary-amd64/

It describes the depencies of Salt, which will help us troubleshoot our problem.

![dependencies](https://github.com/user-attachments/assets/845f0847-bc09-4055-bb3d-6af447d0f95e)

The required depedencies can also be looked at https://docs.saltproject.io/salt/install-guide/en/latest/topics/install-dependencies.html

I have a bad feeling about this... It could be a lot. After thinking about it for a while and getting a little bit desperate,  
I thought about if I can use apt get just to download dependencies and store them on my local computer.

I invoked cmd sudo **apt-cache depends salt-common >> ~/salt_master_deps** to create a list of dependencies for salt-master,
salt-minion and salt-common in the following way. Apt has stored copies of the dependencies on my original Salt Master at  
**/var/cache/apt/archives** Now I'm really starting to get a bad feeling about this when I look what I scrounged up..

![uh_oh](https://github.com/user-attachments/assets/ebe13ac3-2ae7-4b49-ba6b-1724b902a15b)  

With that, I'll head to bed and figure out if I can grep and script this into something workable for our Vagrantfile,
or if I should just install over the web so I can move on with the task.







