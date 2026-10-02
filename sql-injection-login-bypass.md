# SQL injection permettant le contournement de l'authentification

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection
**Niveau :** Apprentice
**Lien du lab :** https://portswigger.net/web-security/sql-injection/lab-login-bypass

## Contexte

L'application dispose d'un formulaire de connexion classique avec un champ username et un champ password. Quand l'utilisateur soumet le formulaire, l'application vérifie les identifiants avec une requête SQL du type :

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

Si la requête renvoie au moins une ligne, la connexion est acceptée. Si elle renvoie zéro ligne, elle est refusée. L'application ne fait donc pas une vraie comparaison, elle demande simplement à la base s'il existe un utilisateur avec ce nom ET ce mot de passe.

Les deux champs sont liés par un `AND` : les deux conditions doivent être vraies en même temps.

## Objectif

Se connecter en tant qu'utilisateur `administrator` sans connaître son mot de passe.

## Démarche

### Repérage de la requête

Contrairement au lab précédent où les données passaient dans l'URL (requête GET), ici les identifiants sont envoyés dans le corps de la requête, en POST. On ne peut donc pas injecter depuis la barre d'adresse. La donnée part quand on remplit le formulaire.

En interceptant la requête de connexion avec Burp Suite, on voit le corps du POST :

```
csrf=...&username=wiener&password=bluecheese
```

Le point d'injection est le champ `username`.

### Construction de l'injection

L'idée est de fermer la chaîne du username juste après `administrator`, puis de commenter le reste de la requête pour faire disparaître la vérification du mot de passe.

Payload injectée dans le champ username :

```
administrator'--
```

La requête côté serveur devient :

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```

L'apostrophe ferme la chaîne du username, et le `--` transforme tout ce qui suit en commentaire. La partie `AND password = ''` est donc ignorée. La requête se résume à :

```sql
SELECT * FROM users WHERE username = 'administrator'
```

La base renvoie la ligne de l'administrateur, et l'application considère la connexion comme réussie.

## Payload utilisée

```
Username : administrator'--
Password : (n'importe quoi)
```

## Résultat

Connexion réussie en tant qu'administrator. La page My Account affiche "Your username is: administrator" et le lab est passé en "Solved".

## Troubleshooting

J'ai d'abord voulu réaliser l'attaque via Burp Suite Repeater. Je suis tombé sur un problème technique : l'envoi de la requête en HTTP/2 échouait avec le message "Stream failed to close correctly", et aucune réponse ne revenait. C'est un comportement lié à la gestion du HTTP/2 dans cette version de Burp, pas à l'injection elle-même.

Pistes de contournement identifiées :
- Forcer le protocole en HTTP/1 via l'onglet Raw (modifier `HTTP/2` en `HTTP/1.1` sur la première ligne de la requête).
- Encoder manuellement l'apostrophe en `%27` dans le corps POST, car Burp n'encode pas automatiquement.

Finalement, le lab se résout aussi directement depuis le navigateur en remplissant le formulaire, car le navigateur construit la requête POST et encode l'apostrophe tout seul. C'est la méthode que j'ai utilisée pour valider.

## Ce que j'en retiens

- La différence GET / POST est centrale. Une injection dans l'URL ne fonctionne que pour un paramètre envoyé en GET. Un formulaire de login envoie ses données en POST, dans le corps de la requête, invisible dans la barre d'adresse.
- Le login repose sur une logique binaire : la requête renvoie une ligne (succès) ou zéro ligne (échec). Faire sauter la condition du mot de passe suffit à être authentifié.
- Dans une requête brute (Burp), l'apostrophe doit être encodée en `%27`. Le navigateur le fait automatiquement, pas un outil comme Burp.
- Un bug d'outil n'est pas un échec de la technique. Savoir diagnostiquer et contourner un problème d'outil fait partie du travail.

## Remédiation

Même principe que pour toute injection SQL : utiliser des requêtes paramétrées plutôt que de concaténer l'entrée utilisateur.

```sql
SELECT * FROM users WHERE username = ? AND password = ?
```

En complément, les mots de passe ne devraient jamais être comparés en clair dans une requête. Ils doivent être hachés (avec un algorithme adapté comme bcrypt), et la vérification se fait côté application en comparant les empreintes, pas directement dans le SQL.
