# Brouillon

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée, liste, met à jour et supprime les brouillons d'expédition.
- Le brouillon est traité de façon asynchrone et peut évoluer en cotation + expédition.

## Conventions
- Les brouillons acceptent des payloads partiels.
- Le statut peut être `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` ou `error`.

## Flux d'utilisation
1. Envoyez le brouillon avec les données déjà disponibles.
2. Attendez le traitement asynchrone.
3. Listez ou consultez les brouillons.
4. Mettez à jour uniquement les brouillons `incomplete`.
5. Supprimez les brouillons si nécessaire.

## Endpoints
- `POST /v1/draft`
- `PUT /v1/draft/{code}`
- `GET /v1/draft`
- `DELETE /v1/draft`
