# SQL injection : lister le contenu de la base (SGBD non-Oracle)

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (examiner la base de données)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle

## Contexte

Même point d'injection que les labs précédents (`category` sur `/filter`). Cette fois, l'objectif n'est plus de deviner les noms de tables et de colonnes, mais de les **découvrir** en interrogeant les tables système, puis d'extraire les identifiants et de se connecter en administrateur.

Sur les bases non-Oracle (MySQL, Microsoft, PostgreSQL), les métadonnées de la base sont exposées dans le schéma standard **`information_schema`** :
- `information_schema.tables` : la liste des tables.
- `information_schema.columns` : la liste des colonnes de chaque table.

C'est une chaîne d'attaque complète, qui combine toutes les briques vues avant.

## Objectif

Trouver la table contenant les identifiants, lire ses colonnes, extraire le mot de passe de l'administrateur, et se connecter.

## Démarche

### Étape 1 : nombre de colonnes

`ORDER BY` incrémenté jusqu'à l'erreur → la requête renvoie **2 colonnes**.

### Étape 2 : lister les tables (en ciblant, pas en balayant)

Plutôt que d'afficher les centaines de tables (dont une majorité de tables système), je filtre directement avec `WHERE` + `LIKE` pour ne garder que les tables de l'application :

```
' UNION SELECT table_name, NULL FROM information_schema.tables WHERE table_name LIKE 'users%'-- -
```

Remarque importante : dans ce lab, la table des utilisateurs n'a pas un nom fixe comme `users`, mais un nom **aléatoire** du type `users_xycnen`. D'où l'intérêt du filtre `LIKE 'users%'` pour la retrouver tout de suite.

### Étape 3 : lister les colonnes de cette table

Une fois le nom exact trouvé, je liste ses colonnes :

```
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name = 'users_xycnen'-- -
```

J'obtiens des noms eux aussi aléatoires, par exemple `username_teicbi` et `password_djhkyq`.

Point clé : `information_schema` me donne directement les **noms** des colonnes. Je n'ai donc pas besoin de tester chaque colonne pour savoir laquelle contient du texte : le nom me dit déjà laquelle prendre.

### Étape 4 : extraire les identifiants

Avec les vrais noms de table et de colonnes :

```
' UNION SELECT username_teicbi, password_djhkyq FROM users_xycnen-- -
```

La page affiche la liste des comptes avec leurs mots de passe, dont celui de `administrator`.

### Étape 5 : se connecter

Je récupère le mot de passe de l'administrateur et je me connecte via le formulaire de login. La page affiche "Your username is: administrator" et le lab passe en "Solved".

## Erreurs rencontrées et corrigées

- Au départ j'avais utilisé `SELECT *`, ce qui ne convient pas pour une attaque UNION : il faut un nombre de colonnes fixe et connu, donc on nomme explicitement les colonnes (`table_name`, `column_name`...).
- J'avais aussi interrogé une table système au hasard (`table_privileges`) au lieu de la table applicative : il faut cibler la table des utilisateurs, pas une table système.
- La connexion a d'abord échoué à cause d'une erreur de recopie du mot de passe. Ces mots de passe aléatoires doivent être copiés-collés directement depuis la page, jamais retapés (confusion possible entre `l` et `1`, `0` et `O`).

## Payloads utilisées

```
Lister les tables ciblées :
' UNION SELECT table_name, NULL FROM information_schema.tables WHERE table_name LIKE 'users%'-- -

Lister les colonnes de la table trouvée :
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name = 'users_xycnen'-- -

Extraire les identifiants :
' UNION SELECT username_teicbi, password_djhkyq FROM users_xycnen-- -
```

(Les suffixes `_xycnen`, `_teicbi`, `_djhkyq` sont aléatoires et changent à chaque instance du lab.)

## Résultat

Récupération du mot de passe de l'administrateur, connexion réussie en tant qu'administrateur, lab "Solved".

## Ce que j'en retiens

- `information_schema` permet de **cartographier** une base inconnue : d'abord les tables (`information_schema.tables`), puis les colonnes (`information_schema.columns`).
- On cible au lieu de balayer : un filtre `WHERE table_name LIKE 'users%'` évite de parcourir des centaines de tables système.
- Les noms de tables/colonnes peuvent être aléatoires : il faut les découvrir, pas les supposer.
- Lire les noms de colonnes dans `information_schema` évite de tester chaque colonne pour le type texte.
- C'est la chaîne d'attaque complète d'une fuite de base : identifier le SGBD → compter les colonnes → lister tables → lister colonnes → extraire → se connecter.

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

En complément : les mots de passe doivent être hachés (bcrypt, argon2) et non stockés en clair, et les comptes applicatifs devraient avoir des droits minimaux sur la base pour limiter ce qu'une injection peut atteindre.
