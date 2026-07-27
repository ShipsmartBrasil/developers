# Expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée, met à jour, annule et consulte les expéditions.
- Expose aussi les documents et le suivi.
- C'est le flux legacy le plus complet.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Homologation: `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans `Authorization`.
- Langue optionnelle: `lang`.

## Conventions
- `Content-Type: application/json`
- L'expédition utilise `quotation` comme lien avec la cotation.
- La cotation doit appartenir à la même version d'API.
- `payment.type` définit le modèle financier.

## Flux d'utilisation
1. Créez la cotation.
2. Vérifiez l'éligibilité.
3. Envoyez l'expédition avec le `quotation`.
4. Consultez `status` jusqu'au traitement.
5. Téléchargez les documents dans `/docs`.
6. Si nécessaire, planifiez la collecte.

## Règles métier
- `package`/`pallet` avec type `simple` ne sont pas éligibles.
- La cotation ne peut pas être expirée.
- L'expédition ne peut pas exister déjà pour la même cotation.
- Le type de paiement doit être autorisé pour l'utilisateur.
- Une fois créée, l'expédition est irréversible dans le fonctionnement normal.
- Les modifications ultérieures ne sont admises que dans des cas exceptionnels.

## Endpoints
- `POST /v1/shipping`
- `GET /v1/shipping/{shipping}/status`
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`
- `GET /v1/shipping/{shipping}/docs`
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`
- `PUT /v1/shipping/{shipping}`
- `DELETE /v1/shipping/{shipping}`
