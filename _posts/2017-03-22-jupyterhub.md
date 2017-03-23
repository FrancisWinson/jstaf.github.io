---
title: Getting JupyterHub working on CentOS 7
---

Python and R are the the two biggest languages for data analysis.
The coding experience in each language could not be more different, however.

Coding in R is a nice, wonderful, standardized experience.
Everything comes from a central package repository, and there is an extremely polished,
single and multi-user development environment: RStudio.
Want to install R and RStudio Server in a multi-user configuration?
It's just 3 commands:

{% highlight bash %}
yum install R pandoc
wget https://download2.rstudio.org/rstudio-server-rhel-1.0.136-x86_64.rpm
yum install rstudio-server-rhel-1.0.136-x86_64.rpm
{% endhighlight %}

Annnnnnd we're done... that was easy.

The Python ecosystem on the other hand, is uh, well let's not talk about that too much. 
Competing distributions, 
a gajillion different IDEs to choose from, 
and there's somehow a group of heretics out there that are still using Python 2.

The dominant tool for data science in Python is the Jupyter notebook.
I personally hate them, but that's besides the point. 
Chances are, you might need to deploy them as part of a multi-user setup like a shared compute node. 
JupyterHub is RStudio Server's Python equivalent, 
and is the only real way to get multiple users setup with Jupyter notebooks and share a single compute environment.
Unfortunately, setting up JupyterHub (and doing so securely), is actually a pretty involved process. 
And there's no good documentation out there. 
Having to read the entire official JupyterHub documentation start to finish (broken links and all) isn't really a good option. 
We just want to get it over and done with.

So here's how you do it start to finish on CentOS/RHEL 7. 

-------------------------------------------------------

## Installation prerequisites

Before we go any further, this tutorial assumes that you have root access to a fresh install of CentOS/RHEL 7. 
Ubuntu is also fine, just be aware of the differences and change these instructions accordingly.
This guide assumes some familiarity with sysadmin, make sure you're comfortable with the command-line before starting. 

Anyhow, let's start off with installing some key "desert island" packages first:

{% highlight bash %}
yum install wget git vim xauth xorg-x11-apps eog evince firefox epel-release
yum install bzip2 gzip p7zip p7zip-plugins zip unzip
{% endhighlight %}

We're also going to make a non-root user to test things with and give them sudo access.

{% highlight bash %}
useradd jstaf
passwd jstaf
usermod -aG wheel jstaf
{% endhighlight %}

