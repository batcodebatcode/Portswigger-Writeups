# SQL injection UNION attack : récupérer plusieurs valeurs dans une seule colonne

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (UNION attacks)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column

## Contexte

Même application que les labs UNION précédents : le paramètre `category` sur `/filter` est injectable, et les résultats de la requête sont affichés dans la page.

La particularité de ce lab : la requête renvoie bien plusieurs colonnes, mais **une seule accepte du texte**. On ne peut donc pas placer `username` dans une colonne et `password` dans une autre, comme dans le lab précédent. Il faut combiner les deux valeurs dans la seule colonne affichable.

La base contient une table `users` avec les colonnes `username` et `password`.

## Objectif

Récupérer tous les couples identifiant / mot de passe, repérer celui de `administrator`, et se connecter en tant qu'administrateur.

## Démarche

### Étape 1 : nombre de colonnes et colonne texte

Comme toujours, je confirme d'abord le nombre de colonnes avec `ORDER BY`, puis je cherche la colonne compatible avec du texte. Ici, il y a 2 colonnes et une seule accepte du texte.

### Étape 2 : concaténer les deux valeurs

La leçon montre, pour une requête à une seule colonne, la concaténation avec l'opérateur `||` :

```
' UNION SELECT username || '~' || password FROM users--
```

Mais appliquée telle quelle, cette payload échoue ici avec "Internal Server Error", parce que la requête de ce lab attend **2 colonnes**, pas une. Fournir une seule valeur casse le nombre de colonnes.

Le point clé que ce lab demande de comprendre : il faut combiner deux acquis des labs précédents.
- La requête a plusieurs colonnes (comme au lab "déterminer le nombre de colonnes").
- On remplit la colonne non-affichable avec `NULL`.

Donc je fournis bien 2 valeurs : un `NULL` pour la colonne non-affichable, et la concaténation `username || '~' || password` pour la colonne texte :

```
' UNION SELECT NULL, username || '~' || password FROM users--
```

Si la colonne texte avait été la première, il aurait fallu inverser : `username || '~' || password, NULL`.

## Erreurs rencontrées et corrigées

- Au début j'ai écrit l'opérateur avec des espaces (`| |`), ce qui ne veut rien dire en SQL. Le bon opérateur est `||`, deux barres collées.
- J'ai aussi cru à un filtre et testé la casse (`SeLeCT`), sans effet : les mots-clés SQL sont insensibles à la casse, ce n'était pas un filtre mais une erreur de structure (nombre de colonnes).
- Le vrai manque était l'ajout du `NULL` pour faire correspondre les 2 colonnes, étape qui n'est pas dans l'exemple de la leçon (qui suppose une seule colonne) et qui demande de relier deux techniques vues avant.

## Payload utilisée

```
' UNION SELECT NULL, username || '~' || password FROM users--
```

Le résultat affiche des lignes du type :

```
administrator~<motdepasse>
wiener~peter
carlos~montoya
```

## Résultat

Je récupère le mot de passe de `administrator` dans la liste, je me connecte avec sur le formulaire de login, et le lab passe en "Solved".

## Ce que j'en retiens

- Quand une seule colonne est affichable, on concatène plusieurs valeurs dedans avec un séparateur (`~`) pour les distinguer.
- L'opérateur de concaténation dépend du SGBD : `||` sur Oracle et PostgreSQL, `CONCAT()` sur MySQL.
- Même en concaténant, le `UNION SELECT` doit respecter le nombre total de colonnes de la requête : les colonnes inutilisées se remplissent avec `NULL`.
- Un lab peut exiger de combiner plusieurs techniques vues séparément. L'exemple de la leçon est un point de départ à adapter, pas une solution clé en main.

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

Et les mots de passe doivent être hachés (bcrypt, argon2) et non stockés en clair, pour qu'une extraction de la table `users` ne livre pas directement les mots de passe utilisables.
