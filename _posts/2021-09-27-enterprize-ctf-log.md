---
layout: post
title: CTF Log - EnterPrize {THM}
description: A log of how I did the TryHackMe room EnterPrize by superhero1
summary: A log of the THM room "EnterPrize"
tags: ctf log
minute: 7
---
<br/>

> {{ page.description }}

<br/>

# EnterPrize
## Enumerating Web Presence
My initial portscan finds ports 22, 80, and 443 open. 

Checking the web server, I'm greeting a message `nothing to see here`. I add `enterprize.thm` to my `/etc/hosts` and begin to fuzz for subdomains and subdirectories. 

I find a loose `composer.json` file:
```json
{
    "name": "superhero1/enterprize",
    "description": "THM room EnterPrize",
    "type": "project",
    "require": {
        "typo3/cms-core": "^9.5",
        "guzzlehttp/guzzle": "~6.3.3",
        "guzzlehttp/psr7": "~1.4.2",
        "typo3/cms-install": "^9.5",
	"typo3/cms-backend": "^9.5",
        "typo3/cms-core": "^9.5",
        "typo3/cms-extbase": "^9.5",
        "typo3/cms-extensionmanager": "^9.5",
        "typo3/cms-frontend": "^9.5",
        "typo3/cms-install": "^9.5",
	"typo3/cms-introduction": "^4.0"
    },
    "license": "GPL",
    "minimum-stability": "stable"
}
```

This file proves important later.

The subdomain bruteforce returns a site at `maintest.enterprize.thm`, so I add that to my `/etc/hosts` and fuzz that site for subdirectories to find `fileadmin`, `typo3conf`, `typo3temp`, and `typo3`. 

## Typo3 Deserialization 
I was able to find the following `LocalConfiguration.old` file in the `/typo3conf/` subdirectory.

```php
<?php
	<--SNIPPED-->
    'SYS' => [
        'devIPmask' => '',
        'displayErrors' => 0,
        'encryptionKey' => '712dd4d9c583482940b75514e31400c11bdcbc7374c8e62fff958fcd80e8353490b0fdcf4d0ee25b40cf81f523609c0b',
        'exceptionalErrors' => 4096,
        'features' => [
            'newTranslationServer' => true,
            'unifiedPageTranslationHandling' => true,
        ],
        'sitename' => 'EnterPrize',
        'systemLogLevel' => 2,
        'systemMaintainers' => [
            1,
        ],
    ],
];
```

The relevant portion here will end up being the `encryptionKey`. Doing some research into the [Typo3](https://github.com/TYPO3/typo3) CMS and  the dependencies from the `composer.json` file led me to [this](https://www.synacktiv.com/publications/typo3-leak-to-remote-code-execution.html) disclosure by [Synacktiv](https://www.synacktiv.com/index.html). 

The context allowing for the vulnerability prerequisites (Knowledge of the `encryptionKey`), is different between their post and this situation, but regardless, I have access to the key so I continued to research the vulnerability. 

[For a deeper look into the source code enabling this vulnerability, give the blog post a read; it's well written and informative.](https://www.synacktiv.com/index.html)

>The *`initializeFormStateFromRequest()`* function in Typo3 will insecurely deserialize arbitary data if it is received in a POST request form body with a valid HMAC signature. 
>
>The *`"guzzlehttp/guzzle": "~6.3.3"`* dependency provides a gadget chain that can be used for remote code execution upon deserialization.

I use [`phpgcc`](https://github.com/ambionics/phpggc) to create a remote code execution payload which will write a webshell to the publically accessible `/fileadmin/_temp_/` directory. After signing it with an HMAC created with the `encryptionKey`, I send the malicious serialized payload in a POST form found on the website.   

Navigating to the webshell allows me to execute arbitrary commands on the underlying machine and I catch a reverse shell as `www-data`.

## Symlink and Malicious Library
While enumerating the machine as `www-data`, I find a custom binary named `myapp` in the `/home/john` directory. Looking around the filesystem reveals the following symlink:

```bash
/etc/ld.so.conf.d/x86_64-libc.conf -> /home/john/develop/test.conf
```

Using the `ldd` command on `myapp` shows the `/lib/libcustom.so` library as a dependency. This library contains the `do_ping()` function which I overwrite by compiling a malicious `do_ping()` function in a my own `libcustom.so` file which I store in the `/home/john/develop/` directory. 

After that I write `/home/john/develop/` in the `test.conf` file to make the `myapp` file load my malicious library and execute my `do_ping()` function. 

I use this exploit to copy my ssh public key into john's `authorized_keys` file and am now able to ssh as the user `john`.

## Local Port Forwading and NFS no_root_squash
Once authenticated as `john` I run *linpeas.sh* to find port 2049 is running, and that the `no_root_squash` parameter is enabled for `/var/nfs`. 

To exploit this vulnerability I use ssh to port forward `127.0.0.1:2049` to a port on my own machine and mount the `/var/nfs` share. 

On my machine, I copy `/bin/bash` as `root` and chmod it to have the SUID property. I copy it into the mounted directory and use run it with the `-p` flag on the victim machine to generate a shell as `root`.