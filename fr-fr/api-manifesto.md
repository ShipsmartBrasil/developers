# Manifesto

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Exécute le flux synchrone de devis + expédition.
- Après la création, peut retourner les documents finaux de l'opération.
- Associé au flux opérationnel principal.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Conventions
- Query optionnelle `format` : `pdf` ou `zpl`.
- Le flux utilise `billed` comme type de paiement dans le contrat publié.

## Flux d'utilisation
1. Envoyez les données complètes du manifesto.
2. L'API effectue le devis, crée l'expédition et génère les documents.
3. Si `format` est `pdf` ou `zpl`, la réponse favorise ce format.

## Endpoints
### Créer un manifesto
- Méthode et route : `POST /v1/manifest`
- Query params: `format`.

### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `format` | query | string | non | Type de retour | `pdf` | `pdf` ou `zpl` |
| `object` | body | string | oui | Objet | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | oui | Finalité | `Commercial` | Provient du flux métier |
| `tax` | body | string | oui | Taxe | `sender` | `sender` ou `receiver` |
| `insurance` | body | boolean | oui | Assurance | `true` | - |
| `residential_delivery` | body | boolean | oui | Livraison résidentielle | `false` | - |
| `non_stackable` | body | boolean | oui | Non empilable | `false` | - |
| `measurement` | body | string | oui | Système de mesure | `metric` | `metric` ou `imperial` |
| `ior` | body | boolean | oui | IOR | `false` | - |
| `currency_quote` | body | string | oui | Devise du devis | `USD` | - |
| `currency_payment` | body | string | oui | Devise de paiement | `EUR` | - |
| `address_sender` | body | object | oui | Expéditeur | - | Manuel ou `address_code` |
| `address_receiver` | body | object | oui | Destinataire | - | Manuel ou `address_code` |
| `boxes` | body | array | oui | Colis | - | Obligatoire |
| `items` | body | array | oui | Articles | - | Obligatoire sauf `doc` |
| `payment.type` | body | string | oui | Paiement | `billed` | Manifesto accepte uniquement le facturé |
| `declared_freight` | body | object | non | Fret déclaré | - | Optionnel |

### Payload minimum
```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Florida",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Payload maximum
```json
{
  "format": "pdf",
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto de teste",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Warehouse",
    "street": "Main Street",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678901",
    "state_tax_id": "1234567890",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida",
    "description": "Office",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "federal_tax_id": "7418529630",
    "state_tax_id": "7418529586",
    "foreign": true
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100,
      "selling_unit_value": 110
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Exemple de response
```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "000000",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    }
  }
}
```

## Règles métier
- Le manifesto fonctionne uniquement avec un paiement facturé.
- Le document final peut être PDF, ZPL ou les deux, selon le flux et le transporteur.
- Si le devis est incompatible avec la version, l'exécution échoue.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | Erreur du transporteur ou règle métier | Ajuster le payload ou vérifier le retour du transporteur |
| `422` | Validation | Corriger les champs obligatoires |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- Cet endpoint est synchrone et concentre plus de responsabilité que les flux séparés de devis + expédition.
- Recommandé uniquement pour les intégrations qui ont réellement besoin du flux complet en un seul appel.
