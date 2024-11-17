![SSH_ports](https://github.com/user-attachments/assets/6b0679a2-f54d-4e66-9dc4-2653aa560a24)## Daemon

## A) Apache Easy Mode

Our first task this week is to install Apache. We'll do the first installation manually, then we'll apply our Salt mastery to it,
and create a state file to automatize the process. Let's get it on!

We started the work with cmd **vagrant up** to get our beautifully crafted Salt Master up. We'll use this VM for Apache's manual installation.
One cmd **sudo apt-get install apache2** after we have our apache2 installed.

![apache2_manual_installation](https://github.com/user-attachments/assets/e2874e5f-fa2b-488b-8cf5-e4ca0f07f481)

Next, we will proceed to **/etc/apache2/** to have a look around. We should first create an example website on /etc/apache2/sites-available/
and configure it. We got lucky, and there is a default site configured for us. Let's cmd **cat 000-default-conf** and look around some more.

![default_conf](https://github.com/user-attachments/assets/a002d6a4-254d-490f-a6c7-4ddb5dcc3c7d)  

**/var/www/html/** is the folder where the assets of our default page are stored. Next, we will head there and cmd **ls -la**

Oh boy, there is an example **index.html** in there. Let's replace it! I'll leverage ChatGPT to save some time and I'll ask it to create us a 
simple index.html to test our site. We proceed to drop our new **index.html** at **/var/www/html/**

Let's kick the daemon with cmd **sudo systemctl restart apache2**

Next, we get our own IP with cmd **hostname -i** to have an address to visit with Lynx

And since we don't have Lynx installed, we'll do so with cmd **apt-get install lynx**

Let's view our website by visiting with it Lynx! We cmd **lynx 127.0.2.1/index.html** and cross our fingers...
![top_secret_fail](https://github.com/user-attachments/assets/538b2e3f-0b87-417f-85f0-dfc8b9f98e7d)  

And our top secret test site is up! W00t! I asked ChatGPT to write "This is Lauri's top secret test site" stylized as Ascii-art.  
It seems all that we got was "Ticotchid."...   

I asked ChatGPT what Ticotchid means, and apparently it stands for: "The Incredible Code Of Top-Secret Hidden Information Depot."  
I'll take that.  

Everything works fine when we try to enter our website from our local host, but when we use slave01 to enter our website, we cannot connect to host.
Ping (ICMP) reaches master just fine, TCP using netcat refuses our connection. Salt using port 22 works just fine for our C2..

**update** It's funny, but our Super Secret Test Site works just fine when accessed from slave01, I just need to access the right address for it,
not the Master VM's localhost.

Let's proceed to create a state file for our minion.

We'll create a new folder on our Salt master at **/srv/salt/apache2** where we'll create our state file

We created our state file, however we got the following error message when invoking our desired state for our minion.

![salt_apache_html_fail](https://github.com/user-attachments/assets/d4010890-4ebd-4d8e-94e1-07b483018ebb)  

Just like the error message says, **"Source file salt://var/www/html/index.html not found"** at the specified location. It looks like Salt
is looking for **/var/www/html/index.html** from our **/srv/salt**-folder. I have a feeling this could be configured from **/etc/salt/master**, but
it might be better practice just to copy our **index.html** on the same directory where our **apache2-module** exists. Let's do that and try again!

Our Apache2-module now looks like this:

```
apache2:
  pkg.installed
/var/www/html/index.html:
  file.managed:
    - source: salt://apache2/index.html
kick_the_daemon:
  service.running:
    - name: apache2
```
 
And the result is glorious. Our state file works as intended.  

![apache_state_applied](https://github.com/user-attachments/assets/9dc995c2-07d3-42a1-9185-43c444e44ac2)

Let's visit our Top secret test site on our minion with Lynx.  

![slave_lynx_test](https://github.com/user-attachments/assets/e4d471ba-4ca7-4677-9c15-72065fd8015a)

As you can see cmd **sudo systemctl status apache2** on our minion shows that apache2 is up and running..  

..and apache2 is serving Lauri's top secret test site correctly.  

## B) SSH-Out(o)

I interpreted our next task is to configure SSHD ( SSH Daemon/server ) to open a new port. Let's agree that the new port we want to open is 1234.

First, we create a new folder on our Salt master for our SSH-module. We do that with cmd **sudo mkdir /srv/salt/SSH**. Next, we proceed to  
copy the **sshd-config**-configuration file there.

Let's cmd **cat sshd-config** to tinker with the cfg-file.

![SSH_ports](https://github.com/user-attachments/assets/8d855eb9-350f-4d34-951b-61e6bac0edff)

We opened port 22 ( The default port ) for SSH. We're using that for our Salt C2. We also opened port 1234 AND we configured our SSH server
to display a following banner: "Unauthorized access will be prosecuted." The path for the banner in our SSHD-cfg is **/etc/ssh/sshd_banner**

We created the banner file on our Salt Master directory **/srv/salt/ssh/**

Next, we want to update our firewall configurations on our Salt minions. We'll update our firewall configurations using **cmd.run ufw allow 22 AND ufw allow 1234**
and we'll kick the daemon with **service.start**. **watch** is used to make Salt

Let's create a state file!

As we executed our file, we got the following error:
![salt_state_fail](https://github.com/user-attachments/assets/08cdcac5-4f05-4b4d-a94b-876e13091da2)
![salt_state_fail2](https://github.com/user-attachments/assets/7d7e10b5-073b-43e8-8044-b40054ad940f)

It seems like made a typo with our sshd_config. Let's correct that. We can also notice that we forgot to kick the daemon after dropping out cfg files
on our minion. Let's correct that as well.

We proceeed to cmd **sudo salt 'slave01' state.apply ssh** and we run it two times to prove that our ssh-module is idempotent.

![great_success](https://github.com/user-attachments/assets/425d0dfd-a1b1-4c13-bf7e-210173748150)
![great_success_2](https://github.com/user-attachments/assets/41e76dd5-6e4c-4e7d-929e-cc91fdb15b38)

Great success! Notice only two changes were made in the second run, that's because we use **cmd.run** to cmd **"sudo ufw allow"** to open ports on our
firewall and these commands are run every time we invoke our SSH module on Salt. It'd be nice to be able to make this module fully idempotent by using
**file.managed** to push our full configuration file to our minions, just like we do with **sshd_config**

Let's test that our stuff actually works by connecting with SSH to our minion by cmd **ssh 192.168.1.11** on our Salt Master
![unauthorized_access](https://github.com/user-attachments/assets/a9949a59-b089-4bfc-b870-9f656344f2b9)
![also_works](https://github.com/user-attachments/assets/10f8055a-ef59-4e7c-8003-ce6405ddb29a)  

Also cmd **sudo systemctl status ssh** shows that both ports 22 and 1234 are open.
![sshd_status](https://github.com/user-attachments/assets/214979b2-7899-45c4-951a-692c910a252b)

Here's our state module for updating SSHD_config and opening ports for our ufw firewall.

```
/etc/ssh/sshd_config:
  file.managed:
    - source: salt://ssh/sshd_config
/etc/ssh/sshd_banner:
  file.managed:
    - source: salt://ssh/sshd_banner
open_ufw_port_22:
  cmd.run:
    - name: 'sudo ufw allow 22'
    - unless: 'ufw status | grep -q "22/tcp"'
open_ufw_port_1234:
  cmd.run:
    - name: 'sudo ufw allow 1234'
    - unless: 'ufw  status | grep -q "1234/tcp"'
kick_the_sshd_daemon:
  service.running:
    - name: sshd
    - enable: True
    - watch:
      - file: /etc/ssh/sshd_config
      - file: /etc/ssh/sshd_banner
kick_the_firewall:
  service.running:
    - name: ufw
    - enable: True
```

**Update** 

We managed to make our state file fully idempotent by changing pushing ufw configuration files succeffully with **file.managed**-method.
We previously had a critical error with this one, because Salt was unable to read the **user.rules** file because of its permissions

As you can see, here lies the crux of the issue: 

![permissions](https://github.com/user-attachments/assets/d3f47105-4582-43f1-86a7-b617ef541d57)

I thought Salt runs our states with root permissions, but the **user.rules**- that we copied and modified at **/srv/salt/ssh/** was lacking  
read permission for **others**. This is really weird! We could get the correct permission with cmd **sudo chmod 644 /srv/salt/ssh/user.rules**

I wonder if we can break out state file by changing its permissions so that it does not have reading rights for **others**

let's try it with cmd **sudo chmod 640 /srv/salt/ssh/init.sls**

![its_broken](https://github.com/user-attachments/assets/90811700-9285-4d9b-9bba-2bee634f79d1)

It worked!!!

Pretty interesting!

Here's our modified **FULLY IDEMPOTENT** code for our SSH-module

```
/etc/ssh/sshd_config:
  file.managed:
    - source: salt://ssh/sshd_config
/etc/ssh/sshd_banner:
  file.managed:
    - source: salt://ssh/sshd_banner
kick_the_sshd_daemon:
  service.running:
    - name: sshd
    - enable: True
    - watch:
      - file: /etc/ssh/sshd_config
      - file: /etc/ssh/sshd_banner
/etc/ufw/user.rules:
  file.managed:
    - source: salt://ssh/user.rules
kick_the_firewall:
  service.running:
    - name: ufw
    - enable: True
    - watch:
      - file: /etc/ufw/user.rules
```

That's all, folks!




































 






