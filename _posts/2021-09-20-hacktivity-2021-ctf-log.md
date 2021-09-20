---
layout: post
title: CTF Log - H@cktivityCon 2021 CTF
description: A log of the challenges I completed during the H@cktivityCon 2021 CTF
summary: A log of the H@cktivityCon 2021 CTF
tags: ctf log
minute: 15
---
<br/>

> {{ page.description }}

<br/>

# H@cktivityCon 2021 CTF
## Warmup Challenges
### Read the Rules
Challenge Author: @JohnHammond#6971  

A flag in the source code of the rules page. 

```html
<!--     Thank you for reading the rules! Your flag is:         -->
<!--        flag{90bc54705794a62015369fd8e86e557b}              -->
<!-- You will have to wait until the CTF starts to submit this! -->
```

### Six Four Over Two
Challenge Author: @JohnHammond#6971  

The challenge provided this encoded value:
```
EBTGYYLHPNQTINLEGRSTOMDCMZRTIMBXGY2DKMJYGVSGIOJRGE2GMOLDGBSWM7IK
```
The name of the challenge and the capital letters clued me into the fact that this is a base32 encoded string. 

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity]
└─$ echo -n 'EBTGYYLHPNQTINLEGRSTOMDCMZRTIMBXGY2DKMJYGVSGIOJRGE2GMOLDGBSWM7IK' | base32 -d
flag{a45d4e70bfc407645185dd9114f9c0ef}
 ```
 
### Tsunami
Challenge Author: @JohnHammond#6971  

The challenge provided a file named `tsunami`. Using the `file` command on it reveals that it is a WAVE audio file. 
 
I add the .wav extension to it and opened it in Audacity. After clicking on the *spectogram* view mode, the flag is revealed.
 
![/assets/media/hacktivity2021/tsunami.png|Flag shown as a spectogram]
 
### Pimple
Challenge Author: @JohnHammond#6971  

The challenge provided a file named `pimple`. Using the `file` command on it reveals that it is a type of image file. I open it in Gimp and see multiple layers of pictures of random people's faces. One of the layers has the flag in the image. 
 
![/assets/media/hacktivity2021/pimple.png|Flag in one of the image layers]
 
## Web
### Swaggy
Challenge Author: @congon4tor#2334  

The challenge provided Swagger documentation and an API with the `/flag` GET request endpoint. 
 
As shown below, performing the GET request results in the `missing authorization header` response. 
 
![/assets/media/hacktivity2021/swaggy1.png|Swagger API documentation]
  
I initially try the `admin:password` credentials, which doesn't work. My second guess is `admin:admin` which results in the server responding with the flag.
 
![/assets/media/hacktivity2021/swaggy2.png|Request with basic auth header and flag response]
 
### Confidentiality
Challenge Author: @JohnHammond#6971  

The challenge provided a website with a simple form field. Submitting the name of a file or directory as part of that form would return the output of the `ls -lsa` command using your provided input as the argument.
 
By submitting `/etc/hosts;cat flag.txt`, the underlying application performs the `ls -lsa /etc/hosts` command, then because of the `;` performs the `cat flag.txt` command.
 
![/assets/media/hacktivity2021/confidentiality.png|Showing the output of the aforementioned form submission]
 
## Forensics
### Bacon in a Haystack
Challenge Author: NightWolf#0268  

The challenge provided a bunch of traffic logs of different protocol types. The logs are shown below. 

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity/2021-09-08]
└─$ ls     
capture_loss.00:00:00-01:00:00.log  conn-summary.00:00:00-01:00:00.log  files.00:00:00-01:00:00.log           notice.00:00:00-01:00:00.log         reporter.03:53:19-03:53:19.log  stats.03:53:19-03:53:21.log
capture_loss.01:00:00-02:00:00.log  conn-summary.01:00:00-02:00:00.log  files.01:00:00-02:00:00.log           notice.01:00:00-02:00:00.log         software.02:15:51-03:00:00.log  stderr.02:15:41-03:53:21.log
capture_loss.02:16:42-03:00:00.log  conn-summary.02:00:00-02:01:50.log  files.02:00:00-02:01:50.log           notice.02:20:05-03:00:00.log         ssl.00:00:00-01:00:00.log       stdout.02:15:41-03:53:21.log
capture_loss.03:00:00-03:53:19.log  conn-summary.02:15:56-03:00:00.log  files.02:15:51-03:00:00.log           notice.03:00:00-03:53:19.log         ssl.01:00:00-02:00:00.log       weird.00:00:00-01:00:00.log
capture_loss.03:53:19-03:53:21.log  conn-summary.03:00:00-03:53:19.log  files.03:00:00-03:53:19.log           ocsp.00:00:00-01:00:00.log           ssl.02:00:00-02:01:50.log       weird.02:16:09-03:00:00.log
conn.00:00:00-01:00:00.log          conn-summary.03:53:19-03:53:21.log  http.00:00:00-01:00:00.log            ocsp.01:00:00-02:00:00.log           ssl.02:15:47-03:00:00.log       weird.03:00:00-03:53:19.log
conn.01:00:00-02:00:00.log          dns.00:00:00-01:00:00.log           http.01:00:00-02:00:00.log            ocsp.02:00:00-02:01:50.log           ssl.03:00:00-03:53:19.log       x509.00:00:00-01:00:00.log
conn.02:00:00-02:01:50.log          dns.01:00:00-02:00:00.log           http.02:15:51-03:00:00.log            ocsp.02:16:08-03:00:00.log           stats.00:00:00-01:00:00.log     x509.01:00:00-02:00:00.log
conn.02:15:56-03:00:00.log          dns.02:00:00-02:01:50.log           http.03:00:00-03:53:19.log            ocsp.03:00:00-03:53:19.log           stats.01:00:00-02:00:00.log     x509.02:15:48-03:00:00.log
conn.03:00:00-03:53:19.log          dns.02:15:46-03:00:00.log           known_hosts.02:15:46-03:00:00.log     packet_filter.02:15:42-03:00:00.log  stats.02:15:42-03:00:00.log     x509.03:00:00-03:53:19.log
conn.03:53:19-03:53:21.log          dns.03:00:00-03:53:19.log           loaded_scripts.02:15:42-03:00:00.log  reporter.00:00:00-01:00:00.log       stats.03:00:00-03:53:19.log
```

