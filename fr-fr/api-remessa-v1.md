# Expédition (v1)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md) et [Cotation (v1)](api-cotacao-v1.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Transforme une **cotation v1** en un envoi réel : génère l'AWB, l'invoice, la déclaration et les autres documents.
- C'est le module le plus complet du cycle : création, consultation du statut, consultation des documents, mise à jour,
  suppression de pièces jointes et annulation.
- Les endpoints de statut, de documents, de mise à jour et d'annulation servent **aussi** les expéditions créées via
  l'[Expédition V2](api-remessa-v2.md).
- La création est **asynchrone** : l'API enregistre l'expédition et la prise en charge par le transporteur a lieu ensuite.

> ⚠️ **L'expédition est une opération irréversible sur la route normale.** Une fois créée, elle entre en traitement
> chez le transporteur. Les seules modifications ultérieures admises sont exceptionnelles : lier la NF-e lorsqu'elle
> n'a pas été indiquée, joindre des documents qui n'ont pas été envoyés à la création et indiquer/changer le centre de coûts.

## URL de base
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
1. POST /v1/quotation                  → quotation + carriers[]
2. POST /v1/shipping                   → expédition reçue (asynchrone)
3. GET  /v1/shipping/{shipping}/status → suivre jusqu'à "shipped"
4. GET  /v1/shipping/{shipping}/docs   → télécharger AWB, invoice, etc.
5. POST /v1/schedule                   → planifier l'enlèvement (optionnel)
```

1. [Créez la cotation v1](api-cotacao-v1.md) et vérifiez qu'elle est [éligible](#cotations-non-éligibles-à-lexpédition).
2. Envoyez l'expédition avec le `quotation` et le transporteur choisi (`carrier_code` **ou** `slug`).
3. La création est **asynchrone** : la réponse `200` signifie « nous avons reçu », pas « nous avons expédié ».
4. Consultez le [statut](api-rastreio.md) jusqu'à ce qu'elle apparaisse en `shipped`.
5. Téléchargez les [documents](api-documentos.md).
6. Si nécessaire, [planifiez l'enlèvement](api-agendamento.md).

## Concepts essentiels

### Expédition à partir d'une cotation v1
- La cotation v1 a déjà fourni le **pays, la ville, l'état et le code postal** d'origine et de destination. C'est pourquoi, dans l'expédition v1, vous
  indiquez uniquement **qui** envoie et **qui** reçoit (nom, contact, rue, documents fiscaux).
- Les champs de localité (`city`, `state`, `state_code`, `postal_code`, `country_code`) envoyés dans
  `address_sender`/`address_receiver` sont **ignorés** en v1 : la localité est toujours celle de la cotation.
- Une cotation génère **une seule** expédition.

### L'identifiant `{shipping}`

Dans les endpoints de documents, de mise à jour, de suppression de pièces jointes et d'annulation, le paramètre de path `{shipping}`
accepte **trois formats** :

| Format | Exemple | Remarque |
|---|---|---|
| UUID de la cotation | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forme recommandée. Toute valeur contenant un tiret est traitée comme un UUID |
| Clé de la NF-e | `12345678901234567890123456789012345678901234` | Exactement 44 chiffres |
| Numéro d'AWB | `271249639538` | Expéditions créées au cours des **90 derniers jours** |

> L'endpoint de **statut** n'accepte que l'UUID ou l'AWB. Détails dans [Suivi de l'Expédition](api-rastreio.md).

Pour consulter sans conserver aucune de ces valeurs, utilisez les endpoints **par canal**, basés sur le
`channel_identifier_number` que vous avez indiqué dans la cotation.

## Endpoint : créer une expédition

- **Méthode et route :** `POST /v1/shipping`
- **Quand l'utiliser :** après avoir obtenu une cotation v1 éligible, dans son délai de validité.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **oui** | - | - | - | UUID de la cotation. Elle doit avoir été créée en **v1** |
| `carrier_code` | integer | **oui** si `slug` n'est pas envoyé | - | - | - | Code du contrat (`carriers[].code` de la cotation) |
| `slug` | string | **oui** si `carrier_code` n'est pas envoyé | - | 20 | - | Identifiant textuel du contrat (`carriers[].slug` de la cotation). Lorsqu'il est envoyé, il prévaut sur `carrier_code` |
| `address_sender` | object | **oui** | - | - | - | Expéditeur (voir tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | Destinataire (voir tableau ci-dessous) |
| `payment` | object | **oui** | - | - | - | Mode de paiement (voir tableau ci-dessous) |
| `type` | string | non | - | - | préréglage du compte ou `Commercial` | Finalité de l'envoi. Valeurs : `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | conditionnel | 44 | 44 | `null` | Clé de la NF-e. Lorsqu'elle est indiquée, elle doit comporter **exactement 44 caractères**. Obligatoire pour certains contrats (voir [Règles métier](#restrictions-de-transporteur)) |
| `invoice_number` | string | non | 1 | 50 | code interne | Numéro de la facture/invoice. S'il est omis, le code interne de l'expédition est utilisé |
| `shipment_reference` | string | non | 1 | 50 | ID interne | Référence de l'envoi. Si elle est omise, l'ID interne de l'expédition est utilisé |
| `cost_center_code` | integer | non | - | - | préréglage du compte | Code du centre de coûts. Il doit exister et être autorisé pour votre utilisateur. Vide/nul reçoit le centre de coûts par défaut du compte |
| `payment_notify_shipping_billing` | boolean | non | - | - | préréglage du compte | Avec `payment.type: afterward`, envoie l'e-mail de facturation de l'expédition |
| `attachments` | array | non | - | - | `null` | Documents joints à l'envoi (voir tableau ci-dessous) |
| `biller` | object | non | - | - | préréglage du compte | Facturant affiché sur l'invoice à la place de l'expéditeur (voir tableau ci-dessous) |

