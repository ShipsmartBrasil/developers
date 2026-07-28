# Devis V2

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Génère des devis avec un contrat simplifié pour les nouvelles intégrations.
- Utilisez lorsque vous souhaitez uniquement créer le devis en v2.
- La consultation détaillée du résultat n'existe pas dans cette version dans la source analysée.
- Version documentée : v2.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang: pt|en|es`.

## Conventions
- `Content-Type: application/json`
- `level` existe dans le request body/query de création et contrôle le retour du calcul.
- HTTP les plus courants : `200`, `400`, `422`, `500`.

## Flux d'utilisation
1. Envoyez le devis en v2.
2. Recevez le `quotation` et utilisez ce code pour créer l'expédition en v2.
3. Il n'y a pas d'endpoint de consultation de devis en v2 dans la source analysée.

## Endpoints
### Créer un devis
- Méthode et route : `POST /v2/quotation`
- Quand l'utiliser : nouveau flux de tarification.
- Règles importantes : `object`, `type`, `tax`, `insurance`, `currency_quote`, `currency_payment`, `address_sender`, `address_receiver`, `boxes` et/ou `items` selon le type.

#### Différences fonctionnelles par rapport à la v1
- La v2 documente explicitement `items` comme type de devis.
- La v2 simplifie les champs d'adresse dans le contrat OpenAPI, mais la validation continue d'exiger les données nécessaires selon l'origine/la destination.
- Le résultat détaillé est fourni dans le `POST` lui-même.

#### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `object` | body | string | oui | Objet envoyé | `package` | `doc`, `package`, `pallet`, `not_doc` dans la source |
| `type` | body | string | oui | Type du devis | `items` | `simple`, `advanced`, `items` |
| `tax` | body | string | oui | Taxe | `sender` | `sender` ou `receiver` |
| `insurance` | body | boolean | oui | Assurance | `true` | - |
| `currency_quote` | body | string | oui | Devise du devis | `USD` | ISO 4217 |
| `currency_payment` | body | string | oui | Devise du paiement | `EUR` | ISO 4217 |
| `address_sender` | body | object | oui | Expéditeur | - | `address_code` ou manuel |
| `address_receiver` | body | object | oui | Destinataire | - | `address_code` ou manuel |
| `boxes` | body | array | oui | Colis/palettes | - | Obligatoire sauf quand `type=items` |
| `items` | body | array | oui/conditionnel | Articles | - | Les règles dépendent du type |
| `manufacturer` | body | object | non | Fabricant | - | Optionnel |
| `declared_freight` | body | object | non | Fret déclaré | - | Optionnel |

#### Payload minimum
```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ]
}
```

#### Payload maximum
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 8, "width": 8, "depth": 8 },
    { "box_code": 3, "height": 10, "width": 10, "depth": 10 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100
    }
  ],
  "manufacturer": {
    "name": "Factory LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj"
  }
}
```

#### Exemple de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/quotation \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "simple",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "BRL",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

#### Exemple de response
```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "object": "package",
    "type": "advanced",
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "freight": "181.95",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

#### Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | Transporteur indisponible ou règle refusée | Ajuster pays, poids, colis ou contrat |
| `422` | Validation | Corriger les champs obligatoires |
| `406` | Structure incompatible | Réviser le payload et la version |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- La documentation OpenAPI de la v2 et la validation de request ne sont pas identiques dans tous les détails. Valider avec l'équipe responsable de l'API.
- Pour la consultation détaillée du devis, la source analysée ne fournit pas d'endpoint v2 équivalent à `GET /v1/quotation/{quotation}/{level}`.
- Toutes les actions de devis et d'expédition dans la deuxième version peuvent être suivies via les autres APIs de la première version.
