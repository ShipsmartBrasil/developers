# Expédition (v2)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md) et [Cotation (v2)](api-cotacao-v2.md)

## Vue d'ensemble
- Transforme une cotation **v2** en un envoi réel.
- La différence principale par rapport à l'[Expédition v1](api-remessa-v1.md) : ici vous indiquez l'**adresse complète**
  (ville, état et code postal), parce que la cotation v2 n'a reçu que le pays.
- La v2 n'expose **que la création**. Statut, documents, annulation, mise à jour et planification d'enlèvement
  utilisent les endpoints `/v1/...`, qui fonctionnent normalement pour les expéditions créées en v2.

> ⚠️ **L'expédition est une opération irréversible sur la route normale.** Une fois créée, elle entre en traitement
> chez le transporteur.

## Base URL
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
Content-Type: application/json
lang: pt
```

## Flux d'utilisation

```
1. POST /v2/quotation                 → quotation + carriers[]   (pays/état seuls)
2. POST /v2/shipping                  → expédition reçue         (adresse complète)
3. GET  /v1/shipping/{shipping}/status→ suivre jusqu'à "shipped"
4. GET  /v1/shipping/{shipping}/docs  → télécharger AWB, invoice, etc.
5. POST /v1/schedule                  → planifier l'enlèvement (optionnel)
```

1. [Créez la cotation v2](api-cotacao-v2.md) et vérifiez qu'elle est éligible à l'expédition.
2. Envoyez l'expédition avec le `quotation`, le transporteur choisi et l'adresse complète.
3. La création est **asynchrone** : `200` signifie « nous avons reçu », pas « nous avons expédié ».
4. Suivez via le [suivi](api-rastreio.md) et téléchargez les [documents](api-documentos.md).

## Carte des endpoints par version

| Opération | Route |
|---|---|
| Créer l'expédition | `POST /v2/shipping` |
| Consulter le statut | `GET /v1/shipping/{shipping}/status` |
| Consulter le statut par canal | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` |
| Consulter les documents | `GET /v1/shipping/{shipping}/docs` |
| Consulter les documents par canal | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` |
| Mettre à jour (NF-e, pièces jointes, centre de coûts) | `PUT /v1/shipping/{shipping}` |
| Supprimer des pièces jointes | `DELETE /v1/shipping/{shipping}/docs` |
| Annuler | `DELETE /v1/shipping/{shipping}` |
| Annuler par canal | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` |
| Planifier un enlèvement | `POST /v1/schedule` |

> Il n'existe ni `GET /v2/shipping/...` ni `DELETE /v2/shipping/...`. Utilisez les routes `/v1`.

## Endpoint : créer une expédition

