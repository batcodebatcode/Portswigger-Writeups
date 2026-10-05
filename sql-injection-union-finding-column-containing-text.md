# SQL injection UNION attack : trouver une colonne contenant du texte

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (UNION attacks)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text

## Contexte

Même type d'application que les labs UNION précédents : les produits sont filtrés par catégorie via le paramètre `category` sur la route `/filter`, et ce paramètre est injectable.

```
GET /filter?category=Accessories
```

## Objectif

Le lab fournit une chaîne aléatoire (dans mon cas `ILmpRI`) et demande de la faire apparaître dans les résultats. Pour cela, il faut identifier une colonne de la requête capable d'afficher du texte, puis y injecter cette chaîne via une attaque UNION.

## Démarche

### Étape 1 : retrouver le nombre de colonnes

Même si c'est un nouveau lab, la première étape reste la même que pour l'attaque UNION de base. Je confirme le nombre de colonnes avec `ORDER BY`, en incrémentant jusqu'à l'erreur :

```
Accessories' ORDER BY 1--
Accessories' ORDER BY 2--
Accessories' ORDER BY 3--
```

Comme pour le lab précédent, j'obtiens **3 colonnes**.

Point important que j'ai compris ici : les exemples donnés dans l'énoncé ne sont que des illustrations. Il ne faut pas recopier bêtement le nombre de `NULL` d'un exemple, mais l'adapter au nombre réel de colonnes du lab en cours.

### Étape 2 : tester chaque colonne pour trouver celle qui accepte du texte

Je garde 3 valeurs (pour mes 3 colonnes), toutes en `NULL` sauf une que je remplace par une chaîne de test. Je déplace la chaîne de position en position :

```
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```

Si une colonne n'accepte pas de texte (par exemple une colonne numérique), la requête renvoie une erreur "Internal Server Error". Si la colonne accepte du texte, la chaîne s'affiche dans la page sans erreur.

Dans ce lab, la **colonne 2** accepte le texte.

### Étape 3 : afficher la chaîne demandée

Une fois la bonne colonne trouvée, je remplace la chaîne de test par celle que le lab demande (`ILmpRI`), à la position de la colonne texte :

```
Accessories' UNION SELECT NULL,'ILmpRI',NULL--
```

La chaîne apparaît dans les résultats, ce qui résout le lab.

## Erreurs rencontrées et corrigées

Mes premiers essais renvoyaient "Internal Server Error" à cause de deux fautes de syntaxe, pas à cause de la logique :
- Une virgule manquante entre deux valeurs (`'a' NULL` au lieu de `'a',NULL`).
- Un nombre de valeurs qui ne correspondait pas aux 3 colonnes (j'avais mis 4 valeurs).

Dès que le nombre de valeurs correspondait exactement aux 3 colonnes et que la syntaxe était correcte, l'injection passait.

## Payload utilisée

```
Accessories' UNION SELECT NULL,'ILmpRI',NULL--
```

## Résultat

La chaîne `ILmpRI` s'affiche dans la page. Le lab passe en "Solved".

## Ce que j'en retiens

- Une attaque UNION se fait en deux temps : d'abord trouver le nombre de colonnes, ensuite trouver laquelle accepte le type de donnée voulu (ici du texte).
- On utilise `NULL` pour compter les colonnes car il est compatible avec tous les types. On passe au texte seulement pour localiser la colonne affichable.
- Le nombre de valeurs dans le `UNION SELECT` doit correspondre exactement au nombre de colonnes, sinon erreur.
- Les exemples de l'énoncé sont des illustrations à adapter, pas des solutions à recopier.
- Vocabulaire : `ORDER BY` compte des colonnes (la structure de la table), pas des catégories (des valeurs de données).

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM products WHERE category = ? AND released = 1
```

Avec une requête paramétrée, une valeur comme `Accessories' UNION SELECT NULL,'ILmpRI',NULL--` serait recherchée telle quelle comme nom de catégorie, sans jamais être exécutée comme une seconde requête.
