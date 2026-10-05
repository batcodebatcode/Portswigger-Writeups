# SQL injection UNION attack : extraire des données d'autres tables

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (UNION attacks)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

## Contexte

Même application que les labs UNION précédents : les produits sont filtrés par catégorie via le paramètre `category` sur `/filter`, qui est injectable.

Ce lab combine toutes les briques vues avant (compter les colonnes, trouver les colonnes texte) pour passer à l'objectif réel d'une attaque UNION : extraire des données d'une autre table que celle interrogée au départ.

La base contient une table `users` avec, entre autres, deux colonnes : `username` et `password`. Important sur le vocabulaire : `users` est la **table**, `username` et `password` sont des **colonnes** de cette table.

## Objectif

Récupérer les identifiants stockés dans la table `users`, en particulier le mot de passe de l'utilisateur `administrator`, puis se connecter en tant qu'administrateur.

## Démarche

### Étape 1 : préparer l'attaque UNION

Comme pour les labs précédents, la requête d'origine renvoie ici 2 colonnes, toutes deux compatibles avec du texte. Mon `UNION SELECT` doit donc renvoyer 2 valeurs texte.

### Étape 2 : extraire les identifiants

J'injecte une requête UNION qui va chercher les colonnes `username` et `password` dans la table `users` :

```
' UNION SELECT username, password FROM users--
```

Décomposition :
- `FROM users` : va chercher dans la table `users`.
- `SELECT username, password` : récupère ces deux colonnes.
- Les résultats s'ajoutent à l'affichage des produits.

La page liste alors tous les comptes de l'application avec leur mot de passe.

### Étape 3 : exploiter le résultat

Dans la liste, je repère la ligne de l'utilisateur `administrator` et son mot de passe. Je vais ensuite sur le formulaire de connexion, je me connecte avec `administrator` et le mot de passe récupéré. La page affiche "Your username is: administrator", ce qui résout le lab.

## Payload utilisée

```
' UNION SELECT username, password FROM users--
```

## Résultat

La requête renvoie la liste des utilisateurs et de leurs mots de passe. Je récupère ceux de l'administrateur, je me connecte avec, et le lab passe en "Solved".

## Ce que j'en retiens

- C'est l'aboutissement de l'attaque UNION : à partir d'un seul paramètre vulnérable, on sort des données d'une table que l'application n'avait jamais prévu d'exposer.
- Vocabulaire à ne pas confondre : une **table** (`users`) contient des **colonnes** (`username`, `password`), qui contiennent des valeurs (les lignes).
- Le `UNION SELECT` doit renvoyer le bon nombre de colonnes, du bon type, sinon erreur. D'où l'intérêt des étapes de reconnaissance faites dans les labs précédents.
- Ce scénario illustre concrètement une fuite de base de données : une seule faille d'injection peut exposer tous les comptes.

## Note technique

Ici les deux colonnes affichables suffisaient pour sortir `username` et `password` séparément. Si une seule colonne texte avait été disponible, il aurait fallu concaténer les deux valeurs dans une même colonne, par exemple :

```
' UNION SELECT username || '~' || password, NULL FROM users--
```

La syntaxe de concaténation dépend du SGBD (`||` sur PostgreSQL/Oracle, `CONCAT()` sur MySQL).

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

En complément, les mots de passe ne doivent jamais être stockés en clair. Hachés avec un algorithme adapté (bcrypt, argon2), même une extraction complète de la table `users` ne livrerait pas les mots de passe en clair.