- **Méthode et route :** `POST /v2/shipping`
- **Quand l'utiliser :** après avoir obtenu une cotation **v2** éligible, dans son délai de validité.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **oui** | - | - | - | UUID de la cotation. Elle doit avoir été créée en **v2** |
| `carrier_code` | integer | **oui** si `slug` n'est pas envoyé | - | - | - | Code du contrat (`carriers[].code` de la cotation) |
| `slug` | string | **oui** si `carrier_code` n'est pas envoyé | - | 20 | - | Identifiant textuel du contrat (`carriers[].slug` de la cotation) |
| `address_sender` | object | **oui** | - | - | - | Expéditeur (voir tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | Destinataire (voir tableau ci-dessous) |
| `payment` | object | **oui** | - | - | - | Mode de paiement |
| `type` | string | non | - | 30 | `Commercial` | Finalité de l'envoi. Valeurs : `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | non | 44 | 44 | `null` | Clé de la NF-e. **Exactement 44 chiffres** |
| `invoice_number` | string | non | 1 | 50 | code interne | Numéro de la facture/invoice |
| `shipment_reference` | string | non | 1 | 50 | ID interne | Référence de l'envoi |
| `cost_center_code` | integer | non | - | - | `null` | Code du centre de coûts. Il doit exister et être autorisé pour votre utilisateur |
| `payment_notify_shipping_billing` | boolean | non | - | - | `false` | Notifier le service de facturation de la facturation |
| `attachments` | array | non | - | - | `null` | Documents joints à l'envoi |
| `biller` | object | non | - | - | préréglage du compte | Facturant affiché sur l'invoice à la place de l'expéditeur |

### Paramètres — `address_sender` et `address_receiver`

**C'est la principale différence de la v2 :** ville, état et code postal sont obligatoires ici.

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code de l'adresse enregistrée. Lorsqu'il est utilisé, n'envoyez pas les autres champs |

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `city` | string | **oui** | 3 | 100 | - | Ville |
| `postal_code` | string | conditionnel | - | 12 | - | Code postal/ZIP. Obligatoire si `state` n'est pas indiqué. **Toujours obligatoire pour les adresses au Brésil** |
| `state` | string | conditionnel | - | - | - | Nom de l'état/province. Obligatoire si `postal_code` n'est pas indiqué |
| `state_code` | string | conditionnel | - | - | - | Sigle de l'état/province. Obligatoire si `state` et `postal_code` ne sont pas indiqués |
| `street` | string | **oui** | - | 70 | - | Rue |
| `name` | string | **oui** | 5 | 35 | - | Nom du contact/raison sociale |
| `email` | string | **oui** | - | 60 | - | E-mail valide (reçoit les notifications du transporteur) |
| `phone` | string | **oui** | - | 25 | - | Téléphone |
| `type` | string | **oui** | - | - | `pj` | `pf` (personne physique) ou `pj` (personne morale) |
| `number` | string | non | - | 6 | `null` | Numéro du bâtiment |
| `district` | string | non | - | 30 | `null` | Quartier |
| `complement` | string | non | - | 20 | `null` | Complément |
| `description` | string | non | - | 155 | `null` | Libellé interne de l'adresse |
| `phone_extension` | string | non | - | 5 | `null` | Poste |
| `federal_tax_id` | string | conditionnel | - | 20 | `null` | CPF/CNPJ ou document fédéral. **Obligatoire quand le pays est `BR` ou `AR`** |
| `state_tax_id` | string | conditionnel | - | 20 | `null` | Numéro d'immatriculation de l'état. **Obligatoire quand le pays est `BR`** |
| `foreign` | boolean | conditionnel | - | - | `false` | Indique si la personne est étrangère. **Obligatoire quand le pays est `BR` et que `address_code` n'est pas utilisé** |

> **Règle pratique :** indiquez `city` + (`postal_code` **ou** `state_code`) + `street` + `name` + `email` +
> `phone` + `type`. Le pays vient de la cotation.

### Paramètres — `payment`

| Champ | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|
| `payment.type` | string | **oui** | `billed`, `gateway`, `afterward` | Modèle financier de l'envoi |

| Valeur | Signification |
|---|---|
| `billed` | Facturé : le montant entre dans votre facture périodique |
| `gateway` | Paiement immédiat via gateway, dans la devise définie par `currency_payment` |
| `afterward` | Paiement ultérieur, réglé en dehors du flux automatique |

### Paramètres — `attachments[]`

| Champ | Type | Obligatoire | Max. | Description |
|---|---|---|---|---|
| `description` | string | **oui** | 100 | Nom/description du document (utilisé ensuite pour localiser ou supprimer la pièce jointe) |
| `base64` | string | **oui** | - | Contenu du fichier encodé en Base64 |

| Limite | Valeur |
|---|---|
| Formats acceptés | `PDF`, `XML`, `JPEG`, `PNG` |
| Taille par fichier | **5 MB** |

### Paramètres — `biller` (facturant)

Lorsqu'elle est indiquée, cette adresse apparaît sur l'invoice à la place de l'adresse de l'expéditeur.
Envoyez `address_code` **ou** les champs manuels. **Si vous n'utilisez pas de facturant, n'envoyez pas le champ.**

| Champ | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `address_code` | integer | - | - | Code d'adresse enregistrée |
| `country_code` | string | 2 | 2 | Pays en ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nom de l'état |
| `state_code` | string | - | 10 | Sigle de l'état |
| `postal_code` | string | - | 12 | Code postal/ZIP |
| `city` | string | 3 | 100 | Ville |
| `street` | string | - | 70 | Rue |
| `number` | string | - | 6 | Numéro |
| `district` | string | - | 30 | Quartier |
| `complement` | string | - | 20 | Complément |
| `description` | string | - | 155 | Libellé interne |
| `name` | string | 5 | 35 | Nom/raison sociale |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Téléphone |
| `phone_extension` | string | - | 5 | Poste |
| `type` | string | - | - | `pf` ou `pj` |
| `federal_tax_id` | string | - | 20 | Document fédéral |
| `state_tax_id` | string | - | 20 | Numéro d'immatriculation de l'état |
| `foreign` | boolean | - | - | Indique si la personne est étrangère |

### Payload minimal

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "state_code": "FL",
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

### Payload maximal

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "slug": "dhl-br",
  "type": "Commercial",
  "nfkey": "12345678901234567890123456789012345678901234",
  "invoice_number": "NF-000123",
  "shipment_reference": "REF-2026-0001",
  "cost_center_code": 1,
  "payment_notify_shipping_billing": true,
  "address_sender": {
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centre de distribution SP",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Hangar 2",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Bureau Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Siege",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Bureau 1",
    "name": "Holding Exemple LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcKJcTl8uXrp..."
    },
    {
      "description": "XML de la Nota Fiscal",
      "base64": "PD94bWwgdmVyc2lvbj0i..."
    }
  ]
}
```

### Exemple de requête

```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rue Principale", "name": "Boutique Exemple LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "state_code": "FL", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

