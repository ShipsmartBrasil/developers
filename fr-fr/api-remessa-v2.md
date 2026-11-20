# Expédition (v2)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md) et [Cotation (v2)](api-cotacao-v2.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Transforme une **cotation v2** en un envoi réel.
- La différence principale par rapport à l'[Expédition v1](api-remessa-v1.md) : ici vous indiquez l'**adresse complète**
  (ville, état et code postal), parce que la cotation v2 n'a reçu que le pays (et éventuellement l'état).
- L'adresse complète est **validée lors de l'expédition** (CEP au Brésil, adresse internationale dans les autres pays).
- La v2 n'expose **que la création**. Statut, documents, annulation, mise à jour et planification d'enlèvement
  utilisent les endpoints `/v1/...`, qui fonctionnent normalement pour les expéditions créées en v2.

> ⚠️ **L'expédition est une opération irréversible sur la route normale.** Une fois créée, elle entre en traitement
> chez le transporteur.

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
1. POST /v2/quotation                  → quotation + carriers[]   (pays/état seuls)
2. POST /v2/shipping                   → expédition reçue         (adresse complète)
3. GET  /v1/shipping/{shipping}/status → suivre jusqu'à "shipped"
4. GET  /v1/shipping/{shipping}/docs   → télécharger AWB, invoice, etc.
5. POST /v1/schedule                   → planifier l'enlèvement (optionnel)
```

1. [Créez la cotation v2](api-cotacao-v2.md) et vérifiez qu'elle est [éligible](#cotations-non-éligibles-à-lexpédition).
2. Envoyez l'expédition avec le `quotation`, le transporteur choisi et l'adresse complète.
3. La création est **asynchrone** : `200` signifie « nous avons reçu », pas « nous avons expédié ».
4. Suivez via le [suivi](api-rastreio.md) et téléchargez les [documents](api-documentos.md).

## Concepts essentiels

### Expédition à partir d'une cotation v2
- Le **pays** d'origine et de destination vient toujours de la cotation (il n'est pas indiqué dans l'expédition).
- L'**état** indiqué dans la cotation (`state_code`) est réutilisé ; si la cotation n'avait pas d'état, il est
  déduit du code postal/de l'adresse envoyés dans l'expédition.
- La **ville et le code postal** sont indiqués ici et s'appliquent désormais à l'expédition.
- Une cotation génère **une seule** expédition.

### Carte des endpoints par version

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

> Il n'existe ni `GET /v2/shipping/...` ni `DELETE /v2/shipping/...`. Utilisez les routes `/v1`, documentées dans
> [Expédition (v1)](api-remessa-v1.md).

## Endpoint : créer une expédition

- **Méthode et route :** `POST /v2/shipping`
- **Quand l'utiliser :** après avoir obtenu une cotation **v2** éligible, dans son délai de validité.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `quotation` | string | **oui** | - | - | - | UUID de la cotation. Elle doit avoir été créée en **v2** |
| `carrier_code` | integer | **oui** si `slug` n'est pas envoyé | - | - | - | Code du contrat (`carriers[].code` de la cotation) |
| `slug` | string | **oui** si `carrier_code` n'est pas envoyé | - | 20 | - | Identifiant textuel du contrat (`carriers[].slug` de la cotation). Lorsqu'il est envoyé, il prévaut sur `carrier_code` |
| `address_sender` | object | **oui** | - | - | - | Expéditeur (voir tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | Destinataire (voir tableau ci-dessous) |
| `payment` | object | **oui** | - | - | - | Mode de paiement |
| `type` | string | non | - | - | préréglage du compte ou `Commercial` | Finalité de l'envoi. Valeurs : `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | conditionnel | 44 | 44 | `null` | Clé de la NF-e. **Exactement 44 caractères**. Obligatoire pour certains contrats |
| `invoice_number` | string | non | 1 | 50 | code interne | Numéro de la facture/invoice |
| `shipment_reference` | string | non | 1 | 50 | ID interne | Référence de l'envoi |
| `cost_center_code` | integer | non | - | - | préréglage du compte | Code du centre de coûts autorisé pour votre utilisateur. Vide/nul reçoit la valeur par défaut du compte |
| `payment_notify_shipping_billing` | boolean | non | - | - | préréglage du compte | Avec `payment.type: afterward`, envoie l'e-mail de facturation |
| `attachments` | array | non | - | - | `null` | Documents joints à l'envoi |
| `biller` | object | non | - | - | préréglage du compte | Facturant affiché sur l'invoice à la place de l'expéditeur |

