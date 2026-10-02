# SQL injection dans une clause WHERE permettant la récupération de données cachées

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection
**Niveau :** Apprentice
**Lien du lab :** https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data

## Contexte

L'application est une boutique en ligne qui affiche des produits classés par catégorie. Quand on clique sur une catégorie, le navigateur envoie une requête du type :

```
GET /filter?category=Gifts
```

Côté serveur, l'application construit une requête SQL pour récupérer les produits de la catégorie demandée :

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

La condition `released = 1` sert à n'afficher que les produits déjà commercialisés. Les produits non sortis (`released = 0`) restent donc cachés.

Le paramètre `category` est inséré directement dans la requête SQL sans aucune protection. C'est le point d'injection.

## Objectif

Afficher tous les produits, y compris ceux qui ne sont pas encore sortis, en contournant la condition `released = 1`.

## Démarche

### Repérage du point d'injection

Le paramètre vulnérable est `category`, car sa valeur part telle quelle dans la clause WHERE. J'ai d'abord vérifié que l'entrée n'était pas filtrée en ajoutant une apostrophe :

```
/filter?category=Tech gifts'--
```

La page a affiché des produits supplémentaires, dont des produits cachés comme "Beat the Vacation Traffic". L'injection fonctionne.

### Comprendre les deux approches possibles

Deux façons de résoudre le lab.

**Approche 1 : commenter la fin de la requête**

En injectant une apostrophe suivie de `--`, on ferme la chaîne de la catégorie puis on transforme le reste de la requête en commentaire :

```
/filter?category=Gifts'--
```

La requête devient :

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

Le `--` neutralise `AND released = 1`. Résultat : tous les produits de la catégorie Gifts s'affichent, cachés inclus. Mais cela reste limité à une seule catégorie.

**Approche 2 : forcer une condition toujours vraie**

Pour afficher tous les produits de toutes les catégories, j'utilise une condition toujours vraie :

```
/filter?category=Lifestyle' OR 1=1--
```

La requête devient :

```sql
SELECT * FROM products WHERE category = 'Lifestyle' OR 1=1--' AND released = 1
```

Comme `1=1` est toujours vrai, la clause WHERE renvoie toutes les lignes de la table `products`, sans distinction de catégorie ni de statut de sortie.

## Payload utilisée

```
Lifestyle' OR 1=1--
```

URL complète envoyée :

```
/filter?category=Lifestyle%27+OR+1=1--
```

## Résultat

Tous les produits se sont affichés, y compris les produits non commercialisés. Le lab est passé en "Solved".

## Ce que j'en retiens

- Une injection SQL repose sur trois éléments : le bon point d'injection, la bonne route, et la bonne payload. Si un seul manque, rien ne se passe.
- Tous les paramètres ne sont pas injectables. Ici `category` l'était car il attend du texte libre, alors qu'un paramètre numérique validé en amont aurait résisté.
- La différence entre les deux approches : `'--` supprime une condition gênante, `OR 1=1` ajoute une condition toujours vraie. L'une restreint moins, l'autre ouvre tout.
- `--` est le marqueur de commentaire en SQL : tout ce qui suit sur la ligne est ignoré par la base.

## Remédiation

La faille vient de la concaténation directe de l'entrée utilisateur dans la requête. La bonne pratique est d'utiliser des requêtes paramétrées (requêtes préparées), où la donnée est traitée comme une valeur et jamais comme du code SQL :

```sql
SELECT * FROM products WHERE category = ? AND released = 1
```

La valeur de `category` est alors passée séparément, et une apostrophe injectée n'a plus aucun effet sur la structure de la requête.