## Différences par rapport à la v1

| Thème | v1 | v2 |
|---|---|---|
| Adresse dans l'expédition | Rue, nom, e‑mail, téléphone, type (+ documents fiscaux) | Les mêmes champs **+ `city`, `state`/`state_code`, `postal_code`** |
| `city` | Non accepté (vient de la cotation) | **Obligatoire**, min. 3 et max. 100 caractères |
| `biller` | Champs de localité acceptés | Champs de localité **validés** (`city` min. 3) |
| Endpoints du cycle | Création, statut, documents, mise à jour, pièces jointes, annulation | **Création uniquement.** Le reste utilise `/v1` |
| Origine de la cotation | `POST /v1/quotation` | `POST /v2/quotation` |

## Règles métier

### Éligibilité de la cotation
| Condition | Message en cas d'échec |
|---|---|
| La cotation existe et appartient au compte | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Cotation non expirée | `Esta cotação está expirada...` |
| Cotation non utilisée auparavant | `Cotação já foi utilizada em outra remessa!` |
| Créée en **v2** | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Ce n'est pas une cotation `simple` de `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| Le contrat figure dans la liste de la cotation | `Contrato especificado não foi listado para esta cotação!` |

### Cohérence entre cotation et expédition
Le pays indiqué dans l'expédition doit être le même que celui de la cotation :
- `O país :country é diferente do enviado na cotação!`
- `O Estado :state é diferente do enviado na cotação!`
- `A cidade :city é diferente do enviado na cotação!`

### Documents fiscaux obligatoires
| Pays de l'adresse | `federal_tax_id` | `state_tax_id` | `foreign` |
|---|---|---|---|
| `BR` | Obligatoire (CPF ou CNPJ valide) | Obligatoire | Obligatoire sans `address_code` |
| `AR` | Obligatoire | - | - |
| Autres | Optionnel | Optionnel | Optionnel |

- Pour `type: pf`, le numéro d'immatriculation de l'état ne peut pas être `ISENTO`/`ISENTA`.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier (cotation, adresse, paiement, document fiscal) | Lire `message` et corriger le scénario |
| `401` | Paiement non autorisé | Activer le type de paiement sur le compte |
| `403` | Sans permission pour l'action | Demander la permission au support |
| `404` | `Contrato especificado não foi listado para esta cotação!` | Vérifier `carrier_code`/`slug` |
| `422` | Validation des champs ou cotation de version incompatible | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Utilisez la v2 quand l'adresse complète n'existe qu'au moment de l'envoi.
- Tout le cycle opérationnel après la création se fait via les endpoints `/v1` — ils ne distinguent pas la version
  dans laquelle l'expédition a été créée.
- Pour le détail de chaque opération après la création, voir [Expédition (v1)](api-remessa-v1.md),
  [Suivi](api-rastreio.md), [Documents](api-documentos.md) et [Planification](api-agendamento.md).
