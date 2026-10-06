# SQL injection : récupérer le type et la version de la base (Oracle)

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (examiner la base de données)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle

## Contexte

Même point d'injection que les labs UNION : le paramètre `category` sur `/filter` est injectable et les résultats sont affichés. Cette fois, l'objectif n'est plus de voler des identifiants mais de faire parler la base sur elle-même : connaître le **type de SGBD** et sa **version**.

Connaître le SGBD est une étape de reconnaissance importante en pentest : la suite de l'attaque (syntaxe des requêtes, noms des tables système, fonctions disponibles) dépend entièrement du moteur de base utilisé.

## Objectif

Faire afficher les chaînes de version d'Oracle, par exemple :

```
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production
PL/SQL Release 11.2.0.2.0 - Production
...
```

## Démarche

### Le point clé : la syntaxe dépend du SGBD

Chaque base stocke sa version différemment :

```
Microsoft, MySQL  ->  SELECT @@version
Oracle            ->  SELECT banner FROM v$version
PostgreSQL        ->  SELECT version()
```

Comme ce lab est la version **Oracle**, il faut utiliser la syntaxe Oracle. Deux spécificités Oracle à connaître :
- La version est dans la table système **`v$version`**, dans la colonne **`banner`**.
- Sur Oracle, **tout `SELECT` doit avoir un `FROM`** (d'où aussi l'usage de `FROM dual` quand on n'a pas de vraie table à interroger).

### La payload

En réutilisant l'attaque UNION (2 colonnes, dont une en `NULL` pour le bourrage) :

```
' UNION SELECT BANNER, NULL FROM v$version--
```

- `BANNER` : la colonne qui contient le texte de la version.
- `NULL` : complète pour atteindre les 2 colonnes attendues.
- `FROM v$version` : obligatoire sur Oracle.

Si `BANNER, NULL` échoue, on inverse les positions : `NULL, BANNER`.

La vue `v$version` renvoie plusieurs lignes, donc toutes les chaînes de version s'affichent.

## Erreur rencontrée et corrigée

Mon premier réflexe a été d'utiliser `@@version`. Mais `@@version` est la syntaxe **MySQL / Microsoft**, pas Oracle. Sur un lab Oracle, ça provoque un "Internal Server Error". La leçon : toujours vérifier **sur quel SGBD** on travaille (ici c'était écrit dans le titre du lab) et adapter la syntaxe en conséquence.

J'ai aussi d'abord écrit `SELECT * FROM v$version` sans `UNION` et avec le `NULL` dans le `FROM`, ce qui est doublement faux : il faut `UNION SELECT` pour greffer la requête, et le `NULL` va dans la liste des colonnes, pas dans le `FROM`. On ne peut pas non plus utiliser `*` dans une attaque UNION, car il faut un nombre de colonnes fixe et connu : on nomme donc explicitement `banner`.

## Payload utilisée

```
' UNION SELECT BANNER, NULL FROM v$version--
```

## Résultat

Les chaînes de version Oracle s'affichent dans la page. Le lab passe en "Solved".

## Ce que j'en retiens

- Identifier le SGBD est une étape de reconnaissance clé : tout le reste de l'exploitation en dépend.
- La syntaxe pour obtenir la version change selon la base : `@@version` (MySQL/Microsoft), `banner FROM v$version` (Oracle), `version()` (PostgreSQL).
- Spécificité Oracle : tout `SELECT` exige un `FROM` ; pour la version, c'est `FROM v$version`.
- Dans un `UNION SELECT`, on nomme la colonne voulue (`banner`) et on complète les autres avec `NULL` pour respecter le nombre de colonnes. `SELECT *` ne convient pas, car il faut un nombre de colonnes maîtrisé.
- Le nom de colonne (`banner`, `password`...) doit exister dans la table mise dans le `FROM` : le couple table + colonne doit être cohérent.

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

Par ailleurs, les messages d'erreur détaillés ne devraient pas remonter jusqu'à l'utilisateur : des erreurs génériques limitent les informations qu'un attaquant peut tirer de la base pendant sa phase de reconnaissance.
