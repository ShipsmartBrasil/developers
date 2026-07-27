# Articles

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Gère les articles utilisés dans les cotations, brouillons et expéditions.
- Les articles peuvent être consultés, créés, mis à jour et supprimés en lot.

## Conventions
- `Content-Type: application/json`
- La consultation utilise `items[skus][]`.
- Création/mise à jour/suppression asynchrones.

## Flux d'utilisation
1. Listez les articles déjà enregistrés.
2. Enregistrez les articles de base.
3. Mettez à jour les articles quand dimensions, poids ou classification changent.
4. Supprimez les articles obsolètes.

## Endpoints
- `GET /v1/item`
- `POST /v1/item`
- `PUT /v1/item`
- `DELETE /v1/item`
