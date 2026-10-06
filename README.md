# PortSwigger Web Security Academy - Write-ups

![Plateforme](https://img.shields.io/badge/Plateforme-PortSwigger-orange)
![Domaine](https://img.shields.io/badge/Domaine-S%C3%A9curit%C3%A9%20Web-blue)
![Labs résolus](https://img.shields.io/badge/Labs%20r%C3%A9solus-6-brightgreen)

Write-ups personnels des labs que je résous sur la [Web Security Academy](https://portswigger.net/web-security) de PortSwigger, dans le cadre de ma formation en sécurité offensive web.

Chaque write-up suit la même structure : le contexte du lab, la démarche suivie, la payload utilisée, ce que j'en retiens, et la remédiation côté développeur. L'idée n'est pas seulement de montrer l'exploitation, mais de prouver que je comprends la vulnérabilité et comment la corriger.

## Progression

### SQL injection

| # | Lab | Niveau | Write-up |
|---|-----|--------|----------|
| 1 | Retrieval of hidden data (clause WHERE) | Apprentice | [Lire](sql-injection-where-clause-hidden-data.md) |
| 2 | Login bypass | Apprentice | [Lire](sql-injection-login-bypass.md) |
| 3 | UNION attack : déterminer le nombre de colonnes | Practitioner | [Lire](sql-injection-union-determine-number-of-columns.md) |
| 4 | UNION attack : trouver une colonne contenant du texte | Practitioner | [Lire](sql-injection-union-finding-column-containing-text.md) |
| 5 | UNION attack : extraire des données d'autres tables | Practitioner | [Lire](sql-injection-union-retrieving-data-from-other-tables.md) |
| 6 | UNION attack : récupérer plusieurs valeurs dans une seule colonne | Practitioner | [Lire](sql-injection-union-retrieving-multiple-values-single-column.md) |

## Ce que couvrent ces write-ups

- Injection dans une clause `WHERE` (requête GET) et récupération de données cachées
- Contournement d'authentification via injection dans un formulaire (requête POST)
- Différence GET / POST et identification du point d'injection
- Neutralisation de conditions avec les commentaires SQL (`--`) et les conditions toujours vraies (`OR 1=1`)
- Remédiation par requêtes paramétrées

## Organisation

Les write-ups sont regroupés par catégorie de vulnérabilité. De nouvelles catégories seront ajoutées au fil de ma progression (XSS, authentification, path traversal, etc.).

## À propos

Je me forme à la sécurité offensive web. Ce dépôt documente mon apprentissage, lab par lab, et sera mis à jour régulièrement.
