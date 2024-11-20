## Daemon

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

It worked!

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

![fully_idempotent1](https://github.com/user-attachments/assets/4d6c1853-1422-4a17-8c6a-5217f7f334dc)
![fully_idempotent2](https://github.com/user-attachments/assets/bec47e2f-efc2-401a-b21b-32a8ee779e75)

That's all, folks!

## C) PROJECT

For my project, I was thinking about to automatize putting up a security testing home laboratory.
For this, we'll leverage Vagrant to create an isolated network, for which we'll provision an X number of machines.

We'll do a script for provisioning the VMs - and we'll also provision a VM for Salt Master to be installed on ( with our previous script )

After that, we'll leverage Salt to install "stuff" on our Salt minions and to do some "stuff" on our minions.

The stuff is still up for a debate, so suggestions are welcome! I was thinking about Zed attack proxy, which can be scripted with YAML,
If I've understood it right. Perhaps something else as well, a windows honeybox setup for those Microsoft Service center callers who want you
to install Anydesk on your computer would be fun!

## D) Apache for homedir users without privileged access

Welcome back, Travellers! Today we'll install Apache to serve webpages from people's home directories in such a way, that the users
can edit their content without privileged access. 

For this, we need to tinker with the userdir-module for Apache2. For more detailed information on this, you can visit:
https://httpd.apache.org/docs/2.4/mod/mod_userdir.html.  

Let's first enable userdir-module for all users by cmd **sudo a2enmod userdir**. After cmd **sudo systemctl restart apache2** we should
have a mod called userdir in our **/etc/apache2/mods-enabled/**-folder

This is how **userdir.conf** looks like.

![userdir_conf](https://github.com/user-attachments/assets/2960c749-1c7c-40a3-bc7d-4484c49af585)

As you can see, the users' user directory is served from **/home/~name/public_html/** We're good with these settings.
**UserDir disabled root** means that root doesn't have a public html folder by accident, which is a good idea.

Next, we want to create public_html folder in the home directory. Cmd **sudo mkdir public_html** when in home directory of a user.
We'll later leverage salt to create a user with public_html-folder in its home directory.

We also created a dummy index.html just to check the privileges of the file.
![664_rights](https://github.com/user-attachments/assets/bfc6ff25-acc6-4dd7-9b4b-8668f5b9e07a)

Looks like our index.html has 664 rights by default.

The privileges are calculated in the following way: (r)ead is valued at 4, (w)rite is valued by 2, e(x)ecute is valued by 1
The values are additive, so if the user has reading and writing privileges, the privilege code is 6.

So our example file has 6 for owner privileges, 6 for group privileges, and 4 for others.

I think that should work fine with what we're going for.

Let's ask ChatGPT to create some funny content for our test user's web content.
I asked ChagGPT to give us a index.html that displays a disco-dancing animated ASCII skeleton.

I dropped our index.html in **/home/vagrant/public_html/** by cmd **nano index.html** to see if we can edit the file  
without elevated privileges. It worked fine!

![vagrant_user](https://github.com/user-attachments/assets/88ff55c6-4d51-4fd2-85a0-909d029e3f59)

Now let's go see what we got. I don't think lynx supports javascript that was used to display our test page, so we'll
use host OS's browser.

![disco](https://github.com/user-attachments/assets/61111d74-d21a-45e0-9c99-9d2d30815864)

It's animated, it does disco, and it has flashing lights. It is, however, not a skeleton! I'll give it 4 our of 5.

Let's create a Salt module to automatize this on our salt minion.

```
enable_userdir:
  cmd.run:
    - name: a2enmod userdir
kick_the_daemon:
  service.running:
    - name: apache2
    - enable: True
    - reload: True
/etc/apache2/mods-enabled/userdir.conf
  file.managed:
    - source: salt://public_html/userdir.conf
/home/vagrant/public_html/index.html
  file.managed:
    - source: salt://public_html/index.html
```

So, this is our code. I think enabling userdir and kicking the daemon could be separated into their own module, as these are one-off operations,
but we'll go with this now. We still need to proceed to copy **userdir.conf** to **srv/salt/public_html/** for this to work.
Same counts for our **index.html**

Let's go do this now, and hope there won't be any problems with the privileges as we copy the cfg!

![problems](https://github.com/user-attachments/assets/0fd58fbf-fefe-45ac-bfbf-be4934a5fde3)

Oh well... As it says, there are lacking :'s after a few lines. let's revise and give it another go.

![hmmmm_improvements_required](https://github.com/user-attachments/assets/47dcd5be-23d5-436a-9350-91afac81ba5c)

Hmm.. This could be improved. The CMD.run ran fine, of course, but our daemon was not kicked properly. I wonder, if we can address this using **watch**
moreover, it seems like we need to create public_html directory for our users.

According to https://docs.saltproject.io/en/latest/ref/states/all/salt.states.file.html#salt.states.file.directory we can do this by adding
**- makedirs: True** under our **file.managed:**

Let's try these.

![great_success_3](https://github.com/user-attachments/assets/64066bf0-0c1d-4d62-aa11-434d0fb399cf)

Hooray! Apache2 was properly kicked, a directory for the user html was created! We're living the life!

Let's go test if our **index.html** works on our minion by using our Host OS's browser.

![dance_2](https://github.com/user-attachments/assets/961165e4-4193-4d3d-9447-d5d76219ab64)

It's up!!

Now we need to test can we edit things without elevated privileges.
Oh no, we cannot edit the file. Let's troubleshoot with cmd **ls -la**

![wrong_permissions](https://github.com/user-attachments/assets/2609d228-9184-4e91-bb35-54ad4af7be57)

Let's modify our code by adding **- user: vagrant** and **- group: vagrant** under our **file.managed** for index.html

It also looks like the entire directory needs the proper permissions, too. Let's delete the files and the directory, and then try
again after running our updated state file.

![permissions_worked](https://github.com/user-attachments/assets/2bf4edd2-778b-4b22-b607-71287a7f33b1)

We successfully deleted our old index.html replaced it with a new index.html with KITT car from the cult classic Knight Rider. Let's go check it out!
All this without leveraging sudo!

![Asciii_KITT_success](https://github.com/user-attachments/assets/1bdfb39f-a1a2-432a-80fe-e033bfeb1304)

It's there! Notice we're accessing /~vagrant/ on Apache2 running on slave01

Our final state file is as follows:

```
enable_userdir:
  cmd.run:
    - name: a2enmod userdir
kick_the_daemon:
  service.running:
    - name: apache2
    - enable: True
    - reload: True
    - watch:
      - cmd: enable_userdir
/etc/apache2/mods-enabled/userdir.conf:
  file.managed:
    - source: salt://public_html/userdir.conf
/home/vagrant/public_html/index.html:
  file.managed:
    - source: salt://public_html/index.html
    - makedirs: True
    - user: vagrant
    - group: vagrant
    - mode: 644
```
![right_permission_2](https://github.com/user-attachments/assets/79233280-8f96-4f56-8ad9-ea926176bb1b)


I'm excited to see you again next week, let's have fun!





















































 






