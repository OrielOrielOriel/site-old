---
layout: post
title: CTF Log - Year of the Jellyfish {THM}
description: A log of how I did the TryHackMe room Year of the Jellyfish by Muirland Oracle
summary: A log of the THM room "Year of the Jellyfish"
tags: ctf log
minute: 5
---
<br/>

> {{ page.description }}

<br/>

# Year of the Pig
## Password Guessing
The portscan results came back with just the ports: 22, 80. So I navigate to the web server and poke around. 

It looks like a simple blog a person named `marco`. They like planes. 

I'm able to manually find the `/admin/` directory which redirects me to a login page. After submitting incorrect credentials to capture a login request in Burpsuite, the page displays this message:


> Remember that passwords should be a memorable word, followed by two numbers and a special character.

I go back to read through the site's content and find that marco's favorite plane is the Savoia M.21. I was able to guess the password based on this information.

Once confirming that the password functions against the web server's login page. I check for password reuse against the SSH server and it works. This is why you don't reuse passwords folks. 

## Lateral Privesc & Website Database

From within the web server's admin panel, I notice that there's another user named `curtis`. Since I have access to the machine, I want to look through the website's database to get access to curtis's password hash. 

Since `www-data` is the only user with access to the `admin.db` file, I need to achieve command execution as the web server. Fortunately, marco has write privileges to PHP files actively being served by the website. 

I add `passthru($_GET["c"]);` to the end of `adduser.php`'s PHP code section, then use Burpsuite repeater to send a reverse shell to that endpoint. 

With a shell on the server as `www-data`, I'm now able to download the `admin.db` file. Using `strings` on it reveals curtis's password hash. I toss it an online rainbow table and get his password. 

Using marco's SSH session I'm able to `su` as curtis.

## Sudoedit Double Wildcard Filepath
The first thing I do with a session as curtis is sudo -l and what do you know, this is the output: 

```ansi
Matching Defaults entries for curtis on year-of-the-pig:                                                                                                                             
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR XFILESEARCHPATH XUSERFILESEARCHPATH"                                             
	
User curtis may run the following commands on year-of-the-pig:
    (ALL : ALL) sudoedit /var/www/html/*/*/config.php
```

So I use my shell as `www-data` to make a symbolic link to `/etc/shadow` at `/var/www/html/assets/fonts/config.php`. Sudoedit is then able to write to the `/etc/shadow` file. 

I replace the `root` hash with a known SHA512 and successfully root the box.