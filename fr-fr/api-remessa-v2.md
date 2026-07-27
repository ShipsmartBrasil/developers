# Expédition V2

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée des expéditions avec un contrat simplifié.
- La source analysée n'expose que la création en v2.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Homologation: `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans `Authorization`.
- Langue: `lang`.

## Conventions
- `Content-Type: application/json`
- `quotation` doit être créé en v2.
- `payment.type` continue de définir le flux financier.

## Flux d'utilisation
1. Créez la cotation en v2.
2. Envoyez l'expédition avec le `quotation` renvoyé.
3. Continuez le flux avec les endpoints v1 lorsque nécessaire.

## Endpoint
- `POST /v2/shipping`

## Différences avec v1
- v2 ne laisse que l'endpoint de création dans la documentation publique.
- `attachments` accepte seulement PDF ou XML dans la source analysée.