> **`carrier_code` ou `slug` :** envoyez **l'un** des deux. Le contrat indiqué doit figurer dans la liste
> `carriers[]` retournée par la cotation ; sinon, l'API répond `404` avec
> `Contrato especificado não foi listado para esta cotação!`.

### Paramètres — `address_sender` et `address_receiver`

Deux formes, ne les mélangez pas :

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code de l'adresse enregistrée. **Elle doit avoir les mêmes pays, état et ville que la cotation** |

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `street` | string | **oui** | - | 70 | - | Rue |
| `name` | string | **oui** | 5 | 35 | - | Nom du contact/raison sociale |
| `email` | string | **oui** | - | 60 | - | E-mail valide (reçoit les notifications du transporteur) |
| `phone` | string | **oui** | - | 25 | - | Téléphone |
| `type` | string | **oui** | - | - | - | `pf` (personne physique) ou `pj` (personne morale) |
| `number` | string | non | - | 6 | `null` | Numéro du bâtiment |
| `district` | string | non | - | 30 | `null` | Quartier |
| `complement` | string | non | - | 20 | `null` | Complément |
| `description` | string | non | - | 155 | `null` | Libellé interne de l'adresse |
| `phone_extension` | string | non | - | 5 | `null` | Poste |
| `federal_tax_id` | string | conditionnel | - | 20 | `null` | CPF/CNPJ ou document fédéral. Obligatoire selon le pays (voir [Documents fiscaux](#documents-fiscaux-obligatoires)) |
| `state_tax_id` | string | conditionnel | - | 20 | `null` | Inscription d'État. Obligatoire pour les Brésiliens (`BR` avec `foreign: false`) |
| `foreign` | boolean | non | - | - | `false` si le pays est `BR` ; `true` dans les autres cas | Indique si la personne est étrangère. Définit la manière dont le `federal_tax_id` est validé au Brésil |

> Le pays, l'état, la ville et le code postal **ne** sont **pas** indiqués ici : ils viennent de la cotation.
> Si vous devez les indiquer dans l'expédition, utilisez l'[Expédition V2](api-remessa-v2.md).

### Paramètres — `payment`

| Champ | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|
| `payment.type` | string | **oui** | `billed`, `gateway`, `afterward` | Modèle financier de l'envoi |

| Valeur | Signification | Exigence du compte |
|---|---|---|
| `billed` | Facturé : le montant est ajouté à votre facture périodique | Option de paiement « facturé » activée |
| `gateway` | Paiement immédiat via passerelle, dans la devise définie dans `currency_payment` | Option de paiement « immédiat » activée |
| `afterward` | Paiement ultérieur, réglé en dehors du flux automatique | - |

> Une valeur hors de la liste retourne `422`. Un type non activé pour votre compte retourne
> `401` avec `Opção de pagamento não autorizado!`.

### Paramètres — `attachments[]`

| Champ | Type | Obligatoire | Max. | Description |
|---|---|---|---|---|
| `description` | string | **oui** | 100 | Nom/description du document (utilisé ensuite pour localiser ou supprimer la pièce jointe) |
| `base64` | string | **oui** | - | Contenu du fichier encodé en Base64 |

| Limite | Valeur |
|---|---|
| Formats acceptés | `PDF`, `XML`, `JPEG`, `PNG` |
| Taille totale de la requête | **90 Mo** |

> ⚠️ **À la création, les pièces jointes invalides sont écartées silencieusement :** un fichier dans un format non reconnu ou dépassant
> la limite ne génère pas d'erreur — l'expédition est créée sans lui. Vérifiez les pièces jointes enregistrées dans
> [Documents](api-documentos.md). Pour joindre des fichiers plus tard, avec validation et messages d'erreur, utilisez
> [mettre à jour une expédition](#endpoint--mettre-à-jour-une-expédition) (limite de **5 Mo** par fichier).

### Paramètres — `biller` (facturant)

Lorsqu'elle est indiquée, cette adresse apparaît sur l'invoice à la place de l'adresse de l'expéditeur.
Envoyez `address_code` **ou** les champs manuels. **Si vous n'utilisez pas de facturant, n'envoyez pas le champ** (ou envoyez-le vide :
dans ce cas, c'est le facturant préconfiguré sur le compte qui s'applique, s'il existe).

| Champ | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `address_code` | integer | - | - | Code d'adresse enregistrée |
| `country_code` | string | 2 | 2 | Pays en ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nom de l'état |
| `state_code` | string | - | 10 | Sigle de l'état |
| `postal_code` | string | - | 12 | Code postal/ZIP |
| `city` | string | - | 100 | Ville |
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
| `state_tax_id` | string | - | 20 | Inscription d'État |
| `foreign` | boolean | - | - | Indique s'il s'agit d'un étranger |

### Payload minimal

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Rue Principale",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

> `federal_tax_id` (CNPJ valide) et `state_tax_id` apparaissent dans le payload minimal parce que l'expéditeur est une
> personne morale brésilienne. Pour un expéditeur d'un autre pays (sauf `MX` et `GR`), ces champs peuvent être omis.

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
    "description": "Centre de distribution SP",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Entrepot 2",
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
    "description": "Bureau Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
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
    "district": "Centro",
    "complement": "Salle 1",
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
    "type": "afterward"
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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Rue Principale", "name": "Boutique Exemple LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` lorsque l'expédition a été enregistrée |
| `message` | string | Confirmation de réception |

> La réponse **ne** contient **ni** AWB **ni** documents : l'expédition a seulement été reçue. Utilisez le
> [suivi](api-rastreio.md) pour la suivre et les [documents](api-documentos.md) pour télécharger les fichiers.
> L'identifiant de l'expédition est le `quotation` lui-même.

### Erreurs spécifiques à la création

| Statut | Message | Cause / correction |
|---|---|---|
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Document fédéral absent alors qu'il est obligatoire (voir [Documents fiscaux](#documents-fiscaux-obligatoires)) |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brésilien sans `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ avec un chiffre de contrôle incorrect |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF avec un chiffre de contrôle incorrect |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` au Brésil avec un `state_tax_id` exonéré |
| `400` | `Numero da nota fiscal é requerido` | Le contrat exige `nfkey` pour ce scénario |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` d'un contrat supprimé/inactif |
| `401` | `Opção de pagamento não autorizado!` | Type de paiement non activé sur votre compte |
| `403` | `Cotação Simples é inelegível para remessa!` | Utilisez `type: advanced` ou `items` dans la cotation |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` dépassé — refaites la cotation |
| `403` | `Has no permission!` | Votre profil n'a pas la permission de créer une expédition |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | `quotation` d'un autre compte |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` absent de la liste de la cotation |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` invalide ou d'un autre compte |
| `406` | `Cotação já foi utilizada em outra remessa!` | Une cotation génère **une** expédition |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` avec un pays différent de celui coté |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` avec un état différent de celui coté |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` avec une ville différente de celle cotée |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inexistant ou créé en v2 (arrive dans `data.quotation`) |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente)...` | Compte verrouillé en DDP avec une cotation DDU non payée |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` non autorisé pour vous |

> Lorsqu'il y a plus d'une erreur de document fiscal, les messages arrivent ensemble dans `message`, séparés par ` | `.

## Endpoint : consulter le statut de l'expédition

- **Méthode et route :** `GET /v1/shipping/{shipping}/status`
- **Par canal :** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `shipping` | path | string | **oui** | UUID de la cotation ou AWB |
| `channel_identifier_number` | path | string | **oui** (variante par canal) | Identifiant commercial indiqué dans la cotation |
| `channel_code` | path | integer | non | Code du canal de vente |

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

> Format de la réponse, groupes de statuts et codes dans [Suivi de l'Expédition](api-rastreio.md).

## Endpoint : consulter les documents

- **Méthode et route :** `GET /v1/shipping/{shipping}/docs`
- **Par canal :** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | UUID, clé NF-e ou AWB |
| `attachments` | query | boolean | non | `true` | Inclure les pièces jointes que vous avez envoyées (exige `base64=true`) |
| `embedded` | query | boolean | non | `true` | Inclure les documents intégrés (nécessite une permission spécifique et `base64=true`) |
| `base64` | query | boolean | non | `true` | Inclure le contenu en Base64 en plus de l'`url` |
| `documents` | query | string | non | tous | Liste séparée par des virgules des types souhaités |

Types acceptés dans `documents` : `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

### Exemple de requête

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

> Format de la réponse et détails de chaque document dans [Documents de l'Expédition](api-documentos.md).

## Endpoint : mettre à jour une expédition

- **Méthode et route :** `PUT /v1/shipping/{shipping}`
- **Quand l'utiliser :** uniquement dans des cas exceptionnels — lier la NF-e, joindre des documents manquants
  ou indiquer/changer/supprimer le centre de coûts.
- **Règle :** envoyez `nfkey` et/ou `attachments` et/ou `cost_center_code`. **Au moins un** doit être présent.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | - | UUID, clé NF-e ou AWB |
| `nfkey` | body | string | conditionnel | 44 | 44 | Clé de la NF-e. **Elle n'est enregistrée que si l'expédition n'a pas encore de NF-e** ; sinon, elle est ignorée |
| `cost_center_code` | body | integer | conditionnel | - | - | Code du centre de coûts autorisé pour votre utilisateur. Envoyer `null` **supprime** le centre de coûts |
| `attachments` | body | array | conditionnel | - | - | Fichiers à joindre |
| `attachments[].description` | body | string | **oui** si `attachments` est envoyé | - | 100 | Description du document |
| `attachments[].base64` | body | string | **oui** si `attachments` est envoyé | - | - | Contenu en Base64. Formats : `PDF`, `XML`, `JPEG`, `PNG`. Max. **5 Mo** par fichier |

### Payload minimal

```json
{
  "cost_center_code": 1
}
```

### Payload maximal

```json
{
  "nfkey": "12345678901234567890123456789012345678901234",
  "cost_center_code": 1,
  "attachments": [
    { "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
    { "description": "Photo du produit", "base64": "iVBORw0KGgoAAAANSUhEUg..." }
  ]
}
```

### Exemple de requête

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` lorsque la mise à jour a été appliquée |
| `message` | string | Confirmation |

| Statut | Message | Cause |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | Aucun des trois champs n'a été envoyé |
| `400` | `Status da remessa não permite atualização na NFE.` | L'expédition a déjà dépassé `shipped` (ou est annulée/en erreur) |
| `400` | `Erro ao salvar arquivo!` | `base64` invalide |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Pièce jointe supérieure à 5 Mo |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` invalide ou d'un autre compte |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Expédition annulée ou dont le cycle est clôturé |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Format de la pièce jointe non accepté |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML mal formé |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Centre de coûts non autorisé |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Erreur inattendue |

> Les modifications sont appliquées dans l'ordre NF-e → centre de coûts → pièces jointes. Si une pièce jointe échoue, la NF-e et le centre
> de coûts envoyés dans le même appel **ont déjà été enregistrés**.

## Endpoint : supprimer des pièces jointes

- **Méthode et route :** `DELETE /v1/shipping/{shipping}/docs`
- **Quand l'utiliser :** pour effacer une pièce jointe envoyée par erreur.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | - | UUID, clé NF-e ou AWB |
| `attachments` | body | array | **oui** | 1 élément | - | Liste des pièces jointes à supprimer |
| `attachments[].id` | body | integer | **oui** si `name` n'est pas envoyé | 1 | - | ID de la pièce jointe (retourné dans `GET .../docs`) |
| `attachments[].name` | body | string | **oui** si `id` n'est pas envoyé | - | 100 | Description exacte de la pièce jointe |

> Envoyez **`id` OU `name`** par élément, jamais les deux.

### Payload minimal

```json
{
  "attachments": [
    { "id": 123456 }
  ]
}
```

### Payload maximal

```json
{
  "attachments": [
    { "id": 123456 },
    { "name": "Nota Fiscal" }
  ]
}
```

### Exemple de requête

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!",
  "data": {
    "deleted": [
      { "id": 123456, "name": "Nota Fiscal" }
    ],
    "failed": [
      { "id": 789, "name": null, "reason": "Nenhum documento encontrado!" }
    ]
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.deleted[]` | array | Pièces jointes supprimées (`id`, `name`) |
| `data.failed[]` | array | Pièces jointes non supprimées, avec le motif dans `reason` |

> L'opération est **partielle** : ce qui a été supprimé apparaît dans `deleted` et ce qui a échoué dans `failed`,
> avec le motif. Vérifiez toujours le tableau `failed`.

| Statut | Message | Cause |
|---|---|---|
| `400` | `Remessa não encontrada!` | UUID sans expédition |
| `404` | `Esta remessa não foi encontrada!` | AWB/NF-e introuvable ou d'un autre compte |
| `422` | Validation | `attachments` vide, ou élément sans `id`/`name` |

## Endpoint : annuler une expédition

- **Méthode et route :** `DELETE /v1/shipping/{shipping}`
- **Par canal :** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Quand l'utiliser :** pour demander l'annulation. La réponse confirme la demande ; la finalisation peut être asynchrone.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | - | UUID, clé NF-e, AWB **ou** `channel_identifier_number` |
| `channel_identifier_number` | path | string | **oui** (variante par canal) | - | - | Identifiant commercial |
| `channel_code` | path | integer | non | - | - | Code du canal. Seules les valeurs **numériques** sont interprétées comme canal |
| `reason` | body | string | non | 10 | 255 | Motif de l'annulation, en texte libre |

> **Comment la route est interprétée :**
> - `DELETE /v1/shipping/{valor}/{numero}` est toujours une **annulation par canal** (`channel_identifier_number` + `channel_code`).
> - `DELETE /v1/shipping/{valor}` recherche d'abord une expédition dont le `channel_identifier_number` est `{valor}` ;
>   si elle n'en trouve pas, elle traite `{valor}` comme un UUID, une clé NF-e ou un AWB.

### Payload minimal

Sans corps :

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer VOTRE_TOKEN'
```

### Payload maximal

```json
{
  "reason": "Annulation de la commande demandee par le client"
}
```

### Exemple de requête

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/PED-2026-0001/1 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"reason":"Annulation de la commande demandee par le client"}'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Solicitação de cancelamento da remessa 9c06fd75-d5f4-40eb-a1a8-f12086f49944 recebida. Em breve será processada; consulte o status para acompanhar."
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` lorsque la demande d'annulation a été acceptée |
| `message` | string | Confirmation, avec l'UUID de l'expédition |

| Statut | Message | Cause |
|---|---|---|
| `400` | `Esta remessa não foi encontrada!` | Identifiant invalide ou expédition d'un autre compte |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | Le statut actuel ne permet pas l'annulation (voir tableau ci-dessous) |
| `422` | Validation | `reason` de moins de 10 ou de plus de 255 caractères ; `channel_code` inexistant |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Erreur inattendue |

**Quand l'annulation est acceptée :**

| Statut actuel de l'expédition | Résultat |
|---|---|
| `received`, `processing_payment`, `paid_out` | Annulée immédiatement (`cancelled`) |
| `shipped` | Passe en `cancelling` et est annulée chez le transporteur ; certains contrats annulent directement (`cancelled`) |
| `error` sans envoi enregistré chez le transporteur | Annulée immédiatement (`cancelled`) |
| `processing_shipment` | Acceptée uniquement pour des contrats spécifiques ; refusée pour les autres |
| Autres (`collecting`, `in_transit`, `delivered`, `cancelled`...) | Refusée avec `Remessa não atende requisitos para cancelamento :id !` |

> Après avoir demandé l'annulation, consultez le [statut](api-rastreio.md) : il passe par `cancelling`
> et, une fois confirmé par le transporteur, devient `cancelled`.

## Cotations non éligibles à l'expédition

Avant d'enregistrer l'expédition, l'API vérifie la cotation dans cet ordre. La première condition qui échoue interrompt
la création :

| # | Vérification | Statut | Message |
|---|---|---|---|
| 1 | `quotation` existe et a été créé en **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La cotation appartient à votre compte | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Compte verrouillé en DDP : une cotation DDU doit être payée | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | Documents fiscaux de l'expéditeur et du destinataire | `400` | Voir [Documents fiscaux obligatoires](#documents-fiscaux-obligatoires) |
| 5 | Cotation pas encore utilisée | `406` | `Cotação já foi utilizada em outra remessa!` |
| 6 | N'est pas une `simple` de `package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 7 | Cotation dans son délai de validité | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 8 | Le contrat figure dans la liste de la cotation | `404` | `Contrato especificado não foi listado para esta cotação!` |
| 9 | Type de paiement activé pour le compte | `401` | `Opção de pagamento não autorizado!` |
| 10 | NF-e indiquée lorsque le contrat l'exige | `400` | `Numero da nota fiscal é requerido` |

> Pour savoir, **avant** d'expédier, si une cotation a déjà été utilisée, consultez
> `GET /v1/quotation/{quotation}` et regardez `data.finished`. Détails des scénarios éligibles dans
> [Cotation (v1)](api-cotacao-v1.md#cotations-non-éligibles-à-lexpédition).

## Règles métier

### Cohérence entre la cotation et l'expédition
- Avec une adresse manuelle (Forme B), la localité est toujours celle de la cotation.
- Avec `address_code` (Forme A), l'adresse enregistrée doit avoir **les mêmes pays, état et ville** que la cotation.
  Les divergences retournent `406` :
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documents fiscaux obligatoires
| Pays de l'adresse | `type` | `foreign` | `federal_tax_id` | `state_tax_id` |
|---|---|---|---|---|
| `BR` | `pf` | `false` (par défaut) | **Obligatoire** — CPF valide | **Obligatoire** |
| `BR` | `pj` | `false` (par défaut) | **Obligatoire** — CNPJ valide | **Obligatoire** |
| `BR` | quelconque | `true` | **Obligatoire** (document étranger, sans validation de CPF/CNPJ) | Optionnel |
| `MX`, `GR` | quelconque | - | **Obligatoire** | Optionnel |
| Autres | quelconque | - | Optionnel | Optionnel |

- Pour `type: pf` au Brésil, l'inscription d'État ne peut pas être `ISENTO`/`ISENTA`.
- Pour `type: pj` dans n'importe quel pays, un `federal_tax_id` au format CNPJ est validé comme un CNPJ.
- Avec `address_code`, les documents fiscaux sont lus dans l'enregistrement de l'adresse et sont soumis aux mêmes règles.

### Restrictions de transporteur
- Pour les contrats DHL avec un expéditeur **personne morale au Brésil** et un objet autre qu'un document, la `nfkey` est
  obligatoire. Son absence retourne `400` avec `Numero da nota fiscal é requerido`.

### Préréglages appliqués à l'expédition
| Préréglage | Effet |
|---|---|
| Centre de coûts | Renseigne `cost_center_code` lorsqu'il n'est pas envoyé, ou qu'il est envoyé vide/nul |
| Finalité de l'envoi | Renseigne `type` lorsqu'il n'est pas envoyé |
| Facturant | Renseigne `biller` lorsqu'il n'est pas envoyé (ou qu'il est envoyé vide) et que le compte affiche un facturant |
| Notification de facturation | Renseigne `payment_notify_shipping_billing` lorsqu'il n'est pas envoyé |

### Annulation automatique
Les expéditions qui restent **29 jours** sans numéro de suivi, dans un statut antérieur à la prise en charge, peuvent être annulées
automatiquement par la plateforme. Détails dans [Suivi de l'Expédition](api-rastreio.md).

## Erreurs courantes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Document fiscal, NF-e obligatoire, contrat par `slug` invalide, annulation non autorisée | Lire `message` et corriger le scénario |
| `401` | Paiement non autorisé | Activer le type de paiement sur le compte |
| `403` | Cotation simple/expirée, ou sans permission pour l'action | Refaire la cotation / demander la permission au support |
| `404` | Cotation d'un autre compte, expédition/contrat/adresse introuvable | Vérifier l'identifiant et le `carrier_code`/`slug` |
| `406` | Cotation déjà utilisée ou `address_code` incompatible avec la cotation | Faire une nouvelle cotation / utiliser l'adresse cotée |
| `422` | Validation des champs, cotation d'une autre version, compte DDP uniquement, pièce jointe invalide | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Conservez toujours le `quotation` : c'est l'identifiant le plus stable de l'envoi.
- Indiquer `channel_identifier_number` dans la cotation est la manière la plus pratique de consulter et d'annuler
  en utilisant le numéro de votre propre commande.
- Pour indiquer la ville, l'état et le code postal **dans l'expédition** (et non dans la cotation), utilisez
  l'[Expédition V2](api-remessa-v2.md). Différences dans [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md).
- Pour cotation + expédition + documents en **un seul appel synchrone**, voir [Manifeste](api-manifesto.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
