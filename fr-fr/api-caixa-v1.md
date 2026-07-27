# Boîte

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Gère l'enregistrement et la consultation des boîtes utilisées dans les cotations, brouillons et expéditions.
- Utiliser pour standardiser dimensions, poids et identifiant des emballages.

## Conventions
- `Content-Type: application/json`
- La consultation utilise `boxes[box_ids][]`.
- Création/mise à jour asynchrone.

## Flux d'utilisation
1. Listez les boîtes disponibles.
2. Enregistrez ou mettez à jour les boîtes de base.
3. Supprimez les boîtes si nécessaire.
4. Réutilisez les codes dans les cotations et brouillons.

## Endpoints
- `GET /v1/box`
- `POST /v1/box`
- `DELETE /v1/box`
