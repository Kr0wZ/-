---
layout: default
title: "PHP - Type Juggling"
date:   2020-07-09 10:00:00 +0200
published: true
categories:
  - Web
---

### Ce que nous allons voir dans cet article :

*	[Le typage en PHP](#typage-en-php)
* 	[Jongler avec les types (Type Juggling)](#jongler-avec-les-types)
*	[Loose comparison](#loose-comparison)
*   [Mise en pratique](#mise-en-pratique)
*	[Comment s'en protéger ?](#se-protéger)
*   [A vous de jouer !](#a-vous-de-jouer)
* 	[Sources](#sources)

* * *

## Typage en PHP

PHP fait partie de ses langages qui ont un **typage faible** et donc qui laisse libre court à son imagination (*pour le plus grand plaisir des hackers*).
En effet, lorsqu'on déclare une variable on ne définit pas de type et c'est pas lui même qui va choisir en fonction de ce qu'il voit (pour utiliser des termes plus précis on parle de **contexte d'utilisation**). La plupart du temps ça fonctionne comme on le souhaite mais c'est assez dangereux si on ne l'utilise pas de manière prudente. On va le voir tout au long de cet article.

De plus le **type** d'une variable **peut changer** au cours du temps. 

Un exemple simple :

```php
$var = "0";  //Ici notre variable est de type string
echo gettype($var); // => string 
```

Maintenant si on veut ajouter le chiffre 1 pour l'additionner au 0 de notre chaîne de caractères :

```php
$var = $var + 1;  //On ajoute 1 à notre chaîne de caractères
echo $var."\n";  //On affiche bien 1
echo gettype($var); // => integer 
```

Sur un langage à fort typage on aurait eut une erreur indiquant un problème d'addition entre deux types différents.
On aurait dans la pire des cas pu s'attendre à une concaténation entre le 0 et le 1 (même si le caractère de concaténation est ``.`` on aurait pu s'attendre à tout avec PHP).
Mais ici c'est bien une convertion d'une chaîne de caractères en entier qui a été effectuée de manière automatique sur la variable ``$var`` par PHP pour pouvoir additionner les deux valeurs.

C'est via cette convertion automatique qu'on va pouvoir exploiter une vulnérabilité qui s'appelle le **type juggling**.

* * *

## Jongler avec les types

Le **type juggling** ou **manipulation de types** en français est donc ce qui est utilisé par PHP pour faire ses jolies conversions automatiques. On va donc pouvoir jouer avec ça pour exploiter certaines vulnérabilités, notamment dans des conditions ou lors d'une authentification.

PHP utilise deux principaux modes de comparaisons :

* **Loose comparison (==)** : c'est ce qu'on va pouvoir exploiter.
* **Strict comparison (===)** : ici aucun problème de sécurité.

On s'attardera à la fin sur la strict comparison pour la sécurisation de nos conditions.

* * *

## Loose comparison

Ici on peut voir le tableau des comparaisons avec la loose comparison (``==``) :

![](../../../pictures/php-type-juggling/loose-comparison.png)

Ce qui nous saute tout de suite aux yeux ce sont des conparaisons étranges qui renvoient **VRAI** alors qu'elles ne devraient pas.
Notamment comparer ``0`` avec une chaîne de caractère qui va nous renvoyer **TRUE**.                                                         
Comme nous l'avons vu dans la partie précédente lorsqu'on tente de faire une quelconque opération entre un **string** et un **entier**, PHP va tenter de convertir la chaîne pour pouvoir faire l'opération. C'est ici qu'on va pouvoir s'amuser.

On vient de voir que PHP fait des conversions quand on compare un string avec un entier. Mais ce que je ne vous ai pas dit c'est que si dans une condition les deux chaînes de caractères **ressemblent à un nombre** alors PHP va également les convertir pour pouvoir les comparer. Ce qui va beaucoup nous servir.

Par exemple si on veut comparer cette chaîne : ``"0e55555"`` avec 0 alors PHP va transformer sa forme exponentielle sous forme de chaîne de caractère en entier. Et on sait qu'un très petit nombre (proche de 0) converti en entier est égal à 0.

Donc :

```php
"0e55555" == 0  // -> TRUE
```
On peut également se référer au tableau des loose comparisons pour tenter d'autres conditions qui renvoient TRUE.

Vous commencez à comprendre ce qu'on va faire ? :p


* * *

## Mise en pratique

Imaginons que nous devons entrer un mot de passe pour se connecter en tant qu'admin d'un site web et que nous avons accès à ce code source :

```php
<?php

if(isset($_GET['username']) && isset($_GET['password'])){
	if($_GET['username'] == TRUE && $_GET['username'] == md5($_GET['password'])){
		echo "Welcome MASTER !";
	}
	else {
		echo "Sorry wrong credentials";
	}
}
else {
	echo "You must specify both username and password in URL ...";
}

?>
```

Ce code est plutôt simple (vous pouvez le retrouver sur mon [Github](https://github.com/Kr0wZ/php-loose-comparison/blob/master/article-example.php)).                                                                                             
La première condition va tester si les paramètres ``username`` et ``password`` sont biens passés via la méthode **GET** (par l'URL).                                                      
Ensuite on va regarder si le paramètre ``username`` est égal à ``TRUE`` **ET** que le ``username`` est égal au **hash MD5** du mot de passe (lui aussi passé en paramètres).

A première vu ça semble compliqué de passer ça. Si vous avez lu cet article jusqu'ici et que vous ne savez pas comment passer cette condition pour devenir admin ... c'est normal !                  
Je ne vous ai pas encore parlé de quelque chose qui va justement nous permettre de **bypass** cette condition.                                                 

Et cette fameuse chose n'est autre que le **magic hash**. En soit il n'a rien de bien magique si ce n'est qu'une certaine chaîne de caractère, après être passée sous MD5 va donner un hash plutôt magique.

Passons au concret : 

Disons qu'on va prendre ~~au hasard~~ cette chaîne de caractères : ``0e215962017``. Que se passe-t-il si on le hash avec l'algorithme MD5 ?                                                    
On obtient le hash suivant : ``0e291242476940776845150308577824``

Tiens, tiens, tiens ...

On reprend la phrase écrite plus haut : ``Si dans une condition les deux chaînes de caractères ressemblent à un nombre alors PHP va également les convertir pour pouvoir les comparer.``                

Cette phrase ajoutée à celle-ci devrait vous mettre la puce à l'oreille : ``Par exemple si on veut comparer cette chaîne : "0e55555" avec 0 alors PHP va transformer sa forme exponentielle sous forme de chaîne de caractère en entier. Et on sait qu'un très petit nombre (proche de 0) converti en entier est égal à 0.``.

Toujours pas ? 

```php
echo "0e215962017" == "0e291242476940776845150308577824" ? "TRUE" : "FALSE";
``` 
Le résultat de cette condition est ``TRUE``.

PHP a converti chacune de ces chaînes en nombre entier donc ``"0e215962017" = 0`` et ``"0e291242476940776845150308577824" = 0``. Puis il a fait la comparaison : 

```php
0 == 0 // -> TRUE
```

Maintenant qu'on sait ça reprenons notre code.
La première condition à remplir est simple puisque nous devons juste insérer un ``username`` et un ``password``.
La première partie de la deuxième condition :
```php
$_GET['username'] == TRUE
```

Est également simple puisqu'en regardant notre tableau des loose comparisons on voit qu'il suffit de passer une quelconque chaîne de caractères à ``username`` pour que cette condition soit **TRUE**.

Avec ce que nous venons de voir juste avant avec les **magic hashes** tout devient plus clair : 

```php
$_GET['username'] == md5($_GET['password'])
```

Il suffit de passer en tant que mot de passe un magic hash pour que le résultat de la fonction MD5 soit quelque chose de la forme ``0e...`` avec que des nombres. Le magic hash vu précédemment fonctionne.

On peut tenter de devenir admin en appelant l'URL suivante :

```
http://127.0.0.1/type_juggling.php?username=0e11111111&password=0e215962017
```

``0e11111111`` est donc bien une chaîne de caractère et elle est égale à la valeur du hash md5 de ``0e215962017``.                                                           
Puisque je le répète ``0e11111111`` sera converti en entier valant 0 et sera comparé à ``0e291242476940776845150308577824`` (hash MD5 de ``0e215962017``) qui lui même vaudra 0.

Nous sommes donc devenu admin en utilisant la loose comparison pour bypass les conditions.

Vous pouvez retrouver une grosse liste de magic hash sur ce [Github](https://github.com/spaze/hashes).

* * *

## Se protéger

Il existe un moyen de contrer les caprices de PHP avec ses conversions automatiques. Pour cela on va utiliser la stric comparison.                                         
Il s'agit du triple ``=`` : ``===``.

Qu'est ce qu'il fait de plus que la loose comparison ? Et bien en plus de tester les valeurs passées il va regarder si leurs **types** sont également **identiques**.
Ici on évite donc à PHP de faire des conversions douteuses tout seul de son côté.
```
(a === b) revient à (a == b && gettype(a) == gettype(b))
```

Voici le tableau correspondant :

![](../../../pictures/php-type-juggling/strict-comparison.png)

De ce fait le loose comparison de la partie précédente que renvoyait **TRUE** avec ``"0e11111111" == 0`` renvoie maintenant ``FALSE`` car leurs types sont différents (string et int).


## A vous de jouer

Maintenant que vous en savez plus sur les **loose comparisons** vous voulez peut-être tenter par vous même un petit **challenge** que j'ai réalisé à ce sujet ? 

Si oui alors je vous invite à vous rendre sur mon **[Github](https://github.com/Kr0wZ/php-loose-comparison)** pour y accéder et tenter de devenir le nouvel administrateur de ce site ;)

Si vous avez des **questions** et/ou que vous voulez me faire un retour sur cet article vous pouvez rejoindre le **[Discord](https://discord.gg/v6rhJay)**.

* * *

### Pour lancer un serveur PHP local

Si vous êtes sur **Linux** vous devez aller dans le répertoire dans lequel se trouve votre fichier PHP puis lancer un serveur local avec la commande ``php`` :

```bash
php -S localhost:8080
```

Il faut ensuite se rendre sur votre navigateur : ``http://localhost:8080/votre_fichier.php``. En remplaçant bien évidemment ``votre_fichier`` par le nom du fichier PHP.

Sur **Windows** vous pouvez installer [EasyPHP](https://www.easyphp.org/) ou bien un serveur [WAMP](https://www.wampserver.com/).


## Sources

[https://medium.com/@Q2hpY2tlblB3bnk/php-type-juggling-c34a10630b10](https://medium.com/@Q2hpY2tlblB3bnk/php-type-juggling-c34a10630b10)

[https://owasp.org/www-pdf-archive/PHPMagicTricks-TypeJuggling.pdf](https://owasp.org/www-pdf-archive/PHPMagicTricks-TypeJuggling.pdf)

[https://www.php.net/manual/fr/language.types.type-juggling.php](https://www.php.net/manual/fr/language.types.type-juggling.php)

[http://dsiohan.free.fr/php/francais/language.types.type-juggling.html](http://dsiohan.free.fr/php/francais/language.types.type-juggling.html)

[https://tls-sec.github.io/documents/RHE/PHPTypeJuggling.pdf](https://tls-sec.github.io/documents/RHE/PHPTypeJuggling.pdf)

[https://github.com/spaze/hashes](https://github.com/spaze/hashes)