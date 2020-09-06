---
layout: default
title: "Writeup Bandit - OverTheWire"
date:   2020-08-08 03:00:00 +0200
published: false
---

Pour ces solutions le mot de passe n'est pas donné, cependant la marche à suivre pour l'obtenir est expliquée.

* * *

## Bandit 0

Cette première étape sert juste à nous apprendre comment se servir de **SSH**.
Il nous suffit de se connecter avec SSH en utilisant la commande suivante :

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

On précise ensuite comme mot de passe **bandit0** et nous voilà connecté.
Il suffit d'afficher le contenu du fichier **readme**.

```bash
cat readme
```

* * *
 
## Bandit 1

```bash
ssh bandit1@bandit.labs.overthewire.org -p 2220
```

Ici on ne peut pas directement afficher le contenu du fichier avec la commande :

```bash
cat -
```

Puisqu'avec cat lorsqu'on spécifie "**-**" cela veut dire qu'on veut laisser l'entrée standard ouverte.                                                                                     
Du coup il y a plusieurs façons de faire.                                                                                                                                                          
Soit en utilisant le **chemin absolu** :

```bash
cat /home/bandit1/-
```

Soit avec le **chemin relatif** :

```bash
cat ./-
```

* * *

## Bandit 2

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```

Le fichier que l'on doit afficher possède des espaces dans son nom.

Pour y remédier on peut soit échapper les caractères à l'aide du backslash (**\\**) soit en ajoutant des guillemets autour du nom pour qu'il soit considéré comme une chaîne de caractères.

```bash
cat spaces\ in\ this\ filename
cat "spaces in this filename"
```

* * *

## Bandit 3

```bash
ssh bandit3@bandit.labs.overthewire.org -p 2220
```

On nous demande de chercher un **fichier caché**.
Sous Linux la commande ``ls`` n'affiche que les fichiers lisibles. Pour afficher également les fichiers cachés il faut utiliser l'option ``-a``

```bash
ls -la ./inhere/
```

* * *

## Bandit 4

```bash
ssh bandit4@bandit.labs.overthewire.org -p 2220
```

Ici nous avons plusieurs fichiers disponibles sans connaitre celui qui contient le mot de passe.
Il y a également plusieurs possibilités.

En utilisant la commande ``find`` :

```bash
find inhere/ -type f | xargs cat
find inhere/ -type f -exec cat {} \;
```

En utilisant simplement ``cat`` :

```bash
cat ./-file0*
```

* * *

## Bandit 5

```bash
ssh bandit5@bandit.labs.overthewire.org -p 2220
```

On doit trouver un fichier avec certaines spécificités :

```bash
find inhere/ -type f -size 1033c ! -executable | xargs cat
find inhere/ -type f -size 1033c ! -executable -exec cat {} \;
```

* * *

## Bandit 6

```bash
ssh bandit6@bandit.labs.overthewire.org -p 2220
```

Même idée que pour **bandit5** sauf qu'ici les propriétés sont différentes :

```bash
find / -size 33c -group bandit6 -user bandit7 2>/dev/null | xargs cat
```

* * *

## Bandit 7

```bash
ssh bandit7@bandit.labs.overthewire.org -p 2220
```

On doit trouver le mot de passe dans le fichier **data.txt**. On sait qu'il se situe à côté du mot **millionth** :

```bash
cat data.txt | grep millionth
```

* * *

## Bandit 8

```bash
ssh bandit8@bandit.labs.overthewire.org -p 2220
```

Un peu plus compliqué cette fois-ci puisque le mot de passe n'apparaît **qu'une seule fois** dans le fichier. Et c'est le seul qui n'apparaît qu'une fois :

```bash
cat data.txt | sort | uniq -c | grep -w 1
```

Il nous suffit de trier chaque ligne, de les compter et d'afficher celle qui n'apparaît qu'une fois.

* * *

## Bandit 9

```bash
ssh bandit9@bandit.labs.overthewire.org -p 2220
```

Dans cette étape nous devons trouver le mot de passe dans un fichier qui contient certains caractères illisibles. Il nous est aussi indiqué


## Bandit 10

```bash
ssh bandit10@bandit.labs.overthewire.org -p 2220
```


## Bandit 11

```bash
ssh bandit11@bandit.labs.overthewire.org -p 2220
```


## Bandit 2

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```


## Bandit 2

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```


## Bandit 2

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```


## Bandit 2

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```

## Bandit 2

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```