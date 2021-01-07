---
layout: default
title: "Writeup CengBox: 3 - Vulnhub"
date:   2020-12-28 10:00:00 +0200
published: false
---

# Informations sur la box :

* **Difficulté** : Intermédiaire / Difficile
* **Description** : *Some of us hold on to poems, songs, movies, books. I guess people can't hold onto people anymore. -- Oguz Atay                                                                    
You know what you have to do. If you get stuck, you can get in touch with me on Twitter. @arslanblcn_*
* **Type de fichier** : .ova (VirtualBox)
* **DHCP** : Activé
* **Lien** : [https://www.vulnhub.com/entry/cengbox-3,576/](https://www.vulnhub.com/entry/cengbox-3,576/)


### Ce que nous allons aborder dans ce writeup :

*	[Énumération](#énumération)
* 	[Virtual Host fuzzing](#virtual-host-fuzzing)
*	[Admin panel via brute force](#admin-panel-via-brute-force)
*   [Admin panel via SQL Injection](#admin-panel-via-sql-injection)
*   [Shell basique](#shell-basique)
*   [Privesc utilisateur](#Privesc-utilisateur)
*   [Privesc root - 1ère méthode](#privilege-escalation)
*   [Privesc root - 2ème méthode](#privilege-escalation)

* * *

## Énumération
* * *

Comme à chaque début de box il faut trouver son adresse IP avec ``nmap`` en utilisant l'option ``-sP`` (**Ping Scan**) :

```bash
nmap -sP 192.168.1.0/24
```

On obtient l'IP : **192.168.1.109**

On va donc énumérer les services et ports ouverts sur la machine à attaquer :

```bash
nmap -p- 192.168.1.109 -A --min-rate 5000
```

* **-p-** : On scan TOUS les ports
* **-A** : Mode aggressif
* **--min-rate 5000** : Définit le nombre d'envoi de paquets par secondes

Pour plus d'informations vous pouvez consulter le ``man`` de nmap.

Voici les informations que l'on obtient :

```
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Agency
443/tcp open   ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Agency
| ssl-cert: Subject: commonName=ceng-company.vm/organizationName=Ceng Company/stateOrProvinceName=\xC3\x84Istanbul/countryName=TR
| Not valid before: 2020-09-24T14:25:07
|_Not valid after:  2021-09-24T14:25:07
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
``` 
On se rend compte que ce sont les **ports basiques** qui sont ouverts. Aucun port exotique. On remarque aussi que **SSH** port **22** est fermé et donc que le shell s'obtiendra exclusivement via la partie web.


Un site web tourne sur la machine sur le port **80**. 

On va se rendre dessus via le navigateur mais avant on lance un fuzzing de directory pour voir si on trouve des choses utiles :

```bash
ffuf -u http://192.168.1.109/FUZZ -w /usr/share/dirb/wordlists/directory-list-2.3-big.txt -e .php,.txt,.html
```

On recherche également de potentiels fichiers qui auraient pour **extension .php, .txt et .html** avec l'option ``-e``.

```
index.html              [Status: 200, Size: 35995, Words: 2110, Lines: 814]
img                     [Status: 301, Size: 318, Words: 20, Lines: 10]
css                     [Status: 301, Size: 318, Words: 20, Lines: 10]
js                      [Status: 301, Size: 317, Words: 20, Lines: 10]
fonts                   [Status: 301, Size: 320, Words: 20, Lines: 10]
poem.txt                [Status: 200, Size: 5, Words: 1, Lines: 1]
```

Les pages et répertoires trouvés ne semblent pas d'être d'un grand intérêt. 

Seul **poem.txt** paraît un peu **suspect** mais on y reviendra plus tard. Il ne contient que *Test1* et pas de possibilité d'intéragir avec puisse que c'est un ``.txt``.

Il existe exactement le **même site** mais sur le **port 443** avec **SSL**. La première faille qui m'est venue à l'esprit était **Heartbleed** qui est une vulnérabilité impactant SSL et dans les CTFs quand on tombe sur du SSL c'est pas étonnant d'en trouver. Mais ici ce n'est pas le cas.

Un petit scan avec **Metasploit** nous indique que la cible n'est **pas vulnérable** à cette attaque. A savoir que j'ai utilisé Metasploit pour la simplicité et rapidité mais il existe également d'autres scripts qui font le même travail.

```bash
msfconsole #On lance d'abord le Framework Metasploit
```

```bash
use auxiliary/scanner/ssl/openssl_heartbleed #On choisit le module
set RHOSTS 192.168.1.109 #On spécifie l'adresse de la cible
run #On lance le scan
```

J'ai également tenté du fuzzing web sur le site en **HTTPS** mais les **mêmes résultats** que pour le port 80 ressortent.

Comme le site est en **HTTPS** on peut aller inspecter son **certificat**. On trouve alors cette ligne :

```
Common Name : ceng-company.vm
```

Ça ressemble quand même beaucoup à un **nom de domaine**. Je l'ai donc ajouté au fichier ``/etc/hosts`` de ma machine et j'ai tenté d'accéder de nouveau au site sur les deux ports non plus via l'IP de la box mais par ce nom de domaine. Malheureusement **aucun changement** n'a été apperçu. J'ai même tenté de relancer un fuzzing avec ``fuff`` sur les deux sites mais rien de plus.


Après avoir tourné pas mal de temps j'ai décidé de retourner inspecter plus **en profondeur** le site web.

Au niveau de la présentation de l'équipe lorsqu'on passe la souris sur la photo des personnes, les **réseaux sociaux** apparaissent. Certains sont inexistant mais d'autres nous amènent vers des choses intéressantes.

Notamment **Eric Thompson** qui permet d'accéder à son [compte Github](https://github.com/arslanblcn/PHP-OOP) mais aussi **Elizabet Sky** qui possède un compte [Reddit](https://www.reddit.com/user/ElizabethSky_)


Sur le profil Reddit on a accès à un **poème** écrit par **Nazim Hikmet**. On sait qu'on a accès à ``poem.txt`` sur le site web. Peut-être qu'il y aura quelque chose à faire avec ces deux choses.

Le poème est le suivant :

```
[POEM]The Walnut Tree

   My head foaming clouds, sea inside me and out   I am a walnut tree in Gulhane Park   an old walnut, knot by knot, shred by shred   Neither you are aware of this, nor the police

   I am a walnut tree in Gulhane Park   My leaves are nimble, nimble like fish in water   My leaves are sheer, sheer like a silk handkerchief   pick, wipe, my rose, the tear from your eyes   My leaves are my hands, I have one hundred thousand   I touch you with one hundred thousand hands, I touch Istanbul   My leaves are my eyes, I look in amazement   I watch you with one hundred thousand eyes, I watch Istanbul   Like one hundred thousand hearts, beat, beat my leaves

   I am a walnuttree in Gulhane Park   neither you are aware of this, nor the police

Nazim Hikmet
```
Encore une fois on reviendra dessus **plus tard** car on ne sait pas quoi en faire pour le moment.

Concernant le Github de notre ami Eric on peut y retrouver des choses **très intéressantes**. <br>
Premièrement le **README.md** indique ceci :

```
Hint for CengBox3 -- namelastname@ceng-company.vm
```

Ce qui nous indique que nous sommes sur la bonne piste. On peut aussi voir la **structure d'une adresse mail** pour l'entreprise qui possède le site web.
De ce fait, par exemple le mail d'Eric sera ``ericthompson@ceng-company.vm`` et celui D'Elizabeth ``elizabethsky@ceng-company.vm``.

Un **code PHP** est également présent :

```php
<?php
    class poemFile
    {
        public function __destruct()
        {
            file_put_contents($this->filename, $this->poemName, FILE_APPEND);
        }
    }

    class Poem
    {
        public $poemName;
        public $isPoetrist;
        public $poemLines;

        public function printData()
        {
            if($this->isPoetrist){
                echo $this->poemName . "\n";
                echo $this->poemLines . "\n";
                echo "\n Poem has been added!";
            } else {
                echo $this->poemName ." is not a poem\n";
            }
        }
    }
    $object = unserialize(urldecode($_REQUEST['data']));
    $object->printData();
?>
```

On observe dans ce code qu'il y a de la **désérialisation** sans **aucune mesure de sécurité** au niveau de la donnée. Donc potentiellement un **vecteur d'attaque** de ce côté.<br><br>
On voit aussi qu'il y a **deux classes**. <br><br>
La première *poemFile* utilise la fonction ``__destruct()`` qui sera appelée lors de la destruction de l'objet. A ce moment cette fonction va insérer dans le fichier ``filename`` le nom du poème (``poemName``). Ces données seront sûrement récupérées depuis l'objet qui sera passé à ce fichier. <br><br>
La deuxième classe *Poem* vérifie simplement que le champ ``isPoetrist`` n'est **pas vide** et va afficher un message en fonction.

Ce sera sûrement un morceau de code à **exploiter**.

On garde ces infos de côté pour plus tard et on continue notre **énumération**. <br>

J'ai décidé de chercher sur **Google** le pseudo ``ElizabethSky_`` pour voir si on ne pouvait pas récupérer des infos en plus. Je suis tombé sur un compte [Twitter](https://twitter.com/elizabethsky_) avec des messages étranges mais c'est sûrement une fausse piste.

J'ai été un peu curieux et au lieu de me contenter du repository Github qu'on m'avait donné je suis allé voir dans les autres disponibles et je suis tombé sur [**celui-ci**](https://github.com/arslanblcn/PhpLogin).<br>

En regardant les fichiers je suis tombé sur des choses plutôt intéressantes :<br>

**setup.php**
```php
$passwd = md5("Sup3rS3cr3tPasswd!");
$insertQuery = "INSERT INTO users (name, email, passwd) VALUES ('John Doe', 'johndoe@example.com', '$passwd'),
                ('Elizabeth Sky', 'elizabethsky@example.com', '$passwd')";
```

On retrouve bien notre amie Elizabeth Sky avec un **mot de passe stocké** dans la base de données. Ce sera peut-être utile pour plus tard.<br>

**conn.php**
```php
$server = "localhost";
$db_user = "elizabeth";
$db_passwd = "elizabeth";
$db_name = "cengbox";
```

Qui indique le nom d'utilisateur et le mot de passe pour se connecter à la base de données.

**login.php**
```php
if(isset($_POST['username']) && isset($_POST['passwd'])){

    $username = validate($_POST['username']);
    $passwd = validate($_POST['passwd']);
    $passwd = md5($passwd);

    $query=$conn->prepare("SELECT * FROM users WHERE email= ?");
    $query->bind_param("s", $_POST['username']);
    $query->execute();
    $result = $query->get_result();
    $users = $result->fetch_assoc();
```

Ici on pourrait avoir une SQLi ? Aussi à garder pour plus tard.


* * *

## Virtual Host fuzzing
* * *

A cet instant j'étais un peu bloqué puisque j'avais pas mal d'informations mais rien de concret à exploiter sur aucun des deux sites web.

Grâce à une aide extérieure, on m'a donné comme indice de regarder du côté des **hôtes virtuels**.

J'ai donc cherché sur Internet et je suis tombé sur ce [très bon article](https://erev0s.com/blog/gobuster-directory-dns-and-virtual-hosts-bruteforcing/) qui expliquait les différents fuzzing possibles avec **Gobuster**.<br>

On peut y avoir un mode **VHOST** qui correspond à Virtual Host. C'est ce qu'on cherche !<br>

Tout d'abord petit point sur ce qu'est un Virtual Host. <br>

Comme nous l'indique très bien la [documentation officielle d'Apache](https://httpd.apache.org/docs/2.4/vhosts/), il s'agit de faire fonctionner **plusieurs serveurs web sur une même machine**. 
En effet, un serveur web ne possède qu'une adresse IP et donc en théorie on ne peut avoir qu'une seule version d'un site sur celui-ci. Les VHOSTS sont là pour remédier à ce problème. On accèdera donc à ces différents sites via un **nom de domaine différent**.

**Attention** cependant à ne pas confondre le **fuzzing DNS** avec le **fuzzing VHOST**.<br>

Dans le premier cas l'outil va essayer de **résoudre le sous domaine** tandis que dans le second il va essayer d'accéder à l'URL pour **vérifier que l'adresse IP est identique**.<br>

Seulement, l'article ne parle que de ``Gobuster`` et j'utilise ``fuff``. Donc j'ai cherché s'il était possible de faire la même chose et la réponse est **oui** :

```bash
/opt/ffuf/ffuf -u http://ceng-company.vm/ -H "Host: FUZZ.ceng-company.vm" -w /usr/share/wordlists/directory-list-2.3-big.txt -fs 35995
```

- **-H** permet de spécifier le *Header* dans lequel on va indiquer le champ à fuzz. Il sera donc remplacé par chaque mot de la wordlist.
- **fs** permet de filtrer la taille de la réponse HTTP. Avec le premier ``ffuf`` au départ on avait l'``index.html`` qui avait une taille de *35995*. Donc on va exclure tous les VHOSTS qui n'ont pas une taille identique.

Finalement on se retrouve avec : 

```
dev                     [Status: 200, Size: 4764, Words: 124, Lines: 99]
```

Ce qui signifie qu'on a trouvé un hôte virtuel grâce au fuzzing, qui va nous permettre d'accéder à un autre site sur le même serveur.<br>

Il nous suffit de l'ajouter au fichier ``/etc/hosts`` tout comme on l'avait fait au début. Voilà à quoi ressemble la ligne correspondante :

```
192.168.1.109   ceng-company.vm  dev.ceng-company.vm
```

On peut maintenant y accéder avec ``http://dev.ceng-company.vm/``.<br>

On se retrouve sur une page de **login**. Pour voir comment elle réagissait j'ai essayé de me connecter avec *admin:admin*, mais c'était à prévoir, ça n'a pas fonctionné.<br>

Cependant on peut voir un message d'erreur ``Incorret username or password``. On remarque qu'il est également indiqué dans l'URL.
J'ai donc tenté d'y **injecter du code javascript** pour voir s'il était vulnérable aux **XSS**: 

```
http://dev.ceng-company.vm/index.php?error=<script> alert("xss");</script>
```

Effectivement une pop-up apparaît bien à l'écran. Cependant on ne peut pas faire gran chose avec une **reflected XSS**.

* * *

## Admin panel via brute force
* * *

La première option qui n'est pas la plus fun est de tenter un brute force sur la page de login en espérant trouver la bonne combinaison.

On connait la forme des adresses mail grâce au ``hint`` trouvé sur **Github**.
On connait également le nom des personnes de l'équipe qui travaillent dans l'entreprise.<br>

On pourrait donc tenter un brute force sur les adresses mail avec une grosse wordlist style ``rockyou``. Le problème c'est le **temps** que ça va prendre.<br>

Il ne faut pas oublier qu'on a un poème trouvé sur **Reddit** qui ne nous a pas servi à grand chose pour le moment. Est-ce qu'on pourrait pas s'en servir pour générer de potentiels mot de passe ? *(Nous sommes dans un CTF après tout :p)*.

On va reprendre le poème qu'on a stocké dans un fichier nommé ``reddit_poem.txt`` et on va y **appliquer des modifications** pour en faire une wordlist utilisable :

```bash
cat reddit_poem.txt| sed 's/[ ,\t]/\n/g' | sed '/^$/d' | sort | uniq > custom_wordlist.txt
```

Cette commande va récupérer le contenu du fichier, remplacer toutes les ``,`` et tabulations par un retour à la ligne pour avoir un mot par ligne.
On va ensuite enlever toutes les lignes vides et retirer les doublons. On redirige maintenant vers un nouveau fichier ``custom_wordlist.txt``.

Maintenant qu'on a notre wordlist personnalisée on va pouvoir tenter le brute force.

Nous avons donc 4 mails *(qu'on va aussi placer dans un fichier ``users.txt``)*:

```
ericthompson@ceng-company.vm
rodneycooper@ceng-company.vm
johndoe@ceng-company.vm
elizabethsky@ceng-company.vm
``` 

```bash
hydra -L users.txt -P custom_wordlist.txt dev.ceng-company.vm http-post-form "/login.php:username=^USER^&passwd=^PASS^:F=Incorret username or password" -t 64 -V
```

Avec **hydra** on spécifie la liste des utilisateurs, la wordlist de mots de passe, la cible (dans notre cas le nom de domaine), le type de formulaire (*POST*), la page de login avec les paramètres ainsi que le message d'erreur lorsque les credentials sont incorrects. On précise aussi **-t 64** pour spécifier le nombre de threads et augmenter la rapidité du brute force.

```
[80][http-post-form] host: dev.ceng-company.vm   login: elizabethsky@ceng-company.vm   password: walnuttree
```

Le mot de passe **walnuttree** est également présent dans ``rockyou`` mais ça aurait été beaucoup trop long de tout tester.<br>

Mais avant de se connecter avec les identifiants on peut utiliser une autre technique pour les récupérer.


* * *

## Admin panel via SQL injection
* * *






* * *

## Shell basique

Pour ce faire j'ai utilisé **nc**.



On ne va pas aller trop loin et rechercher dans le dossier **admin** si on trouve quelque chose d'important.                                                                                     
Et effectivement nous sommes servis !

Ça nous sera sûrement utile par la suite. Avec un petite recherche Google sur **$apr1** on se rend compte qu'il s'agit de MD5 utilisé par Apache avec plusieurs itérations. Le lien [ici](https://httpd.apache.org/docs/2.4/fr/misc/password_encryptions.html).

Heureusement j'ai toujours **hashcat** sous la main et ça tombe bien puisqu'il possède un mode pour cracker les hashs MD5 utilisés par Apache :


Après quelques instants on trouve le mot de passe : **jessie**.

Nous avons donc la combinaison suivante : ``itsmeadmin:jessie``.                                                                                                                                       
Sur la machine il n'y a pas de compte nommé **itsmeadmin** donc on ne peut que l'utiliser sur le site web à l'adresse : http://192.168.1.47/admin

Effectivement on se retrouve connectés. Sur cette page on retrouve un bouton **Clean Downloads** qui exécute une commande via l'URL : 


Ok, donc on doit pouvoir nous aussi exécuter des commandes via cette URL. Essayons :



Le résultat n'est pas aussi flamboyant qu'on l'aurait espéré :



Effectement, on peut **exécuter** des commandes avec les droits de l'utilisateur ``www-data`` mais pour le moment peu d'utilités puisque nous avons déjà un shell avec cet utilisateur.
Il s'agit donc soit d'une fausse piste, soit il y a une autre façon d'exploiter ceci.

De retour côté shell je tente de lancer un petit **[linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)** pour voir les potentielles données intéressantes (SUID, cronjobs, passwords, configs ...). Mais rien de très probant.

J'ai appris il n'y a pas si longtemps que même s'il n'y avait rien dans les cron jobs il fallait quand même vérifier les services qui tournaient périodiquement sur la machine. Et effectivement ça m'a servi puisque qu'en utilisant **[pspy64](https://github.com/DominicBreuker/pspy)** on se rend compte que **root** lance toutes les minutes la commande suivante :

Bingo ! Il nous suffit de le modifier pour ajouter notre reverse shell qui sera exécuté par ``root`` :


Cette box n'était pas très compliquée même si elle m'a fait chercher pendant quelques temps comment **bypass les espaces** dans l'URL ainsi que le service qui tournait en fond mais qui n'était pas dans les **cron jobs**.

Merci à [@over_jt](https://twitter.com/over_jt) pour cette box !

Les prochaines fois faîtes bien **attention** ! Si vous ne voyez pas de cron jobs actifs cela ne veut pas dire qu'il n'y a rien qui tourne **périodiquement** sur la machine !