### Paramètres — `address_sender` et `address_receiver`

**C'est la principale différence de la v2 :** la ville et le code postal (ou l'état) sont obligatoires ici.

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code de l'adresse enregistrée. Lorsqu'il est utilisé, n'envoyez pas les autres champs |

> ⚠️ En v2, utilisez `address_code` dans l'expédition **uniquement si le même `address_code` a été utilisé dans la cotation v2**.
> Une cotation faite uniquement avec le pays n'a pas de ville, et l'adresse enregistrée sera refusée avec
> `406 A cidade :city é diferente do enviado na cotação!`.

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `city` | string | **oui** | 3 | 100 | - | Ville |
| `postal_code` | string | conditionnel | - | 12 | - | Code postal/ZIP. Obligatoire si `state` n'est pas indiqué. **Toujours recommandé ; au Brésil, c'est ce qui permet de valider l'adresse** |
| `state` | string | conditionnel | - | - | - | Nom de l'état/province. Obligatoire si `postal_code` n'est pas indiqué |
| `state_code` | string | non | - | - | état de la cotation | Sigle de l'état/province. Lorsque la cotation avait un `state_code`, c'est lui qui est utilisé à la place de celui-ci |
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
| `federal_tax_id` | string | conditionnel | - | 20 | `null` | CPF/CNPJ ou document fédéral. Obligatoire selon le pays (voir [Documents fiscaux](api-remessa-v1.md#documents-fiscaux-obligatoires)) |
| `state_tax_id` | string | conditionnel | - | 20 | `null` | Inscription d'État. Obligatoire pour les Brésiliens (`BR` avec `foreign: false`) |
| `foreign` | boolean | non | - | - | `false` si le pays est `BR` ; `true` dans les autres cas | Indique si la personne est étrangère |

> **Règle pratique :** indiquez `city` + `postal_code` + `street` + `name` + `email` + `phone` + `type`
> (+ documents fiscaux lorsqu'ils sont exigés). Le pays vient de la cotation.
> Sans `postal_code`, le champ `state` (nom de l'état) devient obligatoire — `state_code` seul ne suffit pas.

### Paramètres — `payment`

| Champ | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|
| `payment.type` | string | **oui** | `billed`, `gateway`, `afterward` | Modèle financier de l'envoi |

| Valeur | Signification | Exigence du compte |
|---|---|---|
| `billed` | Facturé : le montant est ajouté à votre facture périodique | Option de paiement « facturé » activée |
| `gateway` | Paiement immédiat via passerelle, dans la devise définie dans `currency_payment` | Option de paiement « immédiat » activée |
| `afterward` | Paiement ultérieur, réglé en dehors du flux automatique | - |

### Paramètres — `attachments[]`

| Champ | Type | Obligatoire | Max. | Description |
|---|---|---|---|---|
| `description` | string | **oui** | 100 | Nom/description du document (utilisé ensuite pour localiser ou supprimer la pièce jointe) |
| `base64` | string | **oui** | - | Contenu du fichier encodé en Base64 |

| Limite | Valeur |
|---|---|
| Formats acceptés | `PDF`, `XML`, `JPEG`, `PNG` |
| Taille totale de la requête | **90 Mo** |

> Comme en v1, les pièces jointes dans un format non reconnu sont **écartées silencieusement** lors de la création. Pour joindre
> des fichiers avec validation, utilisez `PUT /v1/shipping/{shipping}` (limite de 5 Mo par fichier).

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
| `state_tax_id` | string | - | 20 | Inscription d'État |
| `foreign` | boolean | - | - | Indique s'il s'agit d'un étranger |

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
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
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
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rue Principale", "name": "Boutique Exemple LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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
| `message` | string | Confirmation de réception. L'identifiant de l'expédition est le `quotation` lui-même |

## Différences par rapport à la v1

| Thème | v1 | v2 |
|---|---|---|
| Adresse dans l'expédition | Rue, nom, e‑mail, téléphone, type (+ documents fiscaux) | Les mêmes champs **+ `city`, `postal_code` (ou `state`)** |
| `city` | Ignoré (vient de la cotation) | **Obligatoire**, min. 3 et max. 100 caractères |
| Validation de l'adresse | Faite lors de la cotation | **Faite lors de l'expédition** |
| `address_code` | Doit correspondre à l'adresse de la cotation | Ne fonctionne que si la cotation v2 a aussi utilisé `address_code` |
| `biller` | Champs de localité acceptés | Champs de localité **validés** (`city` min. 3) |
| Endpoints du cycle | Création, statut, documents, mise à jour, pièces jointes, annulation | **Création uniquement.** Le reste utilise `/v1` |
| Origine de la cotation | `POST /v1/quotation` | `POST /v2/quotation` |

## Cotations non éligibles à l'expédition

Les mêmes vérifications que la v1 s'appliquent, dans le même ordre — voir le tableau complet dans
[Expédition (v1)](api-remessa-v1.md#cotations-non-éligibles-à-lexpédition). La seule différence est la version :

| Vérification | Statut | Message |
|---|---|---|
| `quotation` existe et a été créé en **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Cotation simple de colis/palette | `403` | `Cotação Simples é inelegível para remessa!` |
| Cotation déjà utilisée | `406` | `Cotação já foi utilizada em outra remessa!` |
| Cotation expirée | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| Contrat absent de la liste de la cotation | `404` | `Contrato especificado não foi listado para esta cotação!` |

## Règles métier

### Validation de l'adresse complète
Après les vérifications de la cotation, l'adresse manuelle (Forme B) est validée :

| Cas | Statut | Message |
|---|---|---|
| Brésil, sans état dans la cotation : CEP inexistant ou d'un autre état | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Hors du Brésil : adresse introuvable | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Hors du Brésil : état/province non reconnu | `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |

> Envoyez toujours `postal_code` et, si vous le connaissez, `state_code` (de préférence dès la cotation) : c'est ce qui donne le meilleur
> taux de réussite lors de la validation.

### Cohérence entre la cotation et l'expédition
- Le pays est toujours celui de la cotation.
- Le `state_code` de la cotation, lorsqu'il existe, prévaut sur celui envoyé dans l'expédition.
- Avec `address_code`, l'adresse enregistrée doit avoir les mêmes pays, état et ville que ceux enregistrés dans la cotation :
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documents fiscaux, transporteur et préréglages
Identiques à la v1 :
[Documents fiscaux obligatoires](api-remessa-v1.md#documents-fiscaux-obligatoires),
[Restrictions de transporteur](api-remessa-v1.md#restrictions-de-transporteur) et
[Préréglages appliqués à l'expédition](api-remessa-v1.md#préréglages-appliqués-à-lexpédition).

## Erreurs courantes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Document fiscal, CEP brésilien invalide, NF-e obligatoire | Lire `message` et corriger le scénario |
| `401` | Paiement non autorisé | Activer le type de paiement sur le compte |
| `403` | Cotation simple/expirée, ou sans permission | Refaire la cotation / demander la permission au support |
| `404` | Contrat absent de la cotation, adresse internationale introuvable | Vérifier `carrier_code`/`slug` et l'adresse |
| `406` | Cotation déjà utilisée ou `address_code` incompatible | Faire une nouvelle cotation / utiliser une adresse manuelle |
| `422` | Validation des champs, cotation d'une autre version, compte DDP uniquement | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue ou état/province non reconnu | Vérifier l'adresse ; réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Utilisez la v2 lorsque l'adresse complète n'existe qu'au moment de l'envoi.
- Tout le cycle opérationnel après la création passe par les endpoints `/v1` — ils ne font pas de distinction selon la version
  dans laquelle l'expédition a été créée.
- Pour le détail de chaque opération après la création, voir [Expédition (v1)](api-remessa-v1.md),
  [Suivi](api-rastreio.md), [Documents](api-documentos.md) et [Planification](api-agendamento.md).
- Différences entre les versions dans [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
