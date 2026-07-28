# Devis

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Génère des devis internationaux avec détail du fret, de l'assurance et des taxes.
- Utilisez lorsque vous devez simuler l'envoi avant de créer l'expédition.
- Le devis est la base pour l'expédition, le suivi et les documents.
## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Type : Bearer token.
- Header: `Authorization: Bearer SEU_TOKEN`
- Header additionnel accepté : `lang: pt|en|es`.
- La langue par défaut est `pt`.

## Conventions
- `Content-Type: application/json`
- Dates au format `Y-m-d H:i:s` dans les réponses.
- Devises en code ISO 4217 à 3 lettres.
- `level` contrôle le niveau de détail du résultat : `simple` ou `detailed`.
- Status HTTP courants : `200`, `400`, `404`, `406`, `422`, `500`.

## Flux d'utilisation
1. Envoyez un devis avec origine, destination, objet, type et articles/colis.
2. Recevez le `quotation` retourné par l'API.
3. Consultez le résultat avec `GET /v1/quotation/{quotation}/{level}`.
4. Si le devis est éligible, utilisez le même `quotation` pour créer l'expédition.

## Différences de contrat
| Endpoint | Différence principale | Impact |
|---|---|---|
| `POST /v1/quotation` | Crée le devis | Flux initial |
| `GET /v1/quotation/{quotation}/{level}` | Consulte le résultat | Permet de réviser les carriers |

## Sujets fonctionnels
### Devis simple, avancé et `items`
- `simple` : calcule par colis avec poids/valeur direct. Pour `doc`, c'est le flux naturel.
- `advanced` : calcule avec colis et articles distribués par `box_id`.
- `items` : la validation du backend accepte ce type, mais le contrat OpenAPI ne l'expose pas clairement. Valider avec l'équipe responsable de l'API.

### `package`, `pallet` e `doc`
- `doc` : document. Le devis simple pour documents utilise un colis avec poids et prix.
- `package` : paquet/colis.
- `pallet` : palette. Peut utiliser `volume` au lieu de `height/width/depth`.

### Devis éligible pour expédition
- Le devis doit exister, ne pas être expiré, appartenir au même `api_version` que l'expédition et ne pas encore avoir été utilisé.
- `package`/`pallet` avec type `simple` ne sont pas éligibles pour expédition.
- L'expédition exige aussi un contrat compatible et un type de paiement autorisé.

## Endpoints
### Créer un devis
- Objectif : calculer le fret, l'assurance et les taxes.
- Méthode et route : `POST /v1/quotation`
- Quand l'utiliser : avant de créer une expédition.
- Règles importantes : origine et destination obligatoires ; `currency_quote` et `currency_payment` obligatoires ; `measurement` accepte `metric` ou `imperial`.

#### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | non | Canal de vente | `1` | Doit exister dans l'enregistrement |
| `channel_identifier_number` | body | string | non | Identifiant du canal | `334455` | Libre |
| `object` | body | string | oui | Objet envoyé | `package` | `doc`, `package`, `pallet`, `not_doc` dans la source |
| `type` | body | string | oui | Type du devis | `advanced` | `simple`, `advanced`, `items` dans la validation |
| `tax` | body | string | oui | Qui paie les taxes | `sender` | `sender` ou `receiver` |
| `insurance` | body | boolean | oui | Assurance | `true` | - |
| `measurement` | body | string | non | Système de mesure | `metric` | `metric` ou `imperial` |
| `residential_delivery` | body | boolean | non | Livraison résidentielle | `false` | - |
| `non_stackable` | body | boolean | non | Indique non empilable | `false` | - |
| `ior` | body | boolean | non | Importer of Record | `false` | - |
| `currency_quote` | body | string | oui | Devise du devis | `BRL` | ISO 4217 |
| `currency_payment` | body | string | oui | Devise de paiement | `BRL` | ISO 4217 |
| `address_sender` | body | object | oui | Origine | - | Peut utiliser `address_code` ou des champs manuels |
| `address_receiver` | body | object | oui | Destination | - | Peut utiliser `address_code` ou des champs manuels |
| `boxes` | body | array | oui | Colis/paquets | - | Obligatoire sauf quand `type=items` |
| `items` | body | array | non | Articles de la cargaison | - | Obligatoire en `advanced`/`items` |
| `manufacturer` | body | object | non | Fabricant sur l'invoice | - | Optionnel |
| `declared_freight` | body | object | non | Fret déclaré | `{"declared":true,"value":0,"currency":"USD"}` | Seulement lorsque applicable |
| `level` | query | string | non | Niveau de détail | `simple` | `simple` ou `detailed` |

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
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
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
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
  },
  "boxes": [
    {
      "name": "Caixa 1",
      "height": 20,
      "width": 20,
      "depth": 20
    }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "BR",
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
  --url https://api.shipsmart.global/v1/quotation \
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
    "address_sender": {
      "country_code": "BR",
      "state_code": "SP",
      "postal_code": "09941070",
      "city": "Diadema"
    },
    "address_receiver": {
      "country_code": "US",
      "state_code": "FL",
      "postal_code": "33101",
      "city": "Florida"
    },
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
    "description": null,
    "channel": "Domenic Dooley",
    "channel_identifier_number": "334455",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "valid_until": "2021-09-28 20:28:10",
        "transit_days": 5,
        "freight": "181.95",
        "insurance": "11.45",
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
| `400` | Échec de règle métier | Ajuster le poids, le code postal, l'éligibilité ou le type de devis |
| `406` | Payload incohérent | Vérifier les champs obligatoires et les types |
| `422` | Validation | Corriger les champs obligatoires/formats |
| `500` | Erreur inattendue | Réessayer et contacter l'équipe responsable |

## Consulter un devis
### Méthode et route
- `GET /v1/quotation/{quotation}/{level}`
- `level` obligatoire dans le path : `simple` ou `detailed`.

### Exemple
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Response résumée
```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

## Observations finales
- La réponse peut contenir des `warnings` lorsqu'il y a des alertes d'enregistrement ou de transport.
- La source analysée montre une divergence entre la documentation OpenAPI et la validation dans certains champs. Valider avec l'équipe responsable de l'API avant d'intégrer en production.
