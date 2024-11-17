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














 






