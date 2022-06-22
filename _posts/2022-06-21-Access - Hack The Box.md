---
title: Access - Hack The Box
date: 2022-06-22
categories: [Hack The Box]
tags: []     # TAG names should always be lowercase
---

![Desktop View](/assets/hackthebox/access/access-infocard.png)

Access is an enjoyable Windows box marked as medium difficulty. We have anonymous access to the ftp server right from the start. The ftp server contains sensitive files, from which the attacker ultimately extracts login credentials for the exposed telnet service. The attacker thus gains the user's flag. In order to escalate the privileges, we use the *runas* command with the */savecred* option in order to use the saved Administrator credentials - this way we get the root flag.

## **Portscan**
---
```
Nmap scan report for 10.10.10.98
Host is up (0.051s latency).
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
23/tcp open  telnet?
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: MegaCorp
|_http-server-header: Microsoft-IIS/7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

## **Website**
---

Nothing interesting on the website. From dirbusting I got one path (aspnet_client/system_web) but it seems to be a rabbit hole and not worth enumerating.

![Desktop View](/assets/hackthebox/access/website.png)

## **Microsoft ftpd**
---
It is possible to log in to ftp anonymously. There are two folders in **/** directory.

    ┌──(root㉿kali)-[/home/…/pentesting/htb/boxes/access]
    └─# ftp 10.10.10.98
    Connected to 10.10.10.98.
    220 Microsoft FTP Service
    Name (10.10.10.98:emil): anonymous
    331 Anonymous access allowed, send identity (e-mail name) as password.
    Password: 
    230 User logged in.
    Remote system type is Windows_NT.
    ftp> ls
    425 Cannot open data connection.
    200 PORT command successful.
    150 Opening ASCII mode data connection.
    08-23-18  09:16PM       <DIR>          Backups
    08-24-18  10:00PM       <DIR>          Engineer

Two files were pulled from the ftp server. It should be mentioned that in order to download the backup.mdb file in its entirety, you have to switch to **binary mode** in the ftp client. To do that, just type *binary* into the console while being in ftp client.

    ┌──(root㉿kali)-[/home/…/pentesting/htb/boxes/access]
    └─# file Access\ Control.zip 
    Access Control.zip: Zip archive data, at least v2.0 to extract, compression method=AES Encrypted
                                                                                                                                
    ┌──(root㉿kali)-[/home/…/pentesting/htb/boxes/access]
    └─# file backup.mdb         
    backup.mdb: Microsoft Access Database

When we try to unzip the Access Control.zip file we get the following error.

    Archive:  Access Control.zip
    skipping: Access Control.pst      unsupported compression method 99

It is due to the fact that **unzip** does not support unzipping AES encrypted files at this time. Only the **7z** tool is able to do this. When trying to extract with the 7z utility, we get a message about the required password. At this point, we do not have a password - we have to get it somehow.  

### Backup.mdb
---
Let's try opening the downloaded backup using Microsoft Access. In **auth_user** table we find credentials.

![Desktop View](/assets/hackthebox/access/admin-pass.png)

We could have also found them by using `strings` on the *backup.mdb* file.

    ┌──(root㉿kali)-[/home/…/pentesting/htb/boxes/access]
    └─# strings backup.mdb | grep access4u@security   
    access4u@security

### Access Control.zip
---

Let's use the found password to unarchive the zip file.

    ┌──(root㉿kali)-[/home/…/pentesting/htb/boxes/access]
    └─# 7z x Access\ Control.zip

    7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
    p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-10600K CPU @ 4.10GHz (A0655),ASM,AES-NI)

    Scanning the drive for archives:
    1 file, 10870 bytes (11 KiB)

    Extracting archive: Access Control.zip
    --
    Path = Access Control.zip
    Type = zip
    Physical Size = 10870

        
    Enter password (will not be echoed):
    Everything is Ok         

    Size:       271360
    Compressed: 10870

As a result we obtained **Access Control.pst: Microsoft Outlook email folder**.

### Access Control.pst
---

Opening the *pst* file with Microsoft Outlook exposes the following message.

![Desktop View](/assets/hackthebox/access/mail.png)

Let's note the credentials

    security : 4Cc3ssC0ntr0ller

## Telnet - user shell
<hr>

Logging in to Microsoft Telnet Server with the obtained credentials. We are getting a low-level access as **security**.

![Desktop View](/assets/hackthebox/access/telnet-login.png)

Getting the user flag.

![Desktop View](/assets/hackthebox/access/user-flag.png)

## Runas - root shell
<hr>

Using the `cmdkey` command to list the stored credentials on the machine.

![Desktop View](/assets/hackthebox/access/cmdkey.png)

We can use `runas` to read the *root.txt* file, but let's create a more stable `netcat` session. Starting http server on the attacker machine and downloading `nc.exe` with `certutil` on the victim.

![Desktop View](/assets/hackthebox/access/file-transfer.png)

Starting to listen on port 9090 and running `nc.exe` as an Administrator.

![Desktop View](/assets/hackthebox/access/runas.png)

Reading the root flag.

![Desktop View](/assets/hackthebox/access/root-flag.png)