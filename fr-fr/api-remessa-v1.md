# Expédition (v1)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md) et [Cotation (v1)](api-cotacao-v1.md)

## Vue d'ensemble
- Transforme une cotation v1 en un envoi réel : génère l'AWB, l'invoice, la déclaration et les autres documents.
- C'est le module le plus complet du cycle : création, consultation du statut, consultation des documents, mise à jour,
  suppression de pièces jointes et annulation.
- Les endpoints de statut, documents et annulation servent **aussi** aux expéditions créées avec
  l'[Expédition V2](api-remessa-v2.md).

> ⚠️ **L'expédition est une opération irréversible sur la route normale.** Une fois créée, elle entre en traitement
> chez le transporteur. Les seules modifications ultérieures admises sont exceptionnelles : rattacher la NF-e quand elle
> n'a pas été indiquée, joindre des documents qui n'ont pas été envoyés à la création et indiquer le centre de coûts.

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
1. POST /v1/quotation                 → quotation + carriers[]
2. POST /v1/shipping                  → expédition reçue (asynchrone)
3. GET  /v1/shipping/{shipping}/status→ suivre jusqu'à "shipped"
4. GET  /v1/shipping/{shipping}/docs  → télécharger AWB, invoice, etc.
5. POST /v1/schedule                  → planifier l'enlèvement (optionnel)
```

1. [Créez la cotation](api-cotacao-v1.md) et vérifiez qu'elle est **éligible** à l'expédition.
2. Envoyez l'expédition avec le `quotation` et le transporteur choisi (`carrier_code` **ou** `slug`).
3. La création est **asynchrone** : la réponse `200` signifie « nous avons reçu », pas « nous avons expédié ».
4. Consultez le [statut](api-rastreio.md) jusqu'à ce qu'elle apparaisse dans `shipped`.
5. Téléchargez les [documents](api-documentos.md).
6. Si nécessaire, [planifiez l'enlèvement](api-agendamento.md).

## L'identifiant `{shipping}`

Dans tous les endpoints de consultation/modification, le paramètre de path `{shipping}` accepte **trois formats** :

| Format | Exemple | Remarque |
|---|---|---|
| UUID de la cotation | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forme recommandée |
| Numéro de l'AWB | `271249639538` | Fonctionne pour les expéditions créées dans les **90 derniers jours** |
| Clé de la NF-e | `12345678901234567890123456789012345678901234` | Exactement 44 chiffres numériques |

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
| `slug` | string | **oui** si `carrier_code` n'est pas envoyé | - | 20 | - | Identifiant textuel du contrat (`carriers[].slug` de la cotation) |
| `address_sender` | object | **oui** | - | - | - | Expéditeur (voir tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | Destinataire (voir tableau ci-dessous) |
| `payment` | object | **oui** | - | - | - | Mode de paiement (voir tableau ci-dessous) |
| `type` | string | non | - | 30 | `Commercial` | Finalité de l'envoi. Valeurs : `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | non | 44 | 44 | `null` | Clé de la NF-e. Lorsqu'elle est indiquée, elle doit avoir **exactement 44 chiffres** |
| `invoice_number` | string | non | 1 | 50 | code interne | Numéro de la facture/invoice. S'il est omis, le code interne de l'expédition est utilisé |
| `shipment_reference` | string | non | 1 | 50 | ID interne | Référence de l'envoi. Si elle est omise, l'ID interne de l'expédition est utilisé |
| `cost_center_code` | integer | non | - | - | `null` | Code du centre de coûts. Il doit exister et être autorisé pour votre utilisateur |
| `payment_notify_shipping_billing` | boolean | non | - | - | `false` | Notifier le service de facturation de la facturation de l'expédition |
| `attachments` | array | non | - | - | `null` | Documents joints à l'envoi (voir tableau ci-dessous) |
| `biller` | object | non | - | - | préréglage du compte | Facturant affiché sur l'invoice à la place de l'expéditeur (voir tableau ci-dessous) |

