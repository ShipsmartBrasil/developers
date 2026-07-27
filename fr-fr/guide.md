# Guide Technique d'Intégration

## Accès
- Documentation: `https://api.shipsmart.global/docs`
- Base URL: `https://api.shipsmart.global`
- Homologation: `https://hmlapi.shipsmart.com.br`

## Résumé
- La cotation vient avant la création de l'envoi.
- V1 est le flux complet.
- V2 est le flux simplifié pour la cotation et l'envoi, avec une entrée limitée au pays.
- L'envoi est irréversible après création, sauf pour la NF-e et les pièces jointes dans des cas exceptionnels.
- Le suivi renvoie `timeline` et `status` ensemble.
- Le manifeste s'utilise quand un flux synchrone est nécessaire.

## Sujets
- Cotation
- Envoi
- Documents d'envoi
- Suivi
- Planification de collecte
- Manifeste
- Brouillons
- Cartons
- Articles
- Contrats
