# SQL injection UNION attack : déterminer le nombre de colonnes

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (UNION attacks)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns

## Contexte

L'application affiche des produits filtrés par catégorie, via le paramètre `category` sur la route `/filter` :

```
GET /filter?category=Accessories
```

Côté serveur, la requête ressemble à :

```sql
SELECT ... FROM products WHERE category = 'Accessories' AND released = 1
```

Le paramètre `category` est injectable. L'objectif de ce lab n'est pas encore d'extraire des données, mais de préparer une attaque UNION en déterminant d'abord une information clé : le nombre de colonnes renvoyées par la requête.

## Objectif

Déterminer le nombre exact de colonnes retournées par la requête d'origine. C'est l'étape obligatoire avant toute attaque UNION, car une requête `UNION SELECT` doit renvoyer le même nombre de colonnes que la requête initiale.

## Démarche

### Placement de l'injection

Point important : l'injection doit se trouver dans la valeur du paramètre, après le `=`, pas dans son nom. La forme correcte est :

```
/filter?category=Accessories' ORDER BY 1--
```

et non `category'...=Accessories`, qui casse simplement le nom du paramètre sans injecter dans la requête.

### Méthode 1 : ORDER BY

J'incrémente le numéro de colonne dans une clause `ORDER BY` jusqu'à provoquer une erreur :

```
Accessories' ORDER BY 1--   -> OK
Accessories' ORDER BY 2--   -> OK
Accessories' ORDER BY 3--   -> OK
Accessories' ORDER BY 4--   -> Internal Server Error
```

`ORDER BY 4` demande de trier sur une 4ᵉ colonne qui n'existe pas, donc la base renvoie une erreur. Le dernier numéro qui fonctionne, **3**, correspond au nombre de colonnes.

### Méthode 2 : UNION SELECT NULL (confirmation)

Je confirme avec une requête UNION contenant autant de `NULL` que de colonnes supposées :

```
' UNION SELECT NULL,NULL,NULL--
```

La requête passe sans erreur, ce qui confirme qu'il y a bien **3 colonnes**. Si j'avais mis 2 ou 4 `NULL`, j'aurais eu une erreur de nombre de colonnes.

J'utilise `NULL` et pas une vraie valeur parce que `NULL` est compatible avec n'importe quel type de colonne (texte, nombre, date). À ce stade je cherche seulement le bon nombre de colonnes, pas encore à afficher des données, donc je veux éviter toute erreur de type.

## Payloads utilisées

```
Accessories' ORDER BY 4--          (provoque l'erreur, révèle 3 colonnes)
' UNION SELECT NULL,NULL,NULL--     (confirme les 3 colonnes, résout le lab)
```

## Résultat

La requête UNION avec 3 `NULL` s'exécute sans erreur. Le lab passe en "Solved".

## Ce que j'en retiens

- Avant toute attaque UNION, il faut connaître le nombre de colonnes de la requête d'origine, sinon le `UNION SELECT` échoue.
- Deux méthodes se recoupent : `ORDER BY n` incrémenté jusqu'à l'erreur, et `UNION SELECT NULL,...` avec le bon nombre de `NULL`. Les faire toutes les deux permet de valider le résultat.
- Une erreur serveur n'est pas un échec, c'est un signal : ici elle indique qu'on a dépassé le nombre de colonnes.
- `NULL` sert de valeur "neutre" compatible avec tous les types, idéale pour cette phase de reconnaissance.
- Détail pratique : l'injection doit être dans la valeur du paramètre, pas dans son nom.

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, où l'entrée utilisateur est traitée comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

Avec une requête paramétrée, une valeur comme `Accessories' ORDER BY 4--` serait cherchée telle quelle comme nom de catégorie, sans jamais modifier la structure de la requête.
