# Documents d'expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Récupère les documents générés après le traitement de l'expédition.
- Peut renvoyer PDF, ZPL, pièces jointes et documents incorporés.
- Ce module est publié en v1.

## Endpoints
- `GET /v1/shipping/{shipping}/docs`
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

## Notes
- `attachments`, `embedded` et `base64` acceptent des booléens en query.
- `documents` accepte une liste séparée par des virgules.
