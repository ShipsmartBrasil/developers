# Suivi d'expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Consulte l'état opérationnel et l'historique d'une expédition.

## Formats de réponse
- Le même endpoint renvoie `timeline` et `status`.
- `timeline` liste les événements dans l'ordre de réception.
- `status` regroupe les événements par statut.

## Endpoints
- `GET /v1/shipping/{shipping}/status`
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`