> **`carrier_code` ou `slug` :** envoyez **l'un** des deux. Le contrat indiqué doit figurer dans la liste
> `carriers[]` retournée par la cotation ; sinon l'API répond `404` avec
> `Contrato especificado não foi listado para esta cotação!`.

### Paramètres — `address_sender` et `address_receiver`

Deux formes, ne les mélangez pas :

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code de l'adresse enregistrée. **Ce doit être la même adresse que celle utilisée dans la cotation** |

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
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
| `foreign` | boolean | conditionnel | - | - | `false` | Indique si la personne est étrangère. **Obligatoire quand le pays est `BR` et que `address_code` n'est pas utilisé** — c'est ce qui permet de valider correctement le `federal_tax_id` |

> Le pays, l'état, la ville et le code postal ne sont **pas** indiqués ici : ils viennent de la cotation.
> Si vous devez les indiquer dans l'expédition, utilisez l'[Expédition V2](api-remessa-v2.md).

### Paramètres — `payment`

| Champ | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|
| `payment.type` | string | **oui** | `billed`, `gateway`, `afterward` | Modèle financier de l'envoi |

| Valeur | Signification |
|---|---|
| `billed` | Facturé : le montant entre dans votre facture périodique |
| `gateway` | Paiement immédiat via gateway, dans la devise définie par `currency_payment` |
| `afterward` | Paiement ultérieur, réglé en dehors du flux automatique |

> Le type doit être **activé pour votre compte**. Sinon :
> `400 Tipo de pagamento inexistente!` ou `401 Opção de pagamento não autorizado!`.

### Paramètres — `attachments[]`

| Champ | Type | Obligatoire | Max. | Description |
|---|---|---|---|---|
| `description` | string | **oui** | 100 | Nom/description du document (utilisé ensuite pour localiser ou supprimer la pièce jointe) |
| `base64` | string | **oui** | - | Contenu du fichier encodé en Base64 |

| Limite | Valeur |
|---|---|
| Formats acceptés | `PDF`, `XML`, `JPEG`, `PNG` |
| Taille par fichier | **5 MB** |
| Taille totale de la requête | **90 MB** |

> Fichier hors format : `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.`
> Fichier au-delà de la limite : `O arquivo enviado excede o limite máximo de 5MB.`
> XML corrompu : `O arquivo XML enviado é inválido ou está corrompido.`

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
| `state_tax_id` | string | - | 20 | Numéro d'immatriculation de l'état |
| `foreign` | boolean | - | - | Indique si la personne est étrangère |

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
    "state_tax_id": "123456789012",
    "foreign": false
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

> `federal_tax_id`, `state_tax_id` et `foreign` apparaissent dans le payload minimal parce que l'expéditeur est brésilien.
> Pour un expéditeur d'un autre pays (sauf `AR`), ces trois champs peuvent être omis.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Rue Principale", "name": "Boutique Exemple LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

> La réponse ne contient **ni** AWB **ni** documents : l'expédition a seulement été reçue. Utilisez le
> [suivi](api-rastreio.md) pour la suivre et les [documents](api-documentos.md) pour télécharger les fichiers.

### Erreurs spécifiques à la création

| Statut | Message | Cause / correction |
|---|---|---|
| `400` | `Tipo de pagamento inexistente!` | `payment.type` en dehors de `billed`/`gateway`/`afterward` ou non activé |
| `400` | `Esta cotação está expirada. Porfavor, faça outra cotação...` | `valid_until` dépassé — refaites la cotation |
| `400` | `Cotação já foi utilizada em outra remessa!` | Une cotation génère **une** expédition |
| `400` | `Cotação Simples é inelegível para remessa!` | Utilisez `type: advanced` ou `items` dans la cotation |
| `400` | `Chave da Nota não foi informada!!` | Le contrat exige `nfkey` dans ce scénario |
| `400` | `O endereço especificado, não foi encontrado!` | `address_code` invalide ou appartenant à un autre compte |
| `400` | `O país :country é diferente do enviado na cotação!` | L'adresse de l'expédition doit correspondre à celle de la cotation |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Pays `BR`/`AR` sans `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Pays `BR` sans `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ avec un chiffre de contrôle incorrect |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF avec un chiffre de contrôle incorrect |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` avec un `state_tax_id` exonéré |
| `401` | `Opção de pagamento não autorizado!` | Activez le type de paiement sur votre compte |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` en dehors de la liste de la cotation |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | Cotation v2 utilisée sur `/v1/shipping` |

