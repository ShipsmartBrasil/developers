# Manifeste

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Exécute le flux synchrone cotation + expédition.
- Après création, peut retourner les documents finaux de l'opération.

## Conventions
- `format` optionnel: `pdf` ou `zpl`.
- Le contrat publié utilise `billed` comme type de paiement.

## Flux d'utilisation
1. Envoyez les données complètes du manifeste.
2. L'API calcule la cotation, crée l'expédition et génère les documents.
3. Si `format` est `pdf` ou `zpl`, la réponse privilégie ce format.

## Endpoint
- `POST /v1/manifest`
