# SQL injection aveugle (blind) avec réponses conditionnelles

**Plateforme :** PortSwigger Web Security Academy
**Catégorie :** SQL injection (blind / aveugle)
**Niveau :** Practitioner
**Lien du lab :** https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses

## Contexte

Jusqu'ici, mes injections renvoyaient directement des données dans la page (via UNION, ou le message d'erreur). Ici c'est différent : l'application utilise bien le cookie `TrackingId` dans une requête SQL, mais **elle n'affiche jamais le résultat de cette requête**, et elle ne renvoie pas d'erreur non plus. C'est ça une injection "aveugle" : je ne vois pas la donnée, je dois la deviner.

Le seul indice exploitable : quand le `TrackingId` correspond à quelque chose de connu, la page affiche un petit message **"Welcome back!"**. Quand ce n'est pas le cas, le message disparaît. Ce comportement, présent ou absent, me sert d'**oracle** : une réponse à des questions oui/non que je pose à la base.

## Objectif

Récupérer le mot de passe de l'administrateur caractère par caractère, uniquement à partir de ce "présent/absent" du message, puis me connecter.

## Démarche

### Étape 1 : confirmer l'oracle

Le point d'injection est le cookie `TrackingId`. Je teste deux conditions opposées :

```
TrackingId=XXXX' AND '1'='1      -> "Welcome back!" présent   = VRAI
TrackingId=XXXX' AND '1'='2      -> "Welcome back!" absent    = FAUX
```

Ça marche : avec une condition vraie le message est là, avec une condition fausse il disparaît. J'ai donc un détecteur VRAI/FAUX fiable. Tout le reste va s'appuyer là-dessus.

### Étape 2 : vérifier que la cible existe

Avant de brute-forcer, je m'assure que la table `users` et le compte `administrator` existent :

```
TrackingId=XXXX' AND (SELECT 'x' FROM users WHERE username='administrator')='x
```

"Welcome back" apparaît -> la table et le compte sont bien là. Fondation ok.

### Étape 3 : trouver la longueur du mot de passe

Je ne peux pas lire le mot de passe, mais je peux demander "est-ce que sa longueur est supérieure à N ?" :

```
TrackingId=XXXX' AND (SELECT username FROM users WHERE username='administrator' AND LENGTH(password)>N)='administrator
```

Tester tous les N à la main serait long, donc j'automatise avec **Burp Intruder** :
- Attack type : **Sniper**
- Payload : **Numbers**, de 1 à 25
- Je repère les réponses VRAIES soit via un **Grep - Match** sur `Welcome back`, soit plus simplement via la colonne **Length** (les réponses VRAIES sont plus longues que les FAUSSES, parce qu'elles contiennent le message en plus).

Résultat : `LENGTH(password)>19` est VRAI, `LENGTH(password)>20` est FAUX. Le mot de passe fait donc **exactement 20 caractères**.

### Étape 4 : extraire chaque caractère

Maintenant je pose, pour chaque position, la question "le caractère n°i est-il la lettre X ?" :

```
TrackingId=XXXX' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),i,1)='X
```

Deux variables (la position `i` et la lettre `X`), donc dans Intruder :
- Attack type : **Cluster bomb** (teste toutes les combinaisons)
- Position 1 (l'index) : **Numbers**, de 1 à 20
- Position 2 (la lettre) : **Brute forcer**, jeu de caractères `abcdefghijklmnopqrstuvwxyz0123456789`, longueur min/max = 1
- **Grep - Match** sur `Welcome back` pour marquer les réponses VRAIES

Ça fait 20 x 36 = **720 requêtes**. La version Community de Burp est bridée, donc ça prend quelques minutes.

### Étape 5 : reconstituer et se connecter

Une fois l'attaque finie, je trie par **Length** (ou par la colonne Welcome back) : les 20 réponses VRAIES ressortent. Chacune me donne un couple **position + lettre**. Je remets les lettres dans l'ordre des positions 1 à 20, j'obtiens le mot de passe complet, et je me connecte via le formulaire avec `administrator`. Le lab passe en "Solved".

## Erreurs rencontrées et corrigées

- Au début je testais dans **Repeater** en mettant les marqueurs `§1§` directement dans la requête. Erreur : les `§` ne sont pas du SQL, ils cassent la requête. Ces marqueurs s'ajoutent **uniquement dans Intruder** (bouton Add §), pas à la main dans Repeater.
- Le **Grep - Match** ne fonctionne que si on clique bien sur **Add** après avoir tapé `Welcome back` (sinon le texte reste dans le champ sans être pris en compte). La première fois j'avais oublié, donc pas de colonne Welcome back ; je m'en suis sorti en lisant la colonne **Length** à la place.
- Burp plantait avec l'erreur **"Stream failed to close correctly"** (bug HTTP/2). Corrigé en forçant le **HTTP/1** (Inspector -> Request attributes -> Protocol HTTP/1, ou dans les Settings en décochant "Default to HTTP/2 if the server supports it").
- L'instance du lab a **expiré** en cours de route (erreur 504 Gateway Timeout). Il a fallu relancer le lab (nouvelle URL + nouveau TrackingId) et refaire le circuit sur la nouvelle instance.

## Payloads utilisées

```
Oracle VRAI / FAUX :
' AND '1'='1
' AND '1'='2

Vérifier la cible :
' AND (SELECT 'x' FROM users WHERE username='administrator')='x

Longueur du mot de passe (Intruder Sniper, N de 1 à 25) :
' AND (SELECT username FROM users WHERE username='administrator' AND LENGTH(password)>N)='administrator

Extraction caractère par caractère (Intruder Cluster bomb) :
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),i,1)='X
```

## Résultat

Mot de passe de l'administrateur reconstitué caractère par caractère (20 caractères), connexion réussie en tant qu'administrateur, lab "Solved".

## Ce que j'en retiens

- En blind SQLi, on ne lit pas la donnée : on la **devine** via un comportement observable (ici le message "Welcome back"), qui sert d'oracle VRAI/FAUX.
- La méthode est toujours la même : trouver l'oracle, puis poser des questions oui/non (longueur, puis chaque caractère) avec `LENGTH()` et `SUBSTRING()`.
- `SUBSTRING((...),i,1)` isole le i-ème caractère, et on le compare à chaque valeur possible jusqu'à tomber sur le VRAI.
- Burp Intruder sert à **automatiser** ces centaines de questions. L'outil ne crée pas la faille, il exploite une vulnérabilité qui existe déjà : sur un site en requêtes paramétrées, la même attaque ne donne rien.
- La colonne **Length** est un bon plan B quand le Grep Match n'est pas configuré : les réponses VRAIES sont plus longues car elles contiennent le message en plus.

## Remédiation

Comme pour toute injection SQL, la correction passe par des requêtes paramétrées, qui traitent l'entrée utilisateur comme une donnée et jamais comme du code SQL :

```sql
SELECT ... FROM tracking WHERE trackingId = ?
```

En complément : hachage des mots de passe (bcrypt, argon2) plutôt qu'un stockage en clair, et droits minimaux pour le compte applicatif sur la base afin de limiter ce qu'une injection peut atteindre, même aveugle.
