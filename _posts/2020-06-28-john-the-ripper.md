---
layout: default
title: "And his name is John ... The Ripper"
date:   2020-06-28 10:00:00 +0200
published: true
---

### Ce que nous allons voir dans cet article :

*	[John The Ripper c'est quoi ?](#johntheripper)
* 	[Qu'est-ce qu'un hash ?](#le-hachage)
*	[Principe d'un "cracker" de hash](#cracker-un-hash)
*   [Mise en pratique](#mise-en-pratique)
*   [Exemples](#exemples)
*   [A vous de jouer !](#a-vous-de-jouer)

* * *

## John The Ripper

### John The Ripper c'est quoi ?

John The Ripper est un outil pour **casser des mots de passe** (password cracker). 
C'est un programme **libre** d'utilisation que l'on peut retrouver ou télécharger sur le site [Openwall](https://www.openwall.com/john/).

La force de ce logiciel est qu'il permet de détecter (dans la plupart des cas) le type de hash utilisé pour hasher le mot de passe.
Il permet également de choisir entre plusieurs **modes** de cracking :

* **Mode single** : Équivalent au mode wordlist mais utilisant des règles spécifiques qui rendent ce mode rapide
* **Mode wordlist** : Utilise une wordlist pour comparer le(s) hash à cracker avec les mots de passe contenus dans celle-ci
* **Mode incrémental** : C'est le mode bruteforce qu'on connaît, il va essayer caractères par caractères jusqu'à trouver le bon mot de passe (très lent)

Pour avoir plus d'informations sur ces modes je vous invite à vous rendre sur la [documentation officielle](https://www.openwall.com/john/doc/MODES.shtml).

Dans la suite de cet article, pour parler de **John The Ripper** on parlera juste de **John**.

![](../../../pictures/john-the-ripper/John-the-Ripper-logo.png)

* * *

## Le hachage

### Qu'est-ce qu'un hash ?

Un hash est le résultat d'une **fonction de hachage**. C'est une **chaîne de caractères** plus ou moins longue (en fonction du type de hash) composée de caractères imprimables.

Il existe différents types de hash qui n'ont pas tous le même format. De même, certains sont plus **sécurisés** que d'autres.
On peut prendre l'exemple du **MD5** qui n'est plus du tout recommandé à cause de son manque de sécurité.                                             
Il est donc préférable d'utiliser du **SHA256**.

Une fonction de hachage est une transformation mathématique d'un mot ou d'une chaîne de caractères en hash. Chaque fonction de hachage est différente en fonction du résultat recherché.

Le schéma ci-dessous montre bien le fonctionnement.

Comme on peut le voir il n'existe **AUCUNE** de fonction mathématique permettant d'effectuer l'opération inverse et de retrouver le texte clair à partir d'un hash. 

Le hachage est donc très souvent utilisé sur les mots de passe pour les rendre illisibles.                                                 
Il est également utilisé en tant que signature pour savoir si un fichier ou un programme est authentique, on parle alors d'**empreinte numérique**.

Il est possible d'ajouter ce qu'on appelle un **salt**, qui est une chaîne de caractères que l'on va concaténer à celle que l'on veut hacher. Celà va permettre de renforcer la sécurité.

Une bonne fonction de hachage ne possède que très peu de **collisions**. Une collision intervient lorsque deux entrées différentes possèdent le même hash en sortie de la fonction de hachage.         
Ici un exemple avec MD5 :

Nous avons deux chaînes de caractères **différentes** représentées sous forme hexadécimale.

``` 
4dc968ff0ee35c209572d4777b721587d36fa7b21bdc56b74a3dc0783e7b9518afbfa200a8284bf36e8e4b55b35f427593d849676da0d1555d8360fb5f07fea2  
4dc968ff0ee35c209572d4777b721587d36fa7b21bdc56b74a3dc0783e7b9518afbfa202a8284bf36e8e4b55b35f427593d849676da0d1d55d8360fb5f07fea2  
                                                                       ^                                      ^
                                                                       |                                      |
```

On va donc les convertir en fichier binaire :

```bash
echo '4dc968ff0ee35c209572d4777b721587d36fa7b21bdc56b74a3dc0783e7b9518afbfa200a8284bf36e8e4b55b35f427593d849676da0d1555d8360fb5f07fea2' | xxd -r -p | tee >/dev/null > file1  
echo '4dc968ff0ee35c209572d4777b721587d36fa7b21bdc56b74a3dc0783e7b9518afbfa202a8284bf36e8e4b55b35f427593d849676da0d1d55d8360fb5f07fea2' | xxd -r -p | tee >/dev/null > file2  
```

Lorsqu'on calcule leur hash MD5 :

```bash
md5sum file1 file2

008ee33a9d58b51cfeb425b0959121c9  file1
008ee33a9d58b51cfeb425b0959121c9  file2
```

On remarque que les hash sont **identiques**. Il y a donc une **collision**.

Exemple tiré d'[ici](https://marc-stevens.nl/research/md5-1block-collision/).

On peut directement voir que ça pose de gros problèmes de **sécurité**. En effet, il serait possible de trouver un mot de passe sans avoir besoin de le connaître si jamais on arrivait à en générer un autre avec la **même empreinte**. 

De plus MD5, est connu pour ses **faiblesses cryptographiques**.                       
C'est pour ça que MD5 est **déconseillé**, car il ne dispose plus d'une sécurité suffisante de nos jours.

Il existe des alternatives comme **SHA256** avec un salt ou **SHA512** (toujours avec salt) qui sont considérées comme sécurisées.

* * *

## Cracker un hash

### Le principe

Maintenant qu'on en connaît un peu plus sur le hachage on est en droit de se demander comment il est possible de comparer un hash avec un dictionnaire de mots de passe en clairs.
Ce principe n'est pas propre qu'à John et vous pouvez vous aussi faire un petit programme de crack de mots de passe.

A chaque mot de passe testé (que ce soit dans une wordlist ou en bruteforce) on va appliquer la même fonction de hachage que celle utilisée sur celui qu'on doit cracker.
On va ensuite comparer les deux hash et regarder s'ils sont identiques. Si c'est le cas alors c'est soit qu'on a trouvé le mot de passe, soit qu'il y a eu une collision. Dans les deux cas notre objectif est accompli.

Par exemple si le hash à cracker est du MD5 alors pour chaque mot de passe testé on va le hasher en MD5 et le comparer.

Dans le cas des **Rainbow Tables** c'est un peu différent mais comme John ne possède pas d'options concernant les Rainbow Tables on va les laisser de côté.

## Mise en pratique

John est plutôt simple d'utilisation.                                                         

Pour cracker un hash dont nous **connaissons** la chaîne de caractère (par exemple 63a9f0ea7bb98050796b649e85481845), il suffit de stocker cette chaîne dans un fichier texte que l'on nommera *hash.txt*.

On peut ensuite utiliser John pour tenter de le cracker.                                   
Son usage est le suivant :

```
john [OPTIONS] [PASSWORD-FILES]
```

De notre côté on va s'attarder sur les options qu'on va le plus souvent utiliser c'est-à-dire :                                                        

* **\-\-wordlist=WORDLIST** : spécifie une liste de mots de passe potentiels à tester 
* **\-\-format=FORMAT** : spécifie le format du hash (par défaut John essaiera de le trouver par lui même)
* **\-\-list=SECTION** : affiche la liste des hashs ainsi que d'autres informations supportées par John

Pour plus d'informations sur les options possibles on peut consulter le ``man`` de John ou bien en utilisant cette commande ``john --help`` ou ``john -h``.

De ce fait, notre commande devrait ressembler à ceci :

```bash
john --wordlist=/usr/share/wordlist/rockyou.txt hash.txt
```

Ici nous avons utilisé la wordlist **rockyou.txt** qui est plutôt efficace (*notemment pour les CTFs*) et qui regroupe plus de **14 millions** de mot de passes.
Vous pouvez la télécharger [ici](https://github.com/praetorian-code/Hob0Rules/blob/master/wordlists/rockyou.txt.gz).
Il existe aussi d'autres wordlists plus spécifiques et plus courte qui sont disponibles sur ce [Github](https://github.com/danielmiessler/SecLists).



Liste des formats supportés par John : http://pentestmonkey.net/cheat-sheet/john-the-ripper-hash-formats.

Nous venons de voir comment utiliser John en ayant la chaîne de caractères correspondant au hash. Cependant il est possible qu'on veuille cracker un fichier zip protégé par un mot de passe ou encore une clé privée SSH sécurisée par une passphrase.

Dans ce cas il nous faut utiliser d'autres outils fournis avec John.                    
En effet, ils sont nécessaires pour *extraire* et *transformer* le hash contenu dans ces éléments pour le mettre dans un bon format exploitable par John.

Ils sont sous la forme **\<truc\>2john**.

C'est ce que nous allons vois dans les exemples suivants.

* * *

## Exemples
### Zip 

On crée une archive zip avec un mot de passe simple contenant un fichier pour les besoins de cet exemple :

```bash
echo "flag" > file.txt
zip -e encrypted.zip file.txt #avec comme password -> password123
```

Si on tente de ``unzip`` notre archive alors on nous demande le mot de passe. Sauf qu'on est censé ne pas encore le connaître donc il est impossible d'accéder à l'archive (*pour le moment*)

On va utiliser l'utilitaire **zip2john** pour récupérer le hash et le donner à John par la suite :

```bash
ssh2john encrypted.zip > hash_zip.txt
```

On peut regarder à quoi ressemble le contenu de ``hash_zip.txt`` :

```
encrypted.zip/file.txt:$pkzip2$1*2*2*0*11*5*22dc8822*0*42*0*11*22dc*8257*4b96b8bba4f282c40223dd7dd855088021*$/pkzip2$:file.txt:encrypted.zip::encrypted.zip
```

On ne va pas s'attarder sur la forme de cette ligne. Mais maintenant John va pouvoir travailler dessus.

On utilise donc John sur ce nouveau fichier qui contient le hash :

```bash
john --wordlist=/usr/share/wordlist/rockyou.txt hash_zip.txt
```

```bash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
password123      (encrypted.zip/file.txt)
1g 0:00:00:00 DONE (2020-07-07 16:35) 25.00g/s 102400p/s 102400c/s 102400C/s 123456..oooooo
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

On a réussi à récupérer le mot de passe de l'archive. On peut maintenant ``unzip`` notre .zip avec le mot de passe trouvé pour accéder à son contenu.

* * *

### SSH

Comme pour l'exemple précédent on va devoir mettre en place certaines choses pour pouvoir utiliser John.

On commence donc par générer une paire de clé publique/privée SSH avec une passphrase (pour la clé privée) :

```bash
ssh-keygen -t rsa
```

Notre passphrase à cracker sera **100%princess**.

On va utiliser l'utilitaire **ssh2john** pour récupérer le hash de la clé privée et le mettre sous la bonne forme pour que John puisse l'utiliser :

```bash
ssh2john.py id_rsa > hash_ssh.txt
```

Le hash est donc stocké dans le fichier **hash_ssh.txt** :

```
id_rsa:$sshng$1$16$0FEB8C9AB01437B85D283C08E982E043$1200$ffb9d97dbb46e6035400422a37a16fed6d5933943e7fc703beb30d59e8df79ee2f0bb94fed5eb39bc0280af461e760e2dcc0ed037638603f80d48ea4fa77cfc41217034c98d8e1e3fd05538905908595021441985e39b18f864444fd25cca18e4b831685fbabfbfc89a344be1f885162c6a67f4e63af8706de392fcb60f2228926d2839fd8b781ae881d795ba0e22ae940dcea4a995c7984669c82bef6637454baa8b52604949c3046da7dfd044c9e35b385438a5b46252072298749398cc5b62b4a1708acb3fbae760b38b475689616e59a7b5e03bee84129dc7007b371b7c6b7f7795f5c82904f1ad6ecd2b07310a296ed61c39fc0b577b9f8c07af1e80fcb3a4ddd44dbf2037446b2e5a93a367c79e677be527aa445d8662d4e7e1d02962526ea4988c2bac9977b92b729a6f3393d431bdddaa796b08a6098ffea18ec732aee57bc3471d8299a70df43065186b9294cd940403c32270eabe18a6a0ad84e217114af22289ca9e6ff3f77950cf6f0b1e09f556102b6a4b156a31f7b2d7df4e100407fc127559dc21227e0e4da58a89d58c9a18b44a3c326d63143eaaa441711aa6a6881429d8882f665630701e4d166cd7e1f771822666ce0870ac2bec260a729178fdbe9009b695c999b36d8bac259c6aff65d19a4975505412b6e5d6db203c1184a67cf2dd243b8a20ac2bea9453da7a5741ab5fb6bba4d89c1525ce19dd2c0aeb94be2cbdeb9d2f7d495cc9a874044e13029d17538de437b3b1048e805e0e476f3f557111e7a1ee1f4ad11a5bc3008817b0ce7e6de47e3211005766e08b39c8cd952fb8ffe1688814f211b3ae6267424c7dec669e267e4cff6e903280161672109112ece14a34ad47551473be553f33bee0991a2ba543102ef51b0f00ef9602355b4407b6a5ddca72fe178b1f28edbc5c409deaf6dfb3640f7972d6d1d3c92a9effc0e6e07ab496128769156f32f9f259158a5885e12e78c7fe450892d92bbade2826c482754f075d9d88f2f34dd0c99ff643187778c4994bc9910e050ac7c53dca9244d1fa948992cdd5e63716d8893e8948b8253dc306d85cfbb33bdb5e1d55f41307630d124061b6576c471aaa200996e1bbaac0174675e534bd3fcc55764a2eaffe2ee056e8d9ab6791130d570351cfd6370e478d6797a989d5054d32ad0dcf390257df23866a116ec51d72c50b284449d662499a133c457c8b2079570c2238d13eee2dbc9eea387a3fedb1c044e7dd785745c4974071aeb6dc93e7ededa1d4335fbd3367516c308680f6a41d1f4d6e8c59bb357f87adab7f06efca0998336df06b72d7d9faa6f42fb15d122d0d743e1278a5a90a124993490eddfa147a22af1cd76300427f0b3fd8c4fbbc423ab4a6f3668bfcde46836394db0aa52c6f4efc0ff84291e8fec12a26bf64c0e0f3e73cf7cf434bbf43f00123720a3657c72bdba6a805d31b0b978153af3d36996f4475e1d5dac69a4084c36d2b38aa7d3c028cdbe4aa2f64887731dad9b655cede54b8772af77caacc05d911380c1821552a5c7c96b35f13bc8c6facb894b3bc0b0c17e0ce181443ab2d92236ed342ed7edc8b9a2e1b16e78c28439b3d128fca774c38d6d6bdfc32857f2c45f3d848377f9c9c6970eb9d3d9bb6c5f47e44e7a083d325b4cb1c1573c9040ddf3665918a65ec466804ea7b411e742e05850d59c
```

Nous n'avons plus qu'à utiliser John sur ce fichier : 

```bash
john --wordlist=/usr/share/wordlist/rockyou.txt hash_ssh.txt
```

Et il nous ressort :

```bash
Note: This format may emit false positives, so it will keep trying even after finding a
possible candidate.
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
100%princess     (id_rsa)
1g 0:00:00:06 DONE (2020-07-07 16:54) 0.1610g/s 2309Kp/s 2309Kc/s 2309KC/s *7¡Vamos!
Session completed.
```

On peut donc maintenant utiliser la clé privée pour accéder au serveur SSH sur lequel on est censé se connecter.

* * *

## A vous de jouer

Dans cet article on a vu comment utiliser John en ayant des **hashs**, des **archives zip** ainsi que des **clés privées SSH**.

Cependant il existe beaucoup plus de possibilités que ce qu'on a vu. On peut retrouver tous ces utilitaires en regardant les fichiers sources disponibles sur le [Github officiel de JohnTheRipper](https://github.com/magnumripper/JohnTheRipper/tree/bleeding-jumbo/run).

Vous trouverez sur mon [Github](https://github.com/Kr0wZ/JohnTheRipper-examples) une archive zip, une clé privée SSH ainsi qu'un autre fichier sur lesquels vous pouvez appliquer les exemples précédents.                                                                   
Pour le dernier fichier l'exemple n'a pas été vu dans cet article mais si vous avez compris le fonctionnement ça ne devrait pas poser problème.


Si vous avez des questions n'hésitez pas à vous rendre sur le serveur [Discord](https://discord.gg/v6rhJay) !

* * *

## Sources 

[https://fr.wikipedia.org/wiki/John_the_Ripper](https://fr.wikipedia.org/wiki/John_the_Ripper)

[https://www.openwall.com/john/](https://www.openwall.com/john/)

[https://github.com/magnumripper/JohnTheRipper](https://github.com/magnumripper/JohnTheRipper)

[http://pentestmonkey.net/cheat-sheet/john-the-ripper-hash-formats](http://pentestmonkey.net/cheat-sheet/john-the-ripper-hash-formats)

[https://marc-stevens.nl/research/md5-1block-collision/](https://marc-stevens.nl/research/md5-1block-collision/)