I started by looking through the DNS logs and saw a bunch of requests for random domains. After manually looking through the various domains I grep through the entire directory for a couple sites like `defcon.com` and `github.com`. 

The github grep returns DNS queries for `sketchysite.github.io`. I navigate there and am greeted by the flag. 

![/assets/media/hacktivity2021/bacon.png|Screenshot of flag at sketchysite.github.io]

## Miscellaneous
### Bad Words
Challenge Author: @JohnHammond#6971  

I connect to a listening port and am dropped in a restricted shell. The shell seems to filter a lot of common commands. I try invoking `/bin/bash` to go into a new shell and it works. 

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity/2021-09-08]
└─$ nc challenge.ctf.games 31873
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
user@host:/home/user$ id
id
You said a bad word, "id"!!
user@host:/home/user$ /bin/bash
/bin/bash
ls
just
cd just
ls
out
cd out
ls
of
cd of
ls
reach
cd reach
ls
flag.txt
cat flag.txt
flag{2d43e30a358d3f30fe65cc47a9cbbe98}
```

### Race Car
Challenge Author: @JohnHammond#6971  

This is a neat one, somewhat remeniscent of a level on OverTheWire's "Bandit" wargame. 

The challenge provides me with credentials and an address to ssh to, but the connection is closed immediately upon connecting. 

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity/2021-09-08]
└─$ ssh -p 32532 user@challenge.ctf.games
user@challenge.ctf.games's password: 
Connection to challenge.ctf.games closed by remote host.
Connection to challenge.ctf.games closed.
```

I tried a lot of different things and read a lot of different stackoverflow articles trying to figure out what the problem was. I initially thought it was something like `exit 0` or some kind of `pkill` command in the user's `.bashrc`, `.profile`, `.bash_profile`, or `.login` file. As a result I tried a lot of different on-connect `sed` commands to try to remove the suspected issue from those files. That didn't work, however. 

Eventually I wisened up and tried connecting over `sftp` and was able to poke around the filesystem from there. I found a [`rc` file](https://docstore.mik.ua/orelly/networking_2ndEd/ssh/ch08_04.htm) in the user's `.ssh/` directory with the following contents:

```bash
#!/bin/bash

pkill ssh
logout
exit
```

I used `put` to overwrite the file and was then able to ssh in with no issue. The privesc wasn't really a privesc, I just did `sudo su` to become `root` and get the flag.

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity/2021-09-08]
└─$ ssh -p 32532 user@challenge.ctf.games 
user@challenge.ctf.games's password: 
user@race-car-1744c696a30f0f35-f4fb64845-wjdhc:~$ sudo -l
Matching Defaults entries for user on race-car-1744c696a30f0f35-f4fb64845-wjdhc:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on race-car-1744c696a30f0f35-f4fb64845-wjdhc:
    (root) NOPASSWD: ALL