And finally, we are going to temporarily disable SELinux for the duration of this install.
Yes, leaving it off permanently is bad.
But having it off while doing the install eliminates it as a source of error, 
making it waaaaaaay easier to debug things if stuff doesn't work 
(and then you'll only have yourself to blame, not SELinux). 

{% highlight bash %}
setenforce 0
{% endhighlight %}

Ok, we're all set. 

## Installing Python

JupyterHub needs some form of Python for users to run.
Using the system Python is a bad idea because it offers the opportunity to muck up system programs and potentially break things (plus it comes with almost zero packages installed). 
Because of these considerations, we are going to install Anaconda instead.
Anaconda is a nice all-in-one Python distribution that's independent of system dependencies, 
includes most common data science packages, 
and some has some really sweet optimizations like Intel's MKL libraries.

We'll install the Python 3 version of Anaconda under `/opt`.
The installation will be owned by our non-root user, 
if only because I don't like managing Python packages as root.

{% highlight bash %}
mkdir -p /opt/python
chown jstaf:wheel /opt/python

# switch to our other user and install Anaconda
su - jstaf
cd /opt/python
wget https://repo.continuum.io/archive/Anaconda3-4.3.1-Linux-x86_64.sh
bash Anaconda3-4.3.1-Linux-x86_64.sh -b -p /opt/python/anaconda3
# this link lets you use both versions of pip simultaneously
# if you have both anaconda2 and anaconda3 on your path at
# the same time
ln -s /opt/python/anaconda3/bin/pip /opt/python/anaconda3/bin/pip3
{% endhighlight %}

We want this installation to be available to all users when they log onto this machine.
To do this, we will add an intialization script under `/etc/profile.d/`.
Scripts in this location get run when a user logs on, 
and can be used to setup their environment automatically.

Create a file called `/etc/profile.d/user-setup.sh` and have it include the following:

{% highlight bash %}
#!/bin/bash
export PATH=/opt/python/anaconda3/bin:$PATH
{% endhighlight %}

Log in as any user. 
If things have been setup correctly, 
the command `which python3` should point to our Anaconda install in `/opt/python/anaconda3/bin/python3`.
When you're done, switch back to the root user.

## Setting up a basic JupyterHub install

JupyterHub depends on NodeJS and several related packages to run. 
Let's install them now.

{% highlight bash %}
yum install npm nodejs 
npm install -g configurable-http-proxy
{% endhighlight %}

Now let's install JupyterHub into Anaconda3 (again, lets do this as our non-root user).

{% highlight bash %}
su - jstaf
pip3 install jupyterhub
{% endhighlight %}

To verify that things are working, we can start a JupyterHub server locally.
In one terminal window, run the command `jupyterhub`.
In another terminal window, 
running `firefox localhost:8000` should take us to a JupyterHub login page that looks something like this (you should be able to login with your standard account credentials):

If you get an error message along the lines of 
`Error: Can't open display: localhost:12.0`, 
make sure that you have connected to the server with X-forwarding enabled.
You can use the command `xeyes` to test your X-forwarding setup
(a fairly adorable pair of eyes should pop up).

When you're satisfied that JupyterHub is working, 
close Firefox and kill the JupyterHub server with `Ctrl-c`.

## Setting up JupyterHub as a non-root user

Although we currently have a working JupyterHub setup, 
it is not secure. 
The only way to run JupyterHub with multiple users at the momement is to run it as root.
This is a bad idea because any security vulnerability in JupyterHub would lead to your root account 
(and therefore your entire system) being compromised.
So let's be safe and setup JupyterHub as a non-root user.

First let's create a system user for JupyterHub that can't log in.

{% highlight bash %}
useradd -r jupyterhub
{% endhighlight %}

Users on our system will need to be able to start JupyterHub processes as this `jupyterhub` user.
Which is of course, a problem. 
No one can currently do this.
So, let's give them the capability.

Use `visudo` to edit the sudoers file.
Add the following to the end of the file.

{% highlight bash %}
Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/opt/python/anaconda3/bin/
jupyterhub ALL=(ALL) NOPASSWD:/opt/python/anaconda3/bin/sudospawner
{% endhighlight %}

This should allow all users to run JupyterHub, and only JupyterHub, as our `jupyterhub` user. 
You'll notice the `/opt/python/anaconda3/bin/sudospawner` command doesn't actually exist yet.
We'll install this one as well:

{% highlight bash %}
pip3 install git+https://github.com/jupyter/sudospawner
{% endhighlight %}

Ok, all set!
Let's test this out as our non-root user:

{% highlight bash %}
# this should work
sudo -u jupyterhub sudo -n -u $USER sudospawner --help  
# this should not
sudo -u jupyterhub sudo -n -u $USER echo 'fail'
{% endhighlight %}

## Authenticate using system passwords

To be able to authenticate users, JupyterHub must be able to read the shadow file 
(where user passwords are stored).
On Red Hat systems, there is no `shadow` group responsible for reading this file, 
so we'll have to create one and give `jupyterhub` access as a member of this group.

{% highlight bash %}
groupadd shadow
chgrp shadow /etc/shadow
chmod g+r /etc/shadow
usermod -aG shadow jupyterhub
{% endhighlight %}

## Create a systemd service for JupyterHub

Starting with version 7, CentOS and all Red Hat-flavored distros have made the switch to systemd.
If you don't like systemd, well, that's too bad 
(you're almost as bad as the Python 2 users...). 
For the unfamiliar, systemd is the new init system on Linux.
If we want our JupyterHub installation to start at boot and get managed like a normal Linux service, we'll need to create a systemd service for it.

A user-created systemd service is just a text file that lives in `/etc/systemd/system`. 
Let's create one under the path `/etc/systemd/system/jupyterhub.service`. 
Copy-and-paste the following into this file.

{% highlight bash %}
[Unit]
Description=A multi-user Jupyter notebook server

[Service]
User=jupyterhub
WorkingDirectory=/etc/jupyterhub
ExecStart=/opt/python/anaconda3/bin/jupyterhub \
	--JupyterHub.spawner_class=sudospawner.SudoSpawner \
	--config=/etc/jupyterhub/jupyterhub_config.py

[Install]
WantedBy=multi-user.target
{% endhighlight %} 

You'll notice that this config file mentions a directory on our system and several config files for JupyterHub that don't exist yet.
Let's create these now.

{% highlight bash %}
mkdir -p /etc/jupyterhub
cd /etc/jupyterhub
jupyterhub --generate-config
chown -R jupyterhub:jupyterhub /etc/jupyterhub
{% endhighlight %}

This creates a configuration folder (and working directory) under /etc/jupyterhub.
If you want to customize JupyterHub later on, you can edit the config file under 
`/etc/jupyterhub/jupyterhub_config.py`.
Likewise, to change how JupyterHub is started, edit `/etc/systemd/system/jupyterhub.service` and reload the systemd service.
Speaking of which, we will reload and start our JupyterHub service now.

{% highlight bash %}
# reload all systemd units
systemctl daemon-reload
# start jupyterhub
systemctl start jupyterhub
# enable starting jupyterhub whenever this machine reboots
systemctl enable jupyterhub
{% endhighlight %}

Excellent. 
If everything's been done properly so far, 
`systemctl status jupyterhub` should report that it's happily running on port 8000.
To check JupyterHub's logs, you can run `journalctl -u jupyterhub`.

## Setup HTTPS

It's important that when a user logs in, they don't send their passwords in plain text.
I don't think I really need to explain why for this one.

## Configuring a firewall

Now that you've got JupyterHub running successfully, 
let's install and configure a firewall for use with this program 
(if we haven't done so already).

{% highlight bash %}
yum install firewalld
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port=8000/tcp --permanent
firewall-cmd --reload
{% endhighlight %}

Double check and make sure you can still access JupyterHub from outside the system. 
If not, time to debug.

Assuming you've got this far, congratulations!
You are now the new owner of a brand-new JupyterHub installation.

Oh, and before we forget...

{% highlight bash %}
setenforce 1
{% endhighlight %}


