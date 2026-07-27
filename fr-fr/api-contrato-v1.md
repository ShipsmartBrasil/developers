# Contrat

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Liste les contrats disponibles pour l'utilisateur authentifié.
- Les contrats sont utilisés dans la cotation et la création d'expéditions.

## Conventions
- `Content-Type: application/json`
- La liste peut filtrer par `status`.

## Flux d'utilisation
1. Authentifiez l'utilisateur.
2. Listez les contrats disponibles.
3. Utilisez le code du contrat dans la cotation ou l'expédition.

## Endpoint
- `GET /v1/contract`
