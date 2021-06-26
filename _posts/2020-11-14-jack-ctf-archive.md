---
layout: post
title: Archive - CTF Writeup - Jack (THM)
description: An archive of an old writeup I wrote about the THM room "Jack"
summary: A writeup of the THM room "Jack"
tags: archive ctf writeup
minute: 8
---
<br/>

> {{ page.description }}

<br/>

# Introduction
As always, to skip straight to the writeup please use the `Contents Bar` on that right. --------->

[Jack](https://tryhackme.com/room/jack) is a CTF hosted on [TryHackMe](https://tryhackme.com). It involves basic Wordpress enumeration, Python Module Abuse, and Lateral Privilege Escalation.  
<br/>

> As per the instructions on the Jack room page, an entry for jack.thm was added to my /etc/hosts file. 

<br/>

# Reconnaissance

As always, real hackers hack time, and as I value my time when doing CTFs I like to start with Rustscan. The command `rustscan -r 0-65535 -u 5000 10.10.81.39 -- -A -oN scans/nmap.txt` blasts all TCP ports on the target machine to ascertain which ports are open, then automatically performs a Nmap scan with the options `A -oN scans/nmap.txt`.

![Screenshot of Rustscan Initial Screen](/assets/media/jack/Rustscan.png)

The open ports are:
 - 22
 - 80

In this case, the port that I am interested in is port 80. After navigating to the web page using my browser, I find what seems to be a typical blog. I quickly find that it is a Wordpress blog by checking the page source and seeing default Wordpress file paths. 

I send [wpscan](https://github.com/wpscanteam/wpscan) at the site using the command `wpscan --detection-mode aggressive --url jack.thm -e | tee scans/wpscan.txt` and it finds three usernames:
 - jack
 - wendy
 - danny
 
 I save those names to a file then perform a password attack using the command `wpscan --url jack.thm -U loot/users.txt -P /opt/wordlists/fasttrack.txt | tee scans/wpscan-users.txt`. After about a minute, it finds the password for the user `wendy`.
 
 ![Screenshot of WPScan Finding the Right Password](/assets/media/jack/WPScan-Valid-Password-Output.png)
 <br/>

# User
After logging into Wordpress as the user `wendy`, I find that I only have a regular user's Wordpress permissions. I know that achieving code execution as an Admin user would be trivial, so I try to search for ways to pivot to an Admin account or escalate my Wordpress user privileges.

Using [searchsploit](https://github.com/offensive-security/exploitdb/blob/master/searchsploit) to search for `wordpress privilege` finds me a variety of exploits. And after poking around at them I find one that works. It's titled `WordPress Plugin User Role Editor < 4.25 - Privilege Escalation` and has a Metasploit ruby module at `php/webapps/44595.rb`. 

From looking at the code, and reading the module description, the User Role Editor plugin has a problem with its authorization check when it updates a user's profile. This allows the user to arbitrarily edit their own profile to grant themselves administrator privileges. Further reading can be done [here](https://www.wordfence.com/blog/2016/04/user-role-editor-vulnerability/).

I go to edit my profile, change an arbitrary value like my first name, and then capture the POST request in Burp suite. After adding `ure_other_roles=administrator` to the request body and letting the request go through, the page refreshes and the `wendy` user account now has administrator privileges.

Here is what the captured request looks like, with the edit included:

```
POST /wp-admin/profile.php HTTP/1.1
Host: jack.thm
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://jack.thm/wp-admin/profile.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 318
Connection: close
Cookie: <--Snipped for Brevity-->
Upgrade-Insecure-Requests: 1

_wpnonce=473e4f45c7&_wp_http_referer=%2Fwp-a<--Snipped!-->file&ure_other_roles=administrator 
```

With administrator privileges, it is trivial to get command execution on the underlying server. I go to the plugins page, add a Linux PHP reverse-shell code to a deactivated plugin, then activate the plugin and catch a reverse shell with a Netcat listener. 

> Note: PHP Reverse Shells are OS-dependent. Don't get stuck trying to use a Linux PHP Reverse Shell on a Windows server or vice versa.

![Screenshot of Shell as www-data](/assets/media/jack/WWW-Shell.png)

I briefly enumerate the file system for anything out of place, and notice a file named `id_rsa` in the `/var/backups` directory. After looking at `/etc/passwd` and seeing the user entry for a user named `Jack` I connect the dots and copy the RSA private key over to my local machine. 

I then use `chmod 600 id_rsa.jack` to change the key file's permissions and `ssh jack@jack.thm -i id_rsa.jack` to SSH into the machine.

![Screenshot of chmod and Shell as Jack](/assets/media/jack/Jack-Shell.png)
<br/>

# Root
The `id` command is the first command I ever run when I enter into a Linux environment as a certain user for the first time. A user's group membership is something I like to keep in mind as I enumerate through different privilege escalation methods and as I interact with the file system in general. 

> The first command I run in a new Linux environment is `id`. In a Windows environment, `whoami /all` is usually the first command I run.

![Screenshot of id Command as Jack](/assets/media/jack/Jack-id.png)

In this case, the user `Jack` is part of the following groups: 
- jack
- adm
- cdrom
- dip
- plugdev
- lpadmin
- sambashare
- family

Given that this is a CTF, I presume that most of these groups are there for distraction, but the `adm` and `family` group memberships stick out to me. `adm` sticks out because it is often privy to usefol log files and `family` sticks out because it is a custom group; it is a group that has been manually created for some reason.

I use `find / -group family 2>/dev/null` to find what files belong to the `family` group and am surprised to see an output full of Python2.7 modules. Here's an excerpt of what that looked like:

```
/usr/lib/python2.7/weakref.py
/usr/lib/python2.7/sgmllib.pyc
/usr/lib/python2.7/os.py
/usr/lib/python2.7/posixpath.py
/usr/lib/python2.7/copy_reg.py                                                                             
/usr/lib/python2.7/bdb.py
```

With the ability to edit core Python2.7 modules, privilege escalation is trivial. I just need to find a program running as root that uses Python2.7. 

I dig around further and find the `/opt/statuscheck` directory. In this directory, there is a Python script called `checker.py` that imports the `os` module, which is a module I have the privilege to edit.

```
import os

os.system("/usr/bin/curl -s -I http://127.0.0.1 >> /opt/statuscheck/output.log")
```

By looking at the `output.log` file in the same directory, I can see that the script is run every 2 minutes. All I have to do is edit the `/usr/lib/python2.7/os.py` file and any code I add to it will be executed every two minutes, presumably as the `root` user. 

There are a lot of ways to escalate privileges, or just do anything, when you have arbitrary code execution. In this case, I decided to be lazy and simply added code to copy the root flag to a file in my user's directory. I appended `system("cat /root/root.txt >> /home/jack/root.txt;chmod 777 /home/jack/root.txt")` to the `os.py` file, and waited two minutes.

After the two minutes, I found the root flag is sitting patiently in my home directory.

![Screenshot of Cat /home/Jack/root.txt](/assets/media/jack/Root-Flag.png)
<br/>

# Take Aways

 - If a website is using a CMS, enumerate for common vulnerabilities and misconfigurations of that CMS. A lot of scanning tools exist that are specific to a CMS.
 - Check common backup locations, like `/var/backups/`.
 - Always enumerate based on your user's groups. 
 - Pay careful attention to groups which appear to have been custom-made.
<br/>
<br/>

# Tools
<br/>
 : [Rustscan](https://github.com/RustScan/RustScan)
 : [nmap](https://github.com/nmap/nmap)
 : [wpscan](https://github.com/wpscanteam/wpscan)
 : [FastTrack Word List](https://gitlab.com/kalilinux/packages/set/-/raw/2053787e02992b5cbf95192ae4b66a750bbcef7f/src/fasttrack/wordlist.txt)
 : [searchsploit](https://github.com/offensive-security/exploitdb/blob/master/searchsploit)
 : [Burp Suite](https://portswigger.net/burp)
 : [Linux PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)