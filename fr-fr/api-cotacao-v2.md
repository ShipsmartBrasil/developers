# Cotation (v2)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Même moteur de calcul que la [Cotation v1](api-cotacao-v1.md), avec un **contrat d'adresse beaucoup plus allégé**.
- Utilisez la v2 quand, au moment de la cotation, vous n'avez **pas encore** la ville et le code postal du destinataire — seulement le pays
  (et éventuellement l'état).
- La ville, le code postal et l'adresse complète sont indiqués plus tard, dans l'[Expédition V2](api-remessa-v2.md).

> **Important :** une cotation créée en v2 ne peut générer une expédition qu'en v2 (`POST /v2/shipping`).
> Voir le [comparatif](api-comparativo-cotacao-remessa.md) pour choisir la version.

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
Détails dans [Premiers Pas](api-primeiros-passos.md).

## Flux d'utilisation
1. `POST /v2/quotation` en indiquant uniquement le pays (et l'état, si vous le connaissez) à l'origine et à destination.
2. La réponse contient déjà le résultat calculé (`data.carriers`) et le code `data.quotation`.
3. Choisissez un transporteur et créez l'[Expédition V2](api-remessa-v2.md) avec l'adresse complète,
   **avant** la date `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
  (pays/état seuls)            │
                               ▼
                    POST /v2/shipping
             (adresse complète + carrier_code|slug)
```

## Différences par rapport à la v1

| Thème | v1 | v2 |
|---|---|---|
| Adresse dans la cotation | `country_code` + `city` + (`postal_code` ou `state_code`/`state`) | Uniquement `country_code` (+ `state_code` optionnel) |
| Adresse dans l'expédition | Rue, nom, e‑mail, téléphone, type | Rue, nom, e‑mail, téléphone, type **+ ville, état et code postal** |
| Consultation du résultat | `GET /v1/quotation/{quotation}/{level}` | **N'existe pas.** Le résultat arrive dans le `POST` lui-même |
| `ior` (Importer of Record) | Pris en charge | **Ne fait pas partie du contrat de la v2** |
| `items[].selling_unit_value` | Pris en charge | **Ne fait pas partie du contrat de la v2** |
| Précision du calcul | Supérieure : ville/code postal permettent d'appliquer les taxes régionales dès la cotation | Le montant peut être affiné dans l'expédition, quand l'adresse complète est indiquée |

> Tout le reste (objet, type, boîtes, articles, kits, devises, fret déclaré, fabricant, avertissements)
> est identique à la v1.

## Concepts essentiels

### `object` — ce qui est envoyé
| Valeur | Signification | Remarques |
|---|---|---|
| `doc` | Document | Cotation par boîte, en indiquant `weight` et `price` |
| `package` | Colis/boîte | Flux le plus courant |
| `pallet` | Palette | Permet d'indiquer `volume` au lieu de hauteur/largeur/profondeur |
| `not_doc` | Obsolète | Converti en interne en `package`. **Ne l'utilisez pas dans de nouvelles intégrations** |

### `type` — comment le calcul est construit
| Valeur | Ce que vous envoyez | Quand l'utiliser |
|---|---|---|
| `simple` | Uniquement `boxes[]` avec `weight` et `price` | Simulation rapide. **Non éligible à l'expédition quand `object` vaut `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` avec `box_id` | Flux standard pour l'expédition |
| `items` | Uniquement `items[]` avec `sku` | Quand les articles et les boîtes sont déjà enregistrés et que la plateforme fait l'agencement |

### `measurement` — système de mesure
| Valeur | Dimensions | Poids | Volume |
|---|---|---|---|
| `metric` (par défaut) | centimètres | kilogrammes | m³ |
| `imperial` | pouces | livres | ft³ |

### `level` — niveau de détail du résultat
| Valeur | Retour |
|---|---|
| `simple` (par défaut) | Liste allégée des transporteurs avec les totaux |
| `detailed` | Inclut la composition détaillée (fret, assurance, taxe, avant et après les frais administratifs) |

Envoyé en **query string** : `POST /v2/quotation?level=detailed`.

## Endpoint : créer une cotation

- **Méthode et route :** `POST /v2/quotation`
- **Query string optionnelle :** `level=simple|detailed`

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Valeurs acceptées | Description |
|---|---|---|---|---|---|---|---|
| `object` | string | **oui** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Ce qui est envoyé |
| `type` | string | **oui** | - | - | - | `simple`, `advanced`, `items` | Comment le calcul est construit |
| `tax` | string | **oui** | - | - | - | `sender`, `receiver` | Qui paie les taxes |
| `insurance` | boolean | **oui** | - | - | - | `true`, `false` | Souscrire une assurance |
| `currency_quote` | string | **oui** | 3 | 3 | - | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise dans laquelle la cotation est présentée |
| `currency_payment` | string | **oui** | 3 | 3 | - | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise dans laquelle le paiement sera effectué |
| `address_sender` | object | **oui** | - | - | - | - | Origine (voir tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | - | Destination (voir tableau ci-dessous) |
| `boxes` | array | conditionnel | - | - | - | - | Obligatoire quand `type` n'est **pas** `items` |
| `items` | array | conditionnel | - | - | - | - | Obligatoire quand `type` n'est **pas** `simple` et qu'aucune boîte n'utilise `kit` |
| `measurement` | string | non | - | - | `metric` | `metric`, `imperial` | Système de mesure |
| `residential_delivery` | boolean | non | - | - | `false` | `true`, `false` | Livraison à une adresse résidentielle |
| `non_stackable` | boolean | non | - | - | `false` | `true`, `false` | Colis non empilables |
| `description` | string | non | - | 155 | `null` | - | Description libre de la cotation |
| `channel_code` | integer | non | - | - | préréglage du compte | - | Code du canal de vente enregistré |
| `channel_identifier_number` | string | non | - | - | `null` | - | Votre identifiant commercial (n° de commande). **Recommandé** |
| `channel_order_number` | string | non | - | 100 | `null` | - | Numéro de la commande dans le canal de vente |
| `declared_freight` | object | non | - | - | préréglage du compte | - | Fret déclaré (voir tableau ci-dessous) |
| `manufacturer` | object | non | - | - | `null` | - | Fabricant affiché sur la Commercial Invoice (voir tableau ci-dessous) |
| `level` | string | non (query) | - | - | `simple` | `simple`, `detailed` | Niveau de détail du résultat |

### Paramètres — `address_sender` et `address_receiver`

**C'est la principale différence de la v2.** Choisissez l'une des deux formes :

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code d'une adresse enregistrée sur votre compte |

**Forme B — uniquement le pays (et l'état, si vous le connaissez) :**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **oui** | 2 | 2 | Pays en ISO 3166-1 alpha-2 (ex. : `BR`, `US`) |
| `state_code` | string | non | - | - | Sigle de l'état/province (ex. : `SP`, `FL`). Lorsqu'il est indiqué, il améliore la précision du calcul des taxes régionales |

> N'envoyez ni `city`, ni `postal_code`, ni `state` dans la cotation v2 — ces données sont indiquées dans l'expédition.

### Paramètres — `boxes[]`

Chaque élément du tableau représente un colis. Il y a **trois façons** de décrire la boîte :

**Forme A — kit enregistré** (remplit la boîte **et** les articles automatiquement) :

| Champ | Type | Obligatoire | Max. | Description |
|---|---|---|---|---|
| `kit` | string | **oui** dans cette forme | 50 | SKU du kit enregistré. Les autres champs de la boîte et les articles de cette position sont remplis à partir de l'enregistrement du kit |

**Forme B — boîte enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `box_code` | integer | **oui** dans cette forme | Code de la boîte enregistrée ([Boîte](api-caixa-v1.md)) |
| `box_identifier` | string | alternative à `box_code` | Identifiant de la boîte enregistrée (ex. : `CX0236`) |

**Forme C — boîte indiquée manuellement :**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `name` | string | non | 4 | 60 | Nom/libellé de la boîte |
| `height` | number | **oui** | 1 | - | Hauteur |
| `width` | number | **oui** | 1 | - | Largeur |
| `depth` | number | **oui** | 1 | - | Profondeur |
| `volume` | number | conditionnel | 0.001 | - | **Uniquement quand `object` vaut `pallet`.** Alternative à hauteur/largeur/profondeur |

**Champs valides dans n'importe quelle forme (sauf `kit`) :**

| Champ | Type | Obligatoire | Min. | Description |
|---|---|---|---|---|
| `weight` | number | **oui** quand `object` vaut `doc`/`package` **et** `type` vaut `simple` | > 0 | Poids du colis |
| `price` | number | **oui** quand `object` vaut `doc`/`package` **et** `type` vaut `simple` | > 0 | Valeur déclarée du colis |

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **oui** | 1 | 1000 | Quantité de l'article |
| `unit_value` | number | **oui** | - | - | Valeur unitaire dans la devise de la cotation |
| `box_id` | integer | **oui**, sauf quand `type` vaut `items` | - | - | **Position** de la boîte dans le tableau `boxes` (1 = première boîte). Ce n'est pas le `box_code` |
| `sku` | string | **oui** quand `type` vaut `items` | - | 50 | SKU de l'article enregistré ([Articles](api-itens-v1.md)). Lorsqu'il est indiqué, les données de l'enregistrement sont utilisées |
| `ean` | string | non | - | 50 | Code EAN de l'article |
| `name` | string | **oui** quand `sku` n'est pas indiqué | 6 | 50 | Nom de l'article (apparaît sur l'invoice) |
| `description` | string | non | - | 155 | Description détaillée de l'article |
| `country_code` | string | **oui** quand `sku` n'est pas indiqué | 2 | 2 | Pays d'origine/de fabrication de l'article |
| `hscode` | string | **oui** quand `sku` n'est pas indiqué | 6 | 10 | NCM/HS Code. Définit la table de taxes appliquée |
| `weight` | number | **oui** quand `sku` n'est pas indiqué | 0.001 | - | Poids unitaire de l'article |
| `is_groupable` | boolean | non | - | - | Par défaut `true`. Quand `false`, l'article reçoit une boîte exclusive dans le regroupement `smartbox` |

### Paramètres — `declared_freight`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | non | 0 | - | Montant du fret déclaré |
| `currency` | string | **oui** quand `value` est indiqué | 3 | 3 | Devise du fret déclaré |

### Paramètres — `manufacturer`

Données du fabricant imprimées sur la Commercial Invoice. Envoyez `address_code` **ou** les champs manuels.

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `address_code` | integer | alternative aux champs manuels | - | - | Code d'adresse enregistrée |
| `name` | string | **oui** sans `address_code` | 5 | 30 | Raison sociale/nom du fabricant |
| `country_code` | string | **oui** sans `address_code` | 2 | 2 | Pays |
| `state_code` | string | **oui** sans `address_code` | 1 | 10 | Sigle de l'état |
| `state` | string | non | - | 155 | Nom de l'état |
| `postal_code` | string | **oui** sans `address_code` | - | 12 | Code postal/ZIP |
| `city` | string | **oui** sans `address_code` | - | 100 | Ville |
| `street` | string | **oui** sans `address_code` | - | 70 | Rue |
| `number` | string | non | - | 6 | Numéro |
| `district` | string | non | - | 30 | Quartier |
| `complement` | string | non | - | 20 | Complément |
| `email` | string | **oui** sans `address_code` | - | 60 | E-mail |
| `phone` | string | **oui** sans `address_code` | - | 25 | Téléphone |
| `phone_extension` | string | non | - | 6 | Poste |
| `type` | string | **oui** sans `address_code` | - | - | `pf` ou `pj` |
| `federal_tax_id` | string | non | - | 20 | CPF/CNPJ ou équivalent |
| `state_tax_id` | string | non | - | 20 | Numéro d'immatriculation de l'état |

### Payload minimal

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.5,
      "price": 100
    }
  ]
}
```

### Payload minimal éligible à l'expédition

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "T-shirt coton",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ]
}
```

### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "description": "Commande du marketplace, lot d'octobre",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
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
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton, manches courtes, couleur blanche",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    }
  ],
  "manufacturer": {
    "name": "Usine Exemple LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Hangar 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

### Exemple de requête

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v2/quotation?level=detailed' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "USD",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt coton", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}]
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2026-02-21 14:28:10",
    "description": null,
    "channel": "Marketplace Exemple",
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_quote_exchange_value": "1",
    "currency_payment": "EUR",
    "currency_payment_exchange_value": "0.92",
    "warnings": [
      "Itens com descrições iguais podem causar problemas na fiscalização aduaneira"
    ],
    "carriers": [
      {
        "code": 1,
        "slug": "dhl-br",
        "name": "DHL",
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "insurance": "11.55",
        "insurance_final": "11.55",
        "tax": "24924.09",
        "tax_final": "24924.09",
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.quotation` | string (UUID) | **Code de la cotation.** À utiliser dans `POST /v2/shipping` |
| `data.finished` | boolean | Indique si le traitement de la cotation est terminé |
| `data.warnings` | array | **Optionnel.** N'apparaît que lorsqu'il y a des alertes douanières |
| `carriers[].code` | integer | Code du contrat. À utiliser dans `carrier_code` de l'expédition |
| `carriers[].slug` | string | Identifiant textuel du contrat. Alternative à `carrier_code` |
| `carriers[].valid_until` | string | **Validité de la cotation.** Créez l'expédition avant cette date/heure |
| `carriers[].transit_days` | integer | Délai de transit estimé en jours |
| `carriers[].weight_details.type` | string | `real` ou `cubed` — quel poids a été utilisé pour la tarification |
| `carriers[].freight` / `freight_final` | string | Fret avant / après les frais administratifs |
| `carriers[].insurance` / `insurance_final` | string | Assurance avant / après les frais administratifs |
| `carriers[].tax` / `tax_final` | string | Taxes avant / après les frais administratifs |
| `carriers[].currency_payment_amount` | string | Total à payer dans la devise de paiement |

## Règles métier

### Éligibilité à l'expédition
1. Le `quotation` existe et appartient à votre compte.
2. La cotation **n'est pas expirée** (respectez `valid_until`).
3. La cotation **n'a pas encore été utilisée** dans une autre expédition.
4. La cotation a été créée en **v2** et sera utilisée dans `POST /v2/shipping`.
5. Ce n'est pas une cotation `simple` de `package`/`pallet`.
6. Le transporteur indiqué figure dans la liste retournée par la cotation.

### Restrictions de charge
Les mêmes restrictions que la v1 s'appliquent (dimensions, poids par colis, poids total, limite d'articles,
disponibilité de contrat, route bloquée, HS Code sans table de taxes).
Consultez la section **Restrictions de charge** dans [Cotation (v1)](api-cotacao-v1.md#restrictions-de-charge).

### Pas d'endpoint de consultation
La v2 **ne possède pas** `GET /v2/quotation/{quotation}`. Le résultat complet est livré dans la réponse du `POST`,
donc **stockez la réponse** si vous devez la réafficher à l'utilisateur.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier (restriction de charge, route, contrat, HS Code) | Lire `message` et ajuster le scénario |
| `403` | Sans permission de cotation | Demander la permission au support |
| `406` | Structure du payload incompatible | Revoir les types et l'imbrication |
| `422` | Validation des champs | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- La v2 échange de la précision contre de la simplicité : l'adresse complète n'arrive que dans l'expédition.
- Les flux opérationnels de **statut, documents, annulation et planification d'enlèvement** utilisent les mêmes
  endpoints `/v1/...`, même pour les expéditions créées en v2 — voir
  [Suivi](api-rastreio.md), [Documents](api-documentos.md) et [Planification](api-agendamento.md).
- Conservez la réponse du `POST` : il n'y a pas d'endpoint de reconsultation en v2.
