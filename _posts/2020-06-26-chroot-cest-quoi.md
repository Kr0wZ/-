---
layout: default
title: "Chroot c'est quoi ?"
date:   2020-06-26 11:45:30 +0200
published: true
categories:
  - Linux
---

### Ce que nous allons voir dans cet article :

*	[Chroot c'est quoi ?](#partie-théorique)
* 	[Mettre en place chroot à la main](#partie-pratique)
*	[Automatiser la tâche](#automatiser-la-tâche)
*   [Exemple concret avec SSH](#exemple-avec-ssh)

* * *

## Partie théorique

### Chroot c'est quoi ?

Chroot est une commande Linux qui permet d’isoler une partie du système du système "classique".                                           
Par exemple créer un répertoire chrooté spécial dans lequel les utilisateurs qui se sont connectés via SSH vont accéder.

Techniquement la commande chroot va créer un **environnement virtuel** dans lequel toutes les ressources système seront partagées avec l'hôte.

Chroot va donc changer le répertoire racine par celui qu’on a spécifié. De ce fait, les utilisateurs présents dans cette prison chroot ne pourront pas remonter plus haut dans l’arborescence. Ils ne verront et n'auront accès qu'au répertoire dans lequel ils sont chrooté.

### Pourquoi vouloir mettre en place chroot ?

- Tout dépend des besoins mais celà peut être pour une raison de **sécurité**, si jamais on a une application sensible qui risque d'être compromise. Dans le cas où quelqu'un réussi à pénétrer sur le système alors il ne pourra rien faire en dehors de la portée du chroot.

- On peut aussi vouloir avoir plusieurs **versions différentes** d'une même application. Dans ce cas chroot permet de ne pas avoir de conflits.

- Une autre utilisation serait de faire des tests avant de mettre un produit en **production**.

Bref vous l'aurez compris, les utilisations sont multiples en fonction de vos besoins.

* * *

## Partie pratique

Pour chaque commande qu’on va vouloir utiliser dans la **jail** (prison chroot) on va devoir copier le binaire en question, ainsi que toutes ses librairies associées.                             
Il faut ajouter dans la jail le strict minimum pour ce qu’on veut en faire.                                   
Certaines commandes sont déjà présentes comme cd ou pwd.

Pour se faire, nous allons utiliser la commande ```ldd``` qui va nous permettre de lister toutes les librairies utilisées par un binaire spécifique.

Par exemple si on veut pouvoir utiliser la commande ```ls``` dans notre chroot alors il va falloir copier le **binaire** ainsi que ses **librairies**.                                  
Pour **localiser** l'emplacement du binaire on va utiliser la commande ```whereis```. Voici le résultat de l'exécution sur *Ubuntu 18.04.4 LTS* :                                         

```bash
ls: /bin/ls /usr/share/man/man1/ls.1.gz
```

Ici ce qui nous intéresse c'est le ```/bin/ls``` qui est notre binaire. ```/usr/share/man/man1/ls.1.gz``` correspond au manuel d'utilisation de la commande lorsqu'on exécute ```man ls```

On connait désormais l'emplacement de notre binaire. On veut maintenant connaître ses librairies. On va donc exécuter ```ldd /bin/ls```. Voici le résultats (toujours sur *Ubuntu 18.04.4 LTS*) :

```bash
	linux-vdso.so.1 (0x00007ffc79f68000)
	libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f2197714000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2197323000)
	libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007f21970b1000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f2196ead000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f2197b5e000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f2196c8e000)
```

Tous ces résultats peuvent paraître barbares mais ils sont simples à interpréter :

* La première partie correspond au **nom de la librairie** utilisée par le binaire.
* La deuxième partie correspond à l'**emplacement** où se situe la librairie. C'est ce champ qui nous sera utile par la suite.
* Ce qui se trouve entre parenthèse correspond à l'**adresse mémoire** où a été chargée la librairie.

On peut voir cependant que la première et l'avant dernière ligne n'ont pas la même forme que les autres.                                             
Pour faire bref, 
* la première correspond à une librairie qui est **automatiquement chargée** peut importe le processus qui est lancé. Elle ne nous intéresse pas.                                                         
* L'avant dernière est directement *hardcodée* dans le programme et donc elle est affichée telle qu'elle. Elle nous **intéresse**.

Maintenant qu'on possède ces infos il ne nous reste plus qu'à créer notre **jail** et de copier tout ce qu'il faut dedans pour l'utiliser.

On commence par créer un répertoire à l'emplacement où on veut mettre en place notre **chroot** :

```bash
sudo mkdir -p /home/chroot_jail
```
On a vu précédemment que la commande ```ls``` se situe dans le dossier ```/bin/``` il faut donc également créer ce dossier :

```bash
sudo mkdir -p /home/chroot_jail/bin
```

De même, avec la commande ```ldd``` on a vu que les librairies sont situées dans les répertoires ```/lib/``` et ```/lib64/``` :

```bash
sudo mkdir -p /home/chroot_jail/lib
sudo mkdir -p /home/chroot_jail/lib64
```

On peut donc maintenant copier tout ce petit monde dans les dossiers correspondants :

```bash
sudo cp /bin/ls /home/chroot_jail/bin/
sudo cp /lib/x86_64-linux-gnu/libselinux.so.1 /home/chroot_jail/lib
sudo cp /lib/x86_64-linux-gnu/libc.so.6 /home/chroot_jail/lib
sudo cp /lib/x86_64-linux-gnu/libpcre.so.3 /home/chroot_jail/lib
sudo cp /lib/x86_64-linux-gnu/libdl.so.2 /home/chroot_jail/lib
sudo cp /lib64/ld-linux-x86-64.so.2 /home/chroot_jail/lib64 #Attention ici c'est bien /lib64/
sudo cp /lib/x86_64-linux-gnu/libpthread.so.0 /home/chroot_jail/lib
```

**Attention !** Il est fort probable que vous n'ayez pas les mêmes noms de librairie que moi. Faîtes attention au *copier/coller* !

A cette étape on pourrait se dire que tout est bon et qu'on peut partir à l'aventure dans notre chroot. Pas si vite malheureux ... Il reste le plus important : **le shell**

Pour le moment nous n'avons copié que la commande ```ls``` donc nous pouvons utiliser **QUE** celle-ci.                                                        
Pour que notre chroot puisse fonctionner il faut au **minimum** un shell. Dans notre cas on va utiliser ```/bin/bash```.

Vous avez deviné ? Eh oui ... il faut refaire la même manipulation ...

![](../../../../pictures/chroot/san-andreas.gif)

Il faut savoir que ce que je vous montre ici est vraiment le **strict minimum** pour faire fonctionner une jail chroot. Pour qu’elle soit plus fonctionnelle il faudrait rajouter les fichiers comme /dev/null, /dev/random, etc ...                                                         
Attention tout de même il faut toujours garder en tête qu'il faut mettre dans la jail uniquement ce qui est utile pour son bon fonctionnement. On ne va pas mettre Metasploit dedans si on veut juste faire des ping.                                                 

## Automatiser la tâche

Heureusement pour vous j'ai écrit un petit script bash qui vous permet de **facilement** créer une chroot à l'endroit souhaité avec les binaires voulus.                          
Voici le code :

```bash
#!/bin/bash

#Usage : ./chroot.sh <path> <commands>
#Example : ./chroot /srv/chroot bash id whoami ls
#Create a chroot at <path> 
#If <commands> is empty there will be only bash created in the chroot
#You must specify at least bash if <commands> isn’t empty
path=$1

mkdir -p $path

#Get every arguments except the first one
for command in $*
do
    if [ $command != $1 ]
    then
        location_full_path=`whereis $command | cut -d ' ' -f2`
        location=`echo $location_full_path | rev | cut -d '/' -f2-20 | rev`
        
        mkdir -p $path$location
        cp $location_full_path $path$location

        i=0

        #Read the command line by line
        ldd $location_full_path | while read -r lib;
        do
            #Skip the first line
            if [ $i -ne 0 ]
            then
                library=`echo $lib | cut -d '/' -f2-20 | cut -d ' ' -f1`
                library_location=/`echo $library | rev | cut -d '/' -f2-20 | rev`

                mkdir -p $path$library_location
                cp /$library $path$library_location

            fi
            ((i=i+1))    #i++

        done
    fi
done
```
Ce script peut également être retrouvé sur mon [Github](https://github.com/Kr0wZ/chroot.sh)

Je ne vais pas détailler chaque ligne mais ce qu'il fait c'est qu'il va **copier** tous les binaires ainsi que les librairies passés en paramètre dans l'emplacement de la chroot (qui sera lui même spécifié en paramètre).

Pour exécuter ce script (avec par exemple comme nom *chroot.sh*) rien de plus simple :

```bash
sudo chmod +x chroot.sh
sudo ./chroot.sh /home/chroot_jail bash ls id
```

**Attention !** Comme précisé plus haut il faut AU MOINS le binaire /bin/bash ou un shell fonctionnel (/bin/sh ou autre)

Si l'exécution du script s'est bien passé alors vous devriez avoir un nouveau dossier à l'emplacement spécifié. Dans l'exemple il se situe dans ```/home/chroot_jail```.                        
Maintenant que notre **jail** est créée il suffit d'y "plonger".

Pour y accéder il faut utiliser la commande ```chroot```.                                                                                   
Elle prend 2 arguments : le chemin vers le nouveau répertoire racine et le programme à exécuter (dans notre cas on veut ouvrir un shell dedans)

Avec l'exemple du dessus on exécute ceci :                                              
```bash
sudo chroot /home/chroot_jail bash
```

On va se retrouver dans notre chroot avec toutes les commandes disponibles que nous avons ajouté.                                               
Dans un répertoire normal, avec un ```pwd``` on devrait s'attendre à voir affiché ```/home/chroot_jail/```, sauf qu'ici on nous affiche ```/```.                                         
Comme expliqué dans la partie théorique, chroot va créer un environnement virtuel dans lequel on est à la racine. Impossible donc de remonter plus haut dans l'arborescence.

En théorie il est impossible de s'y échapper. Heureusement pour nous, comme c'est nous-même qui avons lancé la commande chroot il est possible de quitter cette jail avec ```exit```.

## Exemple avec SSH

Je peux vous montrer un exemple plus concret avec un utilisateur SSH qui atterrit dans la chroot lors de sa connexion.

On lance notre script (*sauf si vous aimez vous faire du mal en le faisant manuellement*) pour créer une **jail** dans ```/home/chroot_jail/``` :

```bash
./chroot.sh /home/chroot_jail bash ls cat
```

On veut que l'utilisateur puisse lister les fichier et lire leur contenu.

On va créer un utilisateur qui va être accessible via SSH et qui se **connectera directement** dans notre chroot sans possibilité de sortir du périmètre défini par celle-ci :
```bash
sudo useradd chroot_user -s "/bin/bash"
sudo passwd chroot_user
```

Il faut ensuite ajouter les lignes suivantes dans le fichier de configuration de SSH :
```bash
sudo nano /etc/ssh/sshd_config
```
```bash       
        Match User chroot_user
                    ChrootDirectory /home/chroot_jail
```

On redémarre le service ```sshd``` pour prendre en compte nos changements :

```bash
sudo systemctl restart sshd
```

Et enfin on tente de se connecter en SSH :

```bash
ssh chroot_user@localhost
```

Et hop ! On atterit dans notre chroot !

L'exemple a été fait en local mais évidemment il fonctionne également à distance.

J'espère que cet article a été compréhensible pour vous et qu'il vous aura aidé à mieux comprendre et à utiliser ```chroot```.


## Sources 

[https://www.journaldev.com/38044/chroot-command-in-linux](https://www.journaldev.com/38044/chroot-command-in-linux)

[https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/)

[http://doc.ubuntu-fr.org/chroot](http://doc.ubuntu-fr.org/chroot)

[https://fr.wikipedia.org/wiki/Chroot](https://fr.wikipedia.org/wiki/Chroot)

[http://www.linux-france.org/article/man-fr/man1/ldd-1.html](http://www.linux-france.org/article/man-fr/man1/ldd-1.html)

[https://stackoverflow.com/questions/34428037/how-to-interpret-the-output-of-the-ldd-program](https://stackoverflow.com/questions/34428037/how-to-interpret-the-output-of-the-ldd-program)