user@race-car-1744c696a30f0f35-f4fb64845-wjdhc:~$ sudo su
root@race-car-1744c696a30f0f35-f4fb64845-wjdhc:/home/user# cat /root/flag.txt
flag{f3deae2684d2bbec63d088374502a339}
```

### Redlike
Challenge Author: @JohnHammond#6971  

Didn't get any pastes or screenshots for this one. I got ssh access to a box and just needed to do the Redis privesc wherein you load a custom module that lets you execute commands as the user that Redis is running as. 

Here's a [link](https://book.hacktricks.xyz/pentesting/6379-pentesting-redis#load-redis-module) to that privilege escalation method.

### Shelle
Challenge Author: @fumenoid#9548  

This is another pseudo-shell / shell-escape challenge. I connected to the server and got a shell with a limited number of commands. The `/` character was restricted so I couldn't invoke `/bin/bash`. 

I was able to escape by typing `$SHELL` which, as a common environment variable mapping to the current shell, was analogous to `/bin/bash`. I poked around and found the flag in `/opt/flag.txt`.

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity/2021-09-08]                        
└─$ nc challenge.ctf.games 30414
Welcome to Shelle, a custom psuedo shell utility created by Professor Shelle in order to teach students about Linux terminals                                                                 
Shelle is a restricted environment to prevent any misuse, Please Enter 'HELP' to know about available features, happy learning !

root@pshelle$/bin/bash
/bin/bash
Illegal Character found, for safety reasons only certain characters are allowed

root@psuedoshell$$SHELL
$SHELL
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
bash: groups: command not found

shelle@shelle-cca5d5573c813293-5f5dbcd88b-zlq6s:~$ cat /opt/flag.txt
flag{82ad133488ad326eaf2120e03253e5d7}
```

## Scripting
### UHAHA
Challenge Author: *@JohnHammond#6971* 

They gave me a file named `uhaha`. By running `file` on it I learned that it was an Uharc archive file, which is an old compression algorithm.

The CLI tool for decompressing .uha files is only for Windows, but I'm able to run it on Kali using Wine. I make a bash for loop to throw the top 100 passwords from `rockyou.txt` at it and am greeted by another filed named uhaha, which, as you would guess, is another Uharc compressed archive. 

I used the following bash script to keep bruteforcing the decompression of Uharc archives as I suspect it's like a bajillion files deep. 

```bash
#!/bin/bash

for i in {1..10000};do
	for ii in `head -n 101 /usr/share/wordlists/SecLists/Passwords/Leaked-Databases/rockyou.txt`;do
		wine UHARC.EXE x -y -pw$ii target.uha;
		if ls | grep uhaha;then
			mv uhaha target.uha
			break
		fi
	done
done
```

Eventually I get a file named `flag.png`. 

![/assets/media/hacktivity2021/uhaha.png|image of flag]

I couldn't stop thinking about [this scene](https://www.youtube.com/watch?v=GY8EDRsRhik) from Finding Nemo while doing this challenge. *Sharkbait, Ooh Ha Ha!*

### Movie Marathon
Challenge Author: @Blacknote#1337  

I didn't actually finish this one because the IMDB API went down and I was too lazy to find another one (also they flagged my account). But basically you connect to a port and it gives you a movie and its release date and you have to respond with 5 actors that starred in that movie. 

```bash
┌──(kali㉿kali)-[~/Documents/ctf/hacktivity/uha]
└─$ nc challenge.ctf.games 30016
    __  ___           _         __  ___                 __  __              
   /  |/  /___ _   __(_)__     /  |/  /___ __________ _/ /_/ /_  ____  ____ 
  / /|_/ / __ \ | / / / _ \   / /|_/ / __ `/ ___/ __ `/ __/ __ \/ __ \/ __ \
 / /  / / /_/ / |/ / /  __/  / /  / / /_/ / /  / /_/ / /_/ / / / /_/ / / / /
/_/  /_/\____/|___/_/\___/  /_/  /_/\__,_/_/   \__,_/\__/_/ /_/\____/_/ /_/ 
                                                                            
You think you know movies more than I do? If you can send me any 5 cast members for each movie that I mention, I'll reward you.
e.g. The Avengers (2012-04-25)
Chris Evans; Robert Downey Jr.; Mark Ruffalo; Chris Hemsworth; Scarlett Johansson


> A Dog's Purpose (2017-01-19)
```

I wrote the following Python3 script for this task. 

