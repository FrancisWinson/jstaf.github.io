---
Setting up Linux for the office
---

So, you want to use Linux on your computer at work. 
I won't try to convince you to use it if you're not already convinced. 
Problem is, you can't afford to lose the ability to do anything. 
After all, what's the point of using Linux if it can't send emails or edit documents?
You absolutely *must* have the ability to do the same stuff as everyone else in your office.
And it needs to be easy. 

This has been a problem on Linux for a very long time - 
poor compatibility with common office software and file formats has really hampered adoption of the OS on the desktop. 
Even if something did work, the solution often required sysadmin level skills to implement.
Because of this, Linux hasn't been a viable OS for many people who weren't developers or sysadmins.
Only now, over the last year or two, 
have things improved to the point where stuff "just works" like it would on Windows or OSX.

This is a tutorial on setting up things so that all of the normal stuff you're used to simply works.
Not only that, but it needs to look and feel good. 
Usability is important! 
We need something that looks good, and works as well, 
if not better than what Microsoft or Apple has to offer.
Because of this, we are going to focus on GNOME, 
which is a "Desktop Environment" (DE) and software suite available on all major Linux distributions.
Though there are other great DEs out there, 
GNOME ships a consistent interface across all of its software, 
enjoys great compatibility with most office software/file formats, 
and is very stable.

# Which distribution do I use? 

There are a gajillion different variants of Linux out there. 
Which one(s) do I choose?
Fortunately, this one's an easy decision. 
Use either the latest version of Ubuntu GNOME or Fedora.

Okay, what's the difference between Ubuntu and Fedora? 
Why choose one over the other? 
Here are the biggest pros and cons of each:

**Ubuntu GNOME**

* It's easier
* Multimedia stuff doesn't need special setup
* Better support for videogames and Steam

** Fedora **

* Includes more up-to-date software
* Better compatibility with office software
* Fewer bugs

I personally use both on different computers. 
Ubuntu GNOME is easier to setup, but not everything works as well and there are more bugs.
On the other hand, Fedora "just works" with more office software right out of the box, 
but requires some hand-holding to setup some seemingly very simple things like playing an MP3.
If your priority is that everything "just works" with a minimum of fuss, pick Ubuntu GNOME, 
but if you're willing to deal with some extra setup, Fedora is a better choice. 

At this point, once you've picked a distribution, create a boot disk from USB and install it on your computer.
The installers are fairly self-explanatory, and there are no hidden tricks here.
If you want to setup Linux in a dual-boot configuration 
(so that you've got the option to use both Linux and Windows, for instance), 
install Windows first and THEN Linux (the Linux installer plays nice with other OSes,
Windows' does not).
Make sure you've backed up your data if you're doing this on a computer where you have important data.

After setting up your computer for the first time, 
make sure to look through the "Welcome to GNOME" tutorial that pops up the first time you login.
It's actually extremely informative and useful.

# Setting up home directory encryption

Does your computer have important data on it that could be potentially damaging if it was stolen?
If the answer to either question is yes, you need to setup disk encryption. 
It is an absolutely trivial task to steal someone's data off of a computer, 
regardless of whether or not it has a password. 
To illustrate how easy this is, 
you just need to plugin the Linux boot disk you just made into any computer, 
boot from the USB, 
and then you can copy any files you want off of the computer to your USB.
The only way to stop this is by encrypting your disk.

There are two options here: encrypt everything or just the stuff in your home directory.
In this tutorial, we will encrypt the home directory only.
The reason for choosing home directory encryption over full disk encryption 
is that it protects all of your important data (everything in the `/home` directory) 
without sacrificing computer performance.

## The easy way

While installing Ubuntu GNOME, select the option that says "Encrypt my home directory". You're all done (that was easy!)!

If you've already installed your OS or are using Fedora 
(which only gives the option for full-disk encryption), see below.

## The hard way

If you haven't already, create a root password so that you can log in as the root user.
To do this open a terminal and enter the following commands.
It should let you set a password for the "root user" (read: computer administrator).

{% highlight bash %}
sudo su -
passwd
{% endhighlight %}

Now restart your computer, select "Log in as another user", 
and log in as "root" using the password you just set.
Open up another terminal and enter the following 
(replace "yourUserName" with your user name):

{% highlight bash %}
dnf install keyutils ecryptfs-utils pam_mount
authconfig --enableecryptfs --updateall
usermod -aG ecryptfs yourUserName
ecryptfs-migrate-home -u yourUserName
su - yourUserName
ecryptfs-unwrap-passphrase ~/.ecryptfs/wrapped-passphrase
ecryptfs-insert-wrapped-passphrase-into-keyring ~/.ecryptfs/wrapped-passphrase
{% endhighlight %}

All done! 
You can now log out and log in as that user and your data under the `/home` directory will be encrypted.

# A couple quality-of-life fixes

Everything in this section can be enabled, disabled, 
and customized using the GNOME tweak tool (`apt` or `dnf` install `gnome-tweak-tool`).

Open up Firefox and go to [extensions.gnome.org](extensions.gnome.org). 
Search for and install the following extensions:

[Dash to Dock](https://extensions.gnome.org/extension/307/dash-to-dock/) - Adds a software dock to your screen that works much like the Windows Start Bar or Applications Dock on a Mac.

[Topicons Plus](https://extensions.gnome.org/extension/1031/topicons/) - Moves software tray icons to be with your system tray icons in the top right of your screen.

## Customize the user interface

You can install new icons or shell themes to change the way your user interface looks.
I recommend checking out the following:

* [Numix Circle icons](https://github.com/numixproject/numix-icon-theme-circle) - [Picture](http://me4oslav.deviantart.com/art/Numix-Circle-Linux-Desktop-Icon-Theme-414741466)
* [Moka icons](https://snwh.org/moka)
* [Arc window theme](https://github.com/horst3180/arc-theme)
* [Flat-Plat window theme](https://github.com/nana-4/Flat-Plat)

Pick and choose what you like, but these can make GNOME look even better than it already does 
(if you are unhappy with the default theme).

# Setup your web browser

There's nothing special about setting up a web browser on Linux, but here are a couple tips:

* Google Chrome is the only web browser that works with Netflix.
* Opera works the best if you have a touchscreen (like on a laptop).
* You can install uBlock Origin to get rid of popups and ads.

# Setting up online accounts and email


