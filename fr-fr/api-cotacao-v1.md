# Cotation

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Génère des cotations internationales avec détail du fret, de l'assurance et des taxes.
- À utiliser avant de créer l'expédition.

## Base URL
- Production: `https://api.shipsmart.global`
- Documentation: `https://api.shipsmart.global/docs`
- Homologation: `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans `Authorization`.
- Langue acceptée: `lang: pt|en|es`.

## Flux d'utilisation
1. Envoyez la cotation avec origine, destination, type d'objet et colis/articles.
2. Recevez le code `quotation`.
3. Consultez le résultat avec `GET /v1/quotation/{quotation}/{level}`.
4. Si éligible, réutilisez la même cotation pour créer l'expédition.
