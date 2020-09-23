---
layout: post
title:  "TryHackMe (THM) Kenobi Writeup"
date:   2020-09-12 04:43:59
author: c0mp1ex
categories: Walkthroughs
tags:   ""
cover:  "/assets/images/Kenobi/kenobi-header.png"
---
This is my first writeup! Hope you enjoy

# Introduction

***

[Kenobi](https://tryhackme.com/room/kenobi) is an easy rated box with the tags **Samba**, **Path Var Manipulation**, **SUID**, **SMB**. By the looks of the tags, it's gonna be a pretty straight forward box. 

# Enumeration

***

First things first, I start off with an **nmap** scan:

![alt text](/assets/images/Kenobi/nmap-blog-001.png)

Right of the bat I seen some interesting things, we have NFS, SMB, HTTP and FTP which had a pretty old version.


After some google research on `ProFTPd 1.3.5`, I learned that unauthenticated users can run `SITE CPFR` which allowed me to copy the context of any file and paste that data into a newly created file or perexisting one with ` site CPTO`. This vulnerability can lead to sensitive data leaking, This information is important later on for getting user.

Shifting towards *samba*, I ran **smbmap** and discover a share called anonymous which I had read permissons to.

![alt text](/assets/images/Kenobi/smbmap-blog-001.png)

After connecting with **smbclient** and downloading log.txt and viewing it's content.
![alt text](/assets/images/Kenobi/smbclient-blog-001.png)
![alt text](/assets/images/Kenobi/logfile-blog-001.png)

I learned that the user kenobi had an RSA key in his home directory. Pairing this with the [ProFTPd 1.3.5 file copy exploit](https://www.exploit-db.com/exploits/36742) I was able to leak the RSA key.

The last part required for user came from the NFS. After I ran  **showmount**  on the NFS system.

![alt text](/assets/images/Kenobi/showmount-blog-001.png)

I made the grand discovery, that it allowed me to mount to the /var directory.

I mount the NFS to my /tmp directory with the following command:
`mount -t nfs 10.10.237.20:/var /tmp/`

Next I connected to FTP and moved the RSA key into the /var/tmp directory:

![alt text](/assets/images/Kenobi/ProFTPd-blog-001.png)

And after checking the directory I had an RSA key waiting for me:

![alt text](/assets/images/Kenobi/id_rsa_proof_blog_001.png )

Perfect! after running 

```console
chmod 4000 id_rsa
ssh Kenobi@10.10.237.20 -i id_rsa
```

![alt text](/assets/images/Kenobi/user-blog-001.png)

I had my user access!


# Privilege Escalation

***


First thing I did after getting user was change my directory to `/dev/shm` which is a great place to run enumeration scripts due to the fact that it deletes the data on shutdown/restart. This is very helpful with house cleaning. This happens because `/dev/shm` stores it's data to volatile ram(which deletes it's content when power is lost to the device), giving us the ability to avoid writing to disk(This makes it harder to detect our operations or any trail left behind). It also commonly doesn't have linux file premission restrictions for execution, allowing us to make a file and give it execution permissons.

I downloaded my trusty linpeas script from the [privilege-escalation-awesome-scripts-suite](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite), gave it execution permissions and ran it. After 2 minutes I went over the output and noticed an interesting binary with a root SUID. The SUID(Set-userID) allows a file to be executed as a specific user(in this case root) no matter who executes it.

![alt text](/assets/images/Kenobi/linpeas-blog-001.png)

I started to play around with the binary in question and after some light digging it's clear the program is calling binaries such as *curl* and *ifconfig*.

![alt text](/assets/images/Kenobi/menu-binary-blog-001.png)

Running **strings** on `/usr/bin/menu` confrimed the issue. *Curl*, *ifconfig* and *uname* are being called without an absolute path

![alt text](/assets/images/Kenobi/menu-binary-research-blog-001.png)
I chose to abuse **ifconfig** unstated path. I started by creating a file called *ifconfig* and gave it the contents of `/bin/bash`. After that I gave it execution permissions and changed the $PATH env variable so it would execute `/dev/shm/ifconfig` instead of the real one `/usr/bin/ifconfig`.

Then I called `/usr/bin/menu` and chose the ifconfig option which checked the modified **PATH** env variable and called `/dev/shm/ifconfig` instead of `/usr/bin/ifconfig` giving me a root bash shell.

![alt text](/assets/images/Kenobi/path-hijacking-blog-001.png)

Thanks for reading!
