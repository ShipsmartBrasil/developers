# Expédition V2

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée des expéditions avec un contrat simplifié.
- La v2 n'expose pas, dans la source analysée, d'endpoints de status, documents, annulation ou mise à jour.
- Version documentée : v2.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Conventions
- `Content-Type: application/json`
- `quotation` doit avoir été créée en v2.
- `payment.type` continue de définir le flux financier.

## Flux d'utilisation
1. Créez le devis en v2.
2. Envoyez l'expédition avec le `quotation` retourné.
3. Suivez le reste du flux en utilisant les endpoints de la v1, si applicable au contrat hérité.

## Endpoints
### Créer une expédition
- Méthode et route : `POST /v2/shipping`
- Quand l'utiliser : nouveau flux de shipping.
- Règles importantes : le devis doit être v2 ; `payment.type` doit être autorisé pour l'utilisateur.

#### Différences par rapport à la v1
- La v2 conserve l'endpoint de création, mais la documentation publique ne montre pas les autres endpoints du cycle.
- `attachments` en v2 accepte uniquement `PDF` ou `XML` selon la source analysée.
- Le contrat est plus strict pour l'utilisation des champs d'adresse.

#### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | oui | ID du devis | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Doit être v2 |
| `carrier_code` | body | integer | oui | Contrat | `1` | Doit exister |
| `address_sender` | body | object | oui | Expéditeur | - | Accepte des champs manuels ou `address_code` |
| `address_receiver` | body | object | oui | Destinataire | - | Accepte des champs manuels ou `address_code` |
| `payment.type` | body | string | oui | Type de paiement | `billed` | `billed`, `gateway`, `afterward` |
| `attachments` | body | array | non | Pièces jointes | - | PDF/XML dans la source |
| `biller` | body | object | non | Facturant | - | Optionnel |
| `nfkey` | body | string | non | Clé NF-e | - | 44 chiffres |

#### Payload minimum
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

#### Payload maximum
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "nfkey": "12345678901234567890123456789012345678901234",
  "type": "Commercial",
  "invoice_number": "NF-123",
  "shipment_reference": "REF-2024-01",
  "carrier_code": 1,
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
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
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Factory LTDA",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ]
}
```

#### Exemple de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

#### Exemple de response
```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

## Payload minimum et maximum
- Le payload minimum exige uniquement les champs indispensables pour l'identification du devis, du transporteur, des adresses et du paiement.
- Le payload maximum inclut NF-e, pièces jointes, facturant et référence commerciale.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `422` | Validation | Ajuster le contrat et les champs obligatoires |
| `400` | Règle métier | Vérifier l'éligibilité du devis et du paiement |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- La v2 documentée dans la source est concise ; pour le support opérationnel des documents et du suivi, la source analysée pointe vers la v1.
- Toutes les actions de devis et d'expédition dans la deuxième version peuvent être suivies via les autres APIs de la première version.
