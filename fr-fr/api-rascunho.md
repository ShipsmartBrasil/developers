# Brouillon

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée, liste, met à jour et supprime des brouillons d'envoi.
- Le brouillon est traité de manière asynchrone et peut évoluer vers devis + expédition.
- Associé au flux opérationnel principal.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Conventions
- Le brouillon accepte des payloads partiels.
- Le `status` du brouillon peut être `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` ou `error`.

## Flux d'utilisation
1. Envoyez le brouillon avec les données que vous possédez déjà.
2. Attendez le traitement asynchrone.
3. Listez ou consultez les brouillons.
4. Mettez à jour uniquement les brouillons avec le status `incomplete`.
5. Supprimez les brouillons lorsque nécessaire.

## Endpoints
### Créer un brouillon
- Méthode et route : `POST /v1/draft`

### Mettre à jour un brouillon
- Méthode et route : `PUT /v1/draft/{code}`
- Accepte uniquement l'édition d'un brouillon `incomplete`.

### Lister les brouillons
- Méthode et route : `GET /v1/draft`
- Peut filtrer par `drafts.drafts_ids`.

### Supprimer un brouillon
- Méthode et route : `DELETE /v1/draft`

### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | non | Canal | `1` | - |
| `channel_identifier_number` | body | string | non | Identifiant commercial | `334455` | - |
| `carrier_code` | body | integer | non | Transporteur préféré | `1` | - |
| `object` | body | string | non | Objet | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | non | Type | `Commercial` | - |
| `tax` | body | string | non | Taxe | `sender` | - |
| `insurance` | body | boolean | non | Assurance | `true` | - |
| `measurement` | body | string | non | Mesures | `metric` | Default peut provenir d'un preset |
| `currency_quote` | body | string | non | Devise du devis | `USD` | - |
| `currency_payment` | body | string | non | Devise du paiement | `BRL` | - |
| `address_sender` | body | object | non | Expéditeur | - | Accepte le remplissage partiel |
| `address_receiver` | body | object | non | Destinataire | - | Accepte le remplissage partiel |
| `boxes` | body | array | non | Colis | - | Optionnel |
| `items` | body | array | non | Articles | - | Optionnel |
| `payment.type` | body | string | non | Type de paiement | `billed` | `billed`, `afterward` |

### Exemple de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP", "city": "Diadema"},
    "address_receiver": {"country_code": "US", "city": "Florida"},
    "boxes": [{"height": 10, "width": 10, "depth": 10}],
    "payment": {"type": "billed"}
  }'
```

### Exemple de response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno"
}
```

## Règles métier
- Un brouillon sans payload utile retourne warning.
- Le backend utilise les presets utilisateur lorsqu'ils sont disponibles.
- Les brouillons peuvent être convertis en devis et expédition par le traitement asynchrone.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | Brouillon vide, introuvable ou non modifiable | Envoyer des données valides ou choisir un autre brouillon |
| `422` | Validation | Corriger le payload |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- Le brouillon est la meilleure option lorsque l'intégration ne dispose pas encore de toutes les données finales.
- Consultez `api-cotacao-v1.md` et `api-remessa-v1.md` pour comprendre ce que le traitement asynchrone va générer.
