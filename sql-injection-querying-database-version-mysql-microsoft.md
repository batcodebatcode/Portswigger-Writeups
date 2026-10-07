# SQL injection : récupérer le type et la version de la base (MySQL / Microsoft)

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (examiner la base de données)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft

## Contexte

Même point d'injection que les labs précédents : le paramètre `category` sur `/filter` est injectable et les résultats sont affichés. Objectif de reconnaissance : identifier le type de SGBD et sa version. Ici la cible est **MySQL ou Microsoft SQL Server**.

## Objectif

Faire afficher la chaîne de version, par exemple :

```
8.0.42-0ubuntu0.20.04.1
```

## Démarche

### Étape 1 : nombre de colonnes

Je compte les colonnes avec `ORDER BY` en incrémentant jusqu'à l'erreur :

```
Toys & Games' ORDER BY 1-- -
Toys & Games' ORDER BY 2-- -
Toys & Games' ORDER BY 3-- -   -> erreur
```

Le dernier numéro qui fonctionne (2) donne le nombre de colonnes : **2 colonnes**.

### Étape 2 : syntaxe de la version selon le SGBD

Sur MySQL comme sur Microsoft, la version s'obtient avec :

```
SELECT @@version
```

Bon à savoir : `@@version` fonctionne pour **les deux** bases (MySQL et Microsoft), c'est pour ça que ce lab couvre les deux cas avec une seule syntaxe.

Deux points d'attention propres à MySQL :
- Le commentaire `--` doit être **suivi d'un espace** pour être valide. Comme un espace en fin d'URL se fait avaler, on écrit `-- -` (tiret tiret espace tiret) pour forcer l'espace. On peut aussi utiliser `#` (encodé `%23`).

### Étape 3 : la payload

Avec 2 colonnes, `@@version` dans la colonne texte et `NULL` en complément :

```
' UNION SELECT NULL, @@version-- -
```

(Si `NULL, @@version` échoue, on inverse : `@@version, NULL`.)

## Erreur rencontrée et corrigée

Lors d'un essai, j'avais oublié l'apostrophe de fermeture avant `ORDER BY` (`Games ORDER BY` au lieu de `Games' ORDER BY`). Résultat : l'injection ne s'exécutait pas et s'affichait simplement comme du texte dans le titre de la catégorie. Dès que l'apostrophe était remise pour fermer la chaîne, la requête s'exécutait. C'est l'erreur de syntaxe la plus classique à surveiller.

## Payload utilisée

```
' UNION SELECT NULL, @@version-- -
```

## Résultat

La chaîne de version MySQL s'affiche dans la page. Le lab passe en "Solved".

## Ce que j'en retiens

- `@@version` donne la version sur **MySQL et Microsoft** (une seule syntaxe pour les deux).
- Sur MySQL, le commentaire `--` exige un **espace** derrière : on utilise `-- -` (ou `#`).
- La méthode reste la même que pour Oracle, seule la syntaxe change selon le SGBD : d'abord compter les colonnes, puis injecter la requête de version avec le bon dialecte.
- Toujours vérifier l'apostrophe de fermeture : sans elle, l'injection ne s'exécute pas et s'affiche comme du simple texte.

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

Et les messages d'erreur détaillés ne devraient pas remonter à l'utilisateur : des erreurs génériques réduisent les informations qu'un attaquant peut tirer de la base pendant sa reconnaissance.
