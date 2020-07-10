---
layout: default
title: "Setup son propre serveur Owncloud"
date:   2020-07-09 10:00:00 +0200
published: true
---

### Ce que nous allons voir dans cet article :

*	[Introduction](#introduction)
*	[Préparation de la VM](#préparation-de-la-vm)
* 	[Installation](#installation)
*   [Sécurisation avec SSL](#mise-en-place-ssl)
*	[Comment s'en protéger ?](#se-protéger)
*   [A vous de jouer !](#a-vous-de-jouer)
* 	[Sources](#sources)

* * *


Dans cet article nous allons nous intéresser à mettre en place notre propre serveur Cloud sur notre machine pourqu'il soit accessible depuis n’importe où sur Internet. On va voir pas à pas chaque étape pour que vous puissiez le reproduire chez vous.

Juste avant quelques infos ainsi que des prérequis.

## Introduction

Le modèle de Cloud que nous utilisons est de type **Saas** (Software As A Service) qui veut dire que les applications pour le Cloud nous sont déjà fournies. Son nom est Owncloud.                     
Il sera installé sur une **machine virtuelle** car plus facile à gérer si on a un problème où si on a mal fait quelque chose. On utilisera **VirtualBox** (juste une question de préférence). Mais vous pouvez tout aussi bien utiliser **VMWare** ou tout autre machine virtuelle. 

Il vous faudra également une image disque d’**Ubuntu** pour la VM car on va installer et configurer Owncloud sur une version **Linux**. C’est gratuit et vous pourrez retrouver des images sur le **[site](https://ubuntu-fr.org/telechargement)** de la communauté francophone d’Ubuntu.

Vous verrez qu’avec Owncloud vous pouvez faire énormément de choses. C’est une bonne alternative aux différents Cloud qui vous proposent leurs services puisqu’ici tout est sous votre contrôle. Vos données sont stockées chez vous.

Assez de blabla ! On va commencer à installer tout ce qu’il nous faut. Sachez que dans ce tutoriel j’utilise des logiciels spécifiques (comme VirtualBox par exemple) mais qu’il n’y a pas uniquement ces logiciels qui peuvent être utilisés pour mettre en place un serveur Cloud. Il y a toujours d’autres **alternatives**. Pour avoir plus d’infos si un logiciel est compatible ou non, vous pouvez toujours vous rendre sur le site officiel d’Owncloud dans l’onglet **Documentation** pour avoir la fiche technique du logiciel.

## Préparation de la VM	

Il faut commencer par se rendre sur le site de VirtualBox, et de télécharger le programme.                                                                            
On l’installe, on crée une nouvelle VM avec l’image disque Ubuntu.                                                                                                     
**Très important !** On se rend dans l’onglet réseau et on ajoute une **carte par pont** pour voir la VM sur le réseau et donc pour pouvoir, par la suite, rediriger les ports vers elle et une carte réseau **NAT** pour avoir accès à Internet ainsi que réussir à atteindre notre Cloud depuis Internet.

On se connecte à votre box (Livebox pour moi, depuis l'interface Web) pour **rediriger les ports** vers votre VM pour y avoir accès depuis l'extérieur du réseau local (port **80** et **443**).            
**ATTENTION** de bien activer le **mode promiscuité** pour les VM (à faire dans virtualbox onglet réseau).

Nous venons de configurer l’environnement de travail, on va maintenant mettre en place le serveur cloud ainsi que tous les éléments nécessaires à son bon fonctionnement.                             
Toute l’installation et la configuration du serveur se fait en **CLI** (lignes de commande).    

Il y aura juste la configuration du **DNS** à la fin qui se fera par un site. Cette configuration ne sera nécessaire uniquement si votre **IP est dynamique**. Mais on en reparlera plus tard.

On peut désormais allumer votre VM et faire tranquillement l’installation d’Ubuntu.

Une fois l’installation faite, on ouvre un nouveau terminal et on commence à taper les petites commandes toutes belles.


## Installation 	

La toute première chose à faire avant de commencer à installer le serveur est de vérifier que les packages installés sont à jour avec les commandes :

```bash
apt-get update
apt-get upgrade
```

Il faut ensuite commencer à installer les **composants indispensables** au serveur Owncloud :

```bash
apt-get install apache2 php5 php5-mysql
```

Une fois que l'installation de ces services est terminée il faut installer les **modules** PHP suivants indispensables à Owncloud :

```bash
apt-get install php5-gd php5-json php5-curl php5-intl php5-mcrypt php5-imagick
```

Installation de **MySQL** qui nous servira de base de données autant pour les données du cloud qui sont stockées que les comptes des utilisateurs inscrits sur le cloud :

```bash
apt-get install mysql-server
```

Maintenant que MySQL est installé il faut **créer une nouvelle base de données** pour Owncloud en utilisant les commandes suivantes :

```bash
mysql -u root -p
Enter password :

mysql> CREATE USER 'ownclouduser'@'localhost' IDENTIFIED BY 'root' ;
mysql> CREATE DATABASE ownclouddb ;
mysql> GRANT ALL ON ownclouddb.* TO 'ownclouduser'@'localhost' ;
mysql> FLUSH PRIVILEGES ;
mysql> exit ;
```

La base de données est créée il faut à présent télécharger et décompresser la dernière **version stable** d'Owncloud :

```bash
wget https://download.owncloud.org/community/owncloud-9.0.2.tar.bz2
tar -xvf owncloud-9.0.2.tar.bz2 -C /var/www/html/
```

On administre les permissions sur le dossier que l'on vient de décompresser.

```bash
chown www-data:www-data -R /var/www/html/owncloud/
```

Jusqu'à présent nous n'avions pas encore configuré Apache. Nous allons le faire maintenant :

```bash
nano /etc/apache2/sites-available/owncloud.conf
```
```bash
<IfModule mod_alias.c>
Alias /owncloud /var/www/html/owncloud
</IfModule>
<Directory “/var/www/html/owncloud”>
Options Indexes FollowSymLinks
AllowOverride All
Order allow,deny
allow from all
</Directory>
```

Pour que la configuration soit prise en compte il ne faut pas oublier de relancer les services liés à Apache :

```bash
service apache2 restart
``` 

On se rend dans ``/var/www/html/owncloud/config/config.php`` pour ajouter dans ce fichier la liste des **domaines approuvés** qui ont le droit de se connecter.                                     
On a donc normalement par défaut la loopback qui correspond à votre ordinateur actuel soit ``127.0.0.1``. On va rajouter en dessous l’adresse de notre IP publique.

Owncloud doit maintenant être accessible via un navigateur internet en rentrant par exemple : ``http://ip_publique/owncloud`` ou avec un nom de domaine généré par un DNS mais nous verrons la partie DNS plus tard.

Le serveur est également **accessible en local** sur la machine installée. Pour y accéder il faut se rendre sur un navigateur et taper : ``localhost/owncloud``

On va maintenant s’occuper du DNS pour que ce soit plus facile d’accès.                                                            
Le DNS c’est ce qui va "convertir" une adresse IP en un nom d’hôte et inversement. Par exemple pour aller sur google vous ne tapez pas ``216.58.206.3`` ? Vous tapez ``google.fr``. C’est le DNS qui va se charger de faire la correspondance entre ``google.fr`` et son adresse IP.                                         

A savoir que le DNS est utile pour les adresses **IP dynamiques**. C’est a dire qu’elles changent régulièrement contrairement à une IP statique qui, elle, reste toujours la même.                     
Vous pouvez toujours en utiliser un pour une IP statique pour faciliter l’accès.

Pour ce faire, on se rend sur le site **[NoIP](https://www.noip.com/)** qui propose des **DNS dynamiques gratuits** mais qui expirent au bout d’un mois. Il faudra donc les renouveler régulièrement.    Vous pouvez toujours prendre un abonnement si vraiment c’est quelque chose d’utile et important pour vous ou bien vous pouvez aussi trouver un autre site qui vous permettra de faire vos DNS dynamiques.

On se crée un nouveau compte avec des informations *bidons* sauf le mail et le mot de passe qui seront utiles pour la suite. On va sur notre **compte** dans la barre en haut.                           
On arrive ensuite sur une page avec à notre gauche plusieurs options. On va cliquer sur **Dynamic DNS**. On peut suite créer notre premier DNS dynamique.                                       
On choisit un **nom**, une **extension** puis on va choisir **Web redirect** en précisant l’adresse **IP publique** avec ``/owncloud`` derrière.

Il y a possibilité de masquer l’IP en arrivant sur la page et de remplacer par un titre.

Il faut ensuite se rendre sur l’interface de notre box et dans l’onglet **DynDNS** (en tout cas pour une Livebox), remplir le champ nécessaire.

## Mise en place SSL

Nous allons maintenant faire fonctionner Owncloud à travers une **couche SSL** en transformant les échanges du port 80 (http) en 443 (HTTPS). Cela permet de **sécuriser les communications** entre le serveur et les clients.

Sous Apache2, nous allons commencer par créer un **VirtualHost** avec une configuration qui fonctionnera avec le HTTPS (port 443).                                            
Dans le répertoire ``/etc/apache2/sites-available``, on crée un nouveau fichier ``owncloudhttps.conf`` et on saisit les lignes suivantes :

```bash
NameVirtualHost *:443
# Hôte virtuel qui écoute sur le port HTTPS 443
<VirtualHost *:443>
DocumentRoot /var/www/
# Activation du mode
SSLEngine On
SSLOptions +FakeBasicAuth +ExportCertData +StrictRequire
# On indique ou est le certificat
SSLCertificateFile /etc/ssl/certs/owncloud.crt
SSLCertificateKeyFile /etc/ssl/private/owncloud.key
</VirtualHost>
```

Nous allons maintenant activer le module SSL dans Apache2 pour que celui-ci puisse être
utilisé dans nos échanges Owncloud :

```bash
a2enmod ssl
```

On doit ensuite ajouter notre nouveau site aux sites actifs d’Apache2 :

```bash
a2ensite owncloudhttps.conf
```

Nous avons pratiquement terminé la sécurisation par HTTPS il ne reste plus qu’à générer un **certificat auto-signé**. Le problème c’est qu’on peut encore accéder via le http. 
Il faut donc **forcer le HTTPS**. Pour ce faire, il suffit de se rendre dans ``/var/www/html/config/config.php`` et d’ajouter la ligne ``force ssl => true``.                                                                   
On passe alors à la **génération** de nos clés, on crée un répertoire pour les clés :

```bash
cd /etc/apache2 && mkdir CertOwncloud && cd CertOwncloud
```

On génère notre clé sur 1024 bits :

```bash
openssl genrsa –out owncloud.key 1024
```

Création des .keys et .csr :

```bash
openssl req –new –key owncloud.key –out owncloud.csr
``` 

On remplit ensuite les données du certificat et on crée le fichier du certificat :

```bash
openssl x509 –req –days 365 –in owncloud.csr –signkey owncloud.key –out owncloud.crt
```

On copie ensuite nos certificats là où tous les autres sont stockés :

```bash
cp owncloud.crt /etc/ssl/certs
cp owncloud.key /etc/ssl/private
```

Il ne reste plus qu’à redémarrer Apache2 après avoir vérifié sa configuration :

```bash
apachectl configtest
service apache2 restart
```

Voilà vos données sont désormais **chiffrées** de part en part de la communication entre vous et le cloud. Il y a cependant un petit *hic* et je pense que certains d’entre vous l’auront remarqué. 

Si on éteint notre VM ... il se passe quoi ? Et bien nous n’avons tout simplement plus accès à notre Cloud.                                  
Une solution serait d'utiliser Raspberry Pi pour héberger votre Cloud dessus et le laisser branché.

Il existe également des VPS gratuits ou payants qui offriront une meilleure disponibilité que votre Raspberry (en cas de panne par exemple).
