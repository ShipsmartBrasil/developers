# Comparatif cotation et expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Explique la différence pratique entre les flux de cotation et d'expédition dans les deux versions publiées.
- Utilisez ce guide pour choisir la bonne version avant l'intégration.

## Règle d'utilisation
- Le flux standard utilise les endpoints de la première version.
- La seconde version sert à une cotation plus simple, avec moins de données initiales.

## Cotation
### Différence pratique
- La première version accepte le flux complet avec origine, destination, boîtes et articles.
- La deuxième version est plus légère pour les intégrations qui démarrent avec moins de données.

## Expédition
### Différence pratique
- La première version couvre création, statut, documents, annulation, mise à jour et pièces jointes.
- La deuxième version n'expose que la création dans la source analysée.
