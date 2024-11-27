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





