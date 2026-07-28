# Expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée, met à jour, annule et consulte des expéditions.
- Expose aussi la consultation de documents et de status/suivi.
- C'est la version héritée la plus complète du flux de shipping.
## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue optionnel : `lang`.

## Conventions
- `Content-Type: application/json`
- L'expédition utilise `quotation` comme lien avec le devis.
- Le `quotation` doit être de la même version que l'API d'expédition.
- `payment.type` définit le modèle financier de l'envoi.

## Flux d'utilisation
1. Générez le devis.
2. Vérifiez s'il est éligible pour l'expédition.
3. Envoyez l'expédition avec le `quotation` retourné.
4. Consultez `status` jusqu'à ce que l'expédition soit traitée.
5. Téléchargez les documents dans `/docs`.
6. Si nécessaire, planifiez une collecte.

## Règles métier
- Un devis `package`/`pallet` avec type `simple` n'est pas éligible pour expédition.
- Le devis ne peut pas être expiré.
- L'expédition ne peut pas avoir été créée précédemment pour le même devis.
- Le type de paiement doit être activé pour l'utilisateur.
- Pour DHL avec expéditeur PJ au Brésil, `nfkey` peut être obligatoire.
- Après sa création, l'expédition est irréversible dans l'opération normale.
- Les modifications ultérieures ne sont admises que dans des cas exceptionnels : envoi du numéro de la nota fiscal lorsqu'il n'a pas été renseigné et envoi de documents à joindre à l'expédition lorsqu'ils n'ont pas été envoyés lors de la création.

## Endpoints
### Créer une expédition
- Méthode et route : `POST /v1/shipping`
- Quand l'utiliser : après avoir obtenu un devis éligible.

#### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | oui | ID du devis | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Doit être de la première version |
| `carrier_code` | body | integer | oui | Contrat/transporteur | `1` | Doit exister pour le devis |
| `address_sender` | body | object | oui | Expéditeur | - | Peut utiliser `address_code` ou des champs manuels |
| `address_receiver` | body | object | oui | Destinataire | - | Peut utiliser `address_code` ou des champs manuels |
| `payment.type` | body | string | oui | Type de paiement | `billed` | `billed`, `gateway`, `afterward` |
| `type` | body | string | non | Finalité de l'envoi | `Commercial` | Optionnel |
| `nfkey` | body | string | non | Clé NF-e | `12345678901234567890123456789012345678901234` | 44 chiffres |
| `invoice_number` | body | string | non | Numéro de facture | `NF-123` | Si non envoyé, utilise le code interne |
| `shipment_reference` | body | string | non | Référence de l'envoi | `REF-2024-01` | Si non envoyé, utilise l'ID interne |
| `attachments` | body | array | non | Pièces jointes | - | PDF, XML, JPEG, PNG |
| `biller` | body | object | non | Facturant | - | Utilisé dans l'invoice |

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
    "name": "Doe Jhon",
    "email": "doejhon@example.com",
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
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ],
  "address_sender": {
    "address_code": 1
  },
  "address_receiver": {
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
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
  }
}
```

#### Exemple de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "Doe Jhon", "email": "doejhon@example.com", "phone": "11992494597", "type": "pf"},
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

### Consulter le status de l'expédition
- Méthode et route : `GET /v1/shipping/{shipping}/status`
- `shipping` accepte : UUID du devis, AWB ou clé NF-e de 44 chiffres.

### Consulter le status par canal
- Méthode et route : `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Consulter les documents
- Méthode et route : `GET /v1/shipping/{shipping}/docs`
- Retourne les documents, pièces jointes et, si autorisé, docs intégrés.

### Consulter les documents par canal
- Méthode et route : `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Mettre à jour l'expédition
- Méthode et route : `PUT /v1/shipping/{shipping}`
- Utilisez pour joindre un document et/ou lier une NF-e dans un cas exceptionnel.
- Les fichiers joints doivent être en `base64` et accepter uniquement `PDF`, `XML`, `JPEG` ou `PNG`.
- La taille maximale est de `5 MB` par fichier.

#### Exemple de request
```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "attachments": [
      {
        "description": "Nota Fiscal",
        "base64": "JVBERi0xLjcK..."
      }
    ]
  }'
```

#### Exemple de response
```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Annuler l'expédition
- Méthode et route : `DELETE /v1/shipping/{shipping}`
- `reason` est optionnel dans le body et accepte du texte libre.

#### Exemple de request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

#### Exemple de response
```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

### Supprimer des pièces jointes
- Méthode et route : `DELETE /v1/shipping/{shipping}/docs`
- Envoyez `attachments` avec `id` ou `name`.

#### Exemple de request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

#### Exemple de response
```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!"
}
```

## Consultations et documents
### Paramètres de documents
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `attachments` | query | boolean | non | Inclut les pièces jointes | `true` | Default `true` |
| `embedded` | query | boolean | non | Inclut les docs intégrés | `true` | Default `true` |
| `base64` | query | boolean | non | Inclut le base64 | `true` | Default `true` |
| `documents` | query | string | non | Filtre les types | `invoice_pdf,awb_pdf` | Liste séparée par des virgules |

### Types de documents identifiés
- `awb_pdf`
- `awb_zpl`
- `awb_prc_pdf`
- `awb_prc_zpl`
- `awb_prc_fedex_pdf`
- `awb_prc_fedex_zpl`
- `declaration_pdf`
- `invoice_pdf`
- `non_dangerous_pdf`
- `packing_list_pdf`

### Exemple de consultation de documents
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | Expédition introuvable ou règle métier | Vérifier `quotation`, `carrier_code` et l'éligibilité |
| `401` | Paiement non autorisé | Ajuster la configuration du compte |
| `404` | Ressource introuvable | Confirmer l'identifiant renseigné |
| `422` | Validation | Corriger le payload |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- L'expédition v1 est le point d'intégration le plus complet du cycle opérationnel.
- Consultez `api-documentos.md` et `api-rastreio.md` pour des exemples dédiés de consultation et de suivi.