## Endpoint : consulter le statut de l'expédition

- **Méthode et route :** `GET /v1/shipping/{shipping}/status`
- **Par canal :** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `shipping` | path | string | **oui** | UUID de la cotation, AWB (90 derniers jours) ou clé NF-e (44 chiffres) |
| `channel_identifier_number` | path | string | **oui** | Identifiant commercial indiqué dans la cotation |
| `channel_code` | path | integer | non | Code du canal de vente |

Détails, formats de réponse et codes de statut dans [Suivi de l'Expédition](api-rastreio.md).

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

## Endpoint : consulter les documents

- **Méthode et route :** `GET /v1/shipping/{shipping}/docs`
- **Par canal :** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | UUID, AWB ou clé NF-e |
| `attachments` | query | boolean | non | `true` | Inclure les pièces jointes que vous avez envoyées |
| `embedded` | query | boolean | non | `true` | Inclure les documents incorporés (nécessite une permission spécifique) |
| `base64` | query | boolean | non | `true` | Inclure le contenu en Base64 en plus de l'`url` |
| `documents` | query | string | non | tous | Liste séparée par des virgules des types souhaités |

Types acceptés dans `documents` : `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

Détails complets dans [Documents de l'Expédition](api-documentos.md).

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

## Endpoint : mettre à jour l'expédition

- **Méthode et route :** `PUT /v1/shipping/{shipping}`
- **Quand l'utiliser :** uniquement dans des cas exceptionnels — rattacher la NF-e, joindre des documents manquants
  ou indiquer le centre de coûts.
- **Règle :** envoyez `nfkey` et/ou `attachments` et/ou `cost_center_code`. **Au moins un** doit être présent.

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `nfkey` | string | conditionnel | 44 | 44 | Clé de la NF-e (44 chiffres numériques) |
| `cost_center_code` | integer | conditionnel | - | - | Code du centre de coûts autorisé pour votre utilisateur |
| `attachments` | array | conditionnel | - | - | Fichiers à joindre |
| `attachments[].description` | string | **oui** si `attachments` est envoyé | - | 100 | Description du document |
| `attachments[].base64` | string | **oui** si `attachments` est envoyé | - | - | Contenu en Base64. Formats : `PDF`, `XML`, `JPEG`, `PNG`. Max. **5 MB** par fichier |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

| Statut | Message | Cause |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | Aucun des trois champs n'a été envoyé |
| `404` | `Remessa não encontrada!` | `{shipping}` invalide |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | L'expédition a déjà trop avancé dans le flux |

## Endpoint : supprimer des pièces jointes

- **Méthode et route :** `DELETE /v1/shipping/{shipping}/docs`
- **Quand l'utiliser :** pour effacer une pièce jointe envoyée par erreur.

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `attachments` | array | **oui** | 1 élément | - | Liste des pièces jointes à supprimer |
| `attachments[].id` | integer | **oui** si `name` n'est pas envoyé | 1 | - | ID de la pièce jointe (retourné par `GET .../docs`) |
| `attachments[].name` | string | **oui** si `id` n'est pas envoyé | - | 100 | Description exacte de la pièce jointe |

> Envoyez **`id` OU `name`** par élément, jamais les deux.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

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

> L'opération est **partielle** : ce qui a été supprimé apparaît dans `deleted` et ce qui a échoué dans `failed`,
> avec le motif. Vérifiez toujours le tableau `failed`.

## Endpoint : annuler l'expédition

- **Méthode et route :** `DELETE /v1/shipping/{shipping}`
- **Par canal :** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Quand l'utiliser :** pour demander l'annulation. Le traitement est **asynchrone**.

| Champ | Emplacement | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | - | UUID, AWB ou clé NF-e |
| `channel_identifier_number` | path | string | **oui** (variante par canal) | - | - | Identifiant commercial |
| `channel_code` | path | integer | non | - | - | Code du canal. Seules les valeurs **numériques** sont interprétées comme un canal |
| `reason` | body | string | non | 10 | 255 | Motif de l'annulation, en texte libre |

> **Attention à l'ambiguïté de route :** `DELETE /v1/shipping/{valeur}/{numero}` est interprété comme une
> **annulation par canal**. Pour annuler par identifiant d'expédition, utilisez toujours
> `DELETE /v1/shipping/{shipping}` (un seul segment).

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Annulation demandee par le client"}'
```

```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

| Statut | Message | Cause |
|---|---|---|
| `400` | `Remessa especificada não foi encontrada ou não é possivel mais cancelar a mesma` | Identifiant invalide ou expédition dans un statut qui n'autorise pas l'annulation |
| `422` | Validation | `reason` avec moins de 10 ou plus de 255 caractères |

> Après avoir demandé l'annulation, consultez le [statut](api-rastreio.md) : il passe par `cancelling`
> et, lorsqu'il est confirmé par le transporteur, passe à `cancelled`.

## Règles métier

### Éligibilité de la cotation
| Condition | Message en cas d'échec |
|---|---|
| La cotation existe et appartient au compte | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Cotation non expirée | `Esta cotação está expirada...` |
| Cotation non utilisée auparavant | `Cotação já foi utilizada em outra remessa!` |
| Même version d'API | `Não é possivel finalizar, pois seu quotation pode estar inválido...` |
| Ce n'est pas une cotation `simple` de `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| Le contrat figure dans la liste de la cotation | `Contrato especificado não foi listado para esta cotação!` |

### Cohérence entre cotation et expédition
L'adresse indiquée dans l'expédition doit être compatible avec celle de la cotation. Les divergences retournent :
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
- Le champ `foreign` existe pour que la validation du `federal_tax_id` soit effectuée dans le bon format
  (document brésilien vs. document étranger).

### Restrictions de transporteur
- `Cotações e envios via FedEx entre Europa e Oriente Médio estão temporariamente suspensos pela transportadora.`
- Pour certains contrats (ex. : DHL avec un expéditeur personne morale au Brésil), la `nfkey` est obligatoire.
  Son absence retourne `Chave da Nota não foi informada!!`.

### Annulation automatique
Les expéditions qui restent **29 jours ou plus** sans événement de suivi dans un statut éligible peuvent être annulées
automatiquement par la plateforme.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier (cotation, adresse, paiement, document fiscal) | Lire `message` et corriger le scénario |
| `401` | Paiement non autorisé | Activer le type de paiement sur le compte |
| `403` | Sans permission pour l'action | Demander la permission au support |
| `404` | Expédition/contrat non trouvé | Vérifier l'identifiant et le `carrier_code`/`slug` |
| `422` | Validation des champs ou expédition dans un statut non modifiable | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Conservez toujours le `quotation` : c'est l'identifiant le plus stable de l'envoi.
- Indiquer `channel_identifier_number` dans la cotation est la façon la plus pratique de consulter et d'annuler
  en utilisant le numéro de votre propre commande.
- Pour indiquer la ville, l'état et le code postal **dans l'expédition** (et non dans la cotation), utilisez
  l'[Expédition V2](api-remessa-v2.md).
- Pour cotation + expédition + documents en **un seul appel synchrone**, voir [Manifeste](api-manifesto.md).
