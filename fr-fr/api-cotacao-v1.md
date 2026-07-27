# Cotation

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Génère des cotations internationales avec détail du fret, de l'assurance et des taxes.
- À utiliser pour simuler l'envoi avant de créer l'expédition.
- La cotation est la base pour l'expédition, le suivi et les documents.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Homologation: `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans `Authorization`.
- Langue acceptée: `lang: pt|en|es`.
- La langue par défaut est `pt`.

## Conventions
- `Content-Type: application/json`
- Dates au format `Y-m-d H:i:s` dans les réponses.
- Devises au format ISO 4217 à 3 lettres.
- `level` contrôle le niveau de détail du résultat: `simple` ou `detailed`.

## Flux d'utilisation
1. Envoyez une cotation avec origine, destination, objet, type et articles/boîtes.
2. Recevez le `quotation` renvoyé par l'API.
3. Consultez le résultat avec `GET /v1/quotation/{quotation}/{level}`.
4. Si la cotation est éligible, utilisez le même `quotation` pour créer l'expédition.

## Fonctionnel
- `simple`: calcule par boîte avec poids/valeur direct.
- `advanced`: calcule avec boîtes et articles distribués par `box_id`.
- `items`: la validation backend accepte ce type, mais le contrat OpenAPI ne l'expose pas clairement.

## Éligibilité pour l'expédition
- La cotation doit exister, ne pas être expirée, appartenir à la même version d'API d'expédition et ne pas avoir été utilisée.
- `package`/`pallet` avec type `simple` ne sont pas éligibles.
- L'expédition exige aussi un contrat compatible et un type de paiement autorisé.
