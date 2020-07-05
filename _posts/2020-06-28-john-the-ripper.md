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

(logo de john the ripper)

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

Il existe des alternatives comme **SHA256** avec un salt ou **SHA512** (toujours avec salt) qui sont considérées comme *pas trop mal* sécurisées.

* * *

## Cracker un hash

### Le principe

Maintenant qu'on en connaît un peu plus sur le hachage on est en droit de se demander comment il est possible de comparer un hash avec un dictionnaire de mots de passe en clairs.
Ce principe n'est pas propre qu'à John et vous pouvez vous aussi faire un petit programme de crack de mots de passe.

A chaque mot de passe testé (que ce soit dans une wordlist ou en bruteforce) on va appliquer la même fonction de hachage que celle utilisée sur celui qu'on doit cracker.
On va ensuite comparer les deux hash et regarder s'ils sont identiques. Si c'est le cas alors c'est soit qu'on a trouvé le mot de passe, soit qu'il y a eu une collision. Dans les deux cas notre objectif est accompli.

Par exemple si le hash à cracker est du MD5 alors pour chaque mot de passe testé on va le hasher en MD5 et le comparer.

Dans le cas des **Rainbow Tables** c'est un peu différent.

(parler des rainbow tables)


## Mise en pratique

John est plutôt simple d'utilisation.


## Sources 

[https://fr.wikipedia.org/wiki/John_the_Ripper](https://fr.wikipedia.org/wiki/John_the_Ripper)

[https://www.openwall.com/john/](https://www.openwall.com/john/)

[http://doc.ubuntu-fr.org/chroot](http://doc.ubuntu-fr.org/chroot)

[https://fr.wikipedia.org/wiki/Chroot](https://fr.wikipedia.org/wiki/Chroot)

[http://www.linux-france.org/article/man-fr/man1/ldd-1.html](http://www.linux-france.org/article/man-fr/man1/ldd-1.html)

[https://stackoverflow.com/questions/34428037/how-to-interpret-the-output-of-the-ldd-program](https://stackoverflow.com/questions/34428037/how-to-interpret-the-output-of-the-ldd-program)