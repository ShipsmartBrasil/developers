# Cotation V2

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Génère des cotations avec un contrat simplifié pour les nouvelles intégrations.
- À utiliser quand seule la cotation v2 est nécessaire.
- Le détail complet n'existe pas dans cette version selon la source analysée.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Homologation: `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans `Authorization`.
- Langue: `lang: pt|en|es`.

## Conventions
- `Content-Type: application/json`
- `level` existe à la création et contrôle la réponse.

## Flux d'utilisation
1. Créez la cotation en v2.
2. Réutilisez le `quotation` pour l'expédition v2.
3. Il n'existe pas de consultation de cotation en v2 dans la source analysée.

## Fonctionnel
- v2 documente explicitement `items`.
- v2 simplifie les champs d'adresse, mais la validation garde les données nécessaires.
- Le résultat détaillé revient directement dans la réponse du `POST`.

## Endpoint
- `POST /v2/quotation`