```python
import requests
import nclib
import json
import time

KEY = 'APIKEY'
SEARCH_API = 'https://imdb-api.com/en/API/SearchMovie/'
CAST_API = 'https://imdb-api.com/en/API/FullCast/'

def receive_data(connection):
	data = connection.recv(1).decode("utf-8")
	print(data, end="")
	return data

def parse_movie(current_movie_raw):
	movie = current_movie_raw[1:].rstrip()
	ret = (movie, date) = movie.split("(")
	ret[1] = ret[1][:-1]

	return ret

def get_movie_details(current_movie):
	data = parse_movie(current_movie) 
	
	def get_movie_id(data):
		
		try:
			url = SEARCH_API + KEY + '/' + data[0] + data[1]
			r = requests.get(url)
			ret = json.loads(r.text)["results"][0]["id"]

			return ret
		except (IndexError, TypeError) as e:
			print("sleeping")
			time.sleep(30)
			get_movie_id(data)

	def get_movie_cast(id):
		try: 
			url = CAST_API + KEY + '/' + id
			r = requests.get(url)
			j = json.loads(r.text)
			ret = []

			for num in range(0,5):
				ret.append(unicodedata.normalize('NFD',j["actors"][num]["name"]).encode('ascii', 'ignore'))

			return ret

		except (IndexError, TypeError) as e:
			print("sleeping")
			time.sleep(30)
			get_movie_cast(id)

	id = get_movie_id(data)
	actors = get_movie_cast(id)

	return '; '.join(actors)

def send_movie_details(connection, details):
	connection.send(details)

def main():
	ADDRESS = 'challenge.ctf.games'
	PORT = 31260
	CURRENT_MOVIE = ""
	RECORD_MOVIE = False

	while True:
		connection = nclib.Netcat((ADDRESS, PORT))
		
		while not connection.eof:
			data = receive_data(connection)
			if RECORD_MOVIE:
				CURRENT_MOVIE += data
			if data == '>':
				RECORD_MOVIE = True
			if data == "\n" and RECORD_MOVIE:
				RECORD_MOVIE = False
				details = get_movie_details(CURRENT_MOVIE)
				print(details)
				send_movie_details(connection, str.encode(details))
				CURRENT_MOVIE = ""

if __name__ == '__main__':
	main()
```

A notable line is
```python
ret.append(unicodedata.normalize('NFD',j["actors"][num]["name"]).encode('ascii', 'ignore'))
```
where I normalize characters with accents since those seemed to give the listener trouble. 

## OSINT
### Don T. Mason
Challenge Author: @JohnHammond#6971

I just used the name "Don T. Mason" as a seed for a Spiderfoot scan and clicked through all of the website profile pages it generated. Eventually I came accross [this](https://mastodon.social/@donmason?max_id=106774436605922101) Mastodon page and when I saw all the elephants I knew I'd found it. I also knew I'd found it just when I saw the website name `mastodon`, since it's an ancestor of the elephant species. Also Don T. Mason is an anagram for Mastodon.

The flag was in one of the posts:

```
I guess there is no hiding it.

The trunk is used for breathing, bringing food and water to the mouth, and grasping objects. Tusks, which are derived from the incisor teeth, serve both as weapons and as tools for moving objects and digging. . Among African elephants, forest elephants have smaller and more rounded ears and thinner and straighter tusks than bush elephants and are limited in range to the forested areas of western and Central Africa.

flag{fbec1486e542c5d96f725cd6009ffef5}
```

### Mike Shallot
Challenge Author: @JohnHammond#6971  

I used "Mike Shallot" as a seed for a Spiderfoot scan and clicked through all of the website profile pages it generated. This led to me [this](https://pastebin.com/WVUP8dRD) pastebin:

```
This site is not as safe as we need it to be. 
Meet me in the dark and I will share my secret with you.
 
Find me in the shadows, these may act as your light:
 
strongerw2ise74v3duebgsvug4mehyhlpa7f6kfwnas7zofs3kov7yd
 
pduplowzp/nndw79
```

A shallot is an *allium* vegetable just like an *onion*. Which is a hint that the next portion of this hunt is going to be on a `.onion` website. 

I load up a TOR browser and visit the `.onion` URL provided in the paste. It's another Pastebin-style website. I use the smaller string `pduplowzp/nndw79` as a subdirectory and find the flag.

![/assets/media/hacktivity2021/shallot.png|screenshot of a paste containing the flag]

### Jed Sheeran
Challenge Author: @JohnHammond#6971  

I googled "Jed Sheeran" and find [this](https://soundcloud.com/user-836083929-176777888) Soundcloud profile. The flag is in one of the comments on one of the songs.

![/assets/media/hacktivity2021/jed.png|Picture of a comment with the flag in it]

# Personal Reflection
This was my first jeopardy style CTF and I had a ton of fun. I competed fairly casually, didn't pull any all-nighters or anything like that. 

I think it kind of awakened a flame in me for CTFs though because now I want to compete again next year, and more seriously. I think I might just do tons of CTFs for fun. 

For some reason video games stopped being interesting to me a couple years ago and I've honestly been looking for a fun de-stressing activity to replace it. CTFs might be it. 

I adore cooking but it doesn't have the same engrossing feeling that video games used to have for me. Or that CTFs seem to have for me now. So hell yeah. 

>**Thank you John Hammond.** 

![/assets/media/hacktivity2021/cert.png|A certificate of participation for the CTF, I placed 121/2527. I'll get top 10 next year :)]
