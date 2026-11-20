# Cotation (v2)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Même calcul que la [Cotation v1](api-cotacao-v1.md), avec un **contrat d'adresse beaucoup plus concis**.
- Utilisez la v2 lorsque, au moment de la cotation, vous **n'avez pas encore** la ville et le code postal — seulement le pays
  (et éventuellement l'état).
- La ville, le code postal et l'adresse complète sont indiqués et validés plus tard, dans l'[Expédition V2](api-remessa-v2.md).
- Elle couvre les scénarios de chargement de la v1 **et** la **cotation par articles** (`type: items`), **exclusive à la v2** : vous envoyez
  uniquement le `sku` des articles enregistrés, sans indiquer de boîtes — voir [Scénarios de cotation](#scénarios-de-cotation).

> **Important :** une cotation créée en v2 ne peut générer une expédition qu'en v2 (`POST /v2/shipping`).
> Consultez le [comparatif](api-comparativo-cotacao-remessa.md) pour choisir la version.

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
Détails dans [Premiers Pas](api-primeiros-passos.md).

## Flux d'utilisation
1. Choisissez le [scénario de cotation](#scénarios-de-cotation) qui correspond à votre chargement (`object` + `type`).
2. `POST /v2/quotation` en indiquant uniquement le pays (et l'état, si vous le connaissez) à l'origine et à la destination.
3. La réponse contient déjà le résultat calculé (`data.carriers`) et le code `data.quotation`. **Conservez la réponse** :
   il n'existe pas d'endpoint de nouvelle consultation en v2.
4. Vérifiez que la cotation est [éligible à l'expédition](#cotations-non-éligibles-à-lexpédition).
5. Choisissez un transporteur et créez l'[Expédition V2](api-remessa-v2.md) avec l'adresse complète,
   **avant** le `valid_until`.

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
| Cotation par articles (`type: items`) | **Non disponible** : les boîtes sont toujours indiquées | **Disponible** : uniquement le `sku` des articles enregistrés, sans boîtes |
| Validation de l'adresse | Dans la cotation (code postal, ville, état) | Dans l'expédition, lorsque l'adresse complète est envoyée |
| Adresse dans l'expédition | Voie, nom, e‑mail, téléphone, type | Voie, nom, e‑mail, téléphone, type **+ ville, état et code postal** |
| Consultation du résultat | `GET /v1/quotation/{quotation}/{level}` | **N'existe pas.** Le résultat arrive dans le `POST` lui-même |
| `ior` (Importer of Record) | Pris en charge | **Ne fait pas partie du contrat de la v2** (ignoré) |
| `items[].selling_unit_value` | Pris en charge | **Ne fait pas partie du contrat de la v2** (ignoré) |
| Précision du calcul | Plus élevée : la ville/le code postal permettent d'appliquer les frais régionaux dès la cotation | La valeur tient compte du pays/de l'état ; l'adresse complète intervient dans l'expédition |

> Ce sont les deux raisons d'être de la v2 : **adresse réduite** (uniquement pays/état) et **cotation par articles**.
> Tout le reste (objet, boîtes, kits, devises, fret déclaré, fabricant, warnings, préréglages du compte) est identique à la v1.

## Concepts essentiels

### `object` — ce qui est envoyé
| Valeur | Signification | Remarques |
|---|---|---|
| `doc` | Document | Coté par colis, en indiquant `weight` et `price` de chaque enveloppe/boîte |
| `package` | Colis/boîte | Flux le plus courant |
| `pallet` | Palette | Accepte des dimensions (LLH) **ou** un `volume` en m³/ft³ |
| `not_doc` | Hérité | Converti en interne en `package`. **Ne l'utilisez pas dans les nouvelles intégrations** |

### `type` — comment le calcul est construit
| Valeur | Ce que vous envoyez | Quand l'utiliser |
|---|---|---|
| `simple` | Uniquement `boxes[]` avec `weight` et `price` | Documents et simulations rapides. **N'est pas éligible à l'expédition lorsque `object` vaut `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` avec `box_id` | Flux standard pour l'expédition |
| `items` | `items[]` avec `sku` (et, selon le regroupement du compte, des `boxes[]` candidates) | **Exclusif à la v2.** Lorsque les articles sont déjà enregistrés et que la plateforme constitue les boîtes |

### Matrice d'éligibilité

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Cotation et expédition | Non recommandé | Non recommandé |
| `package` | ⚠️ Simulation uniquement (non éligible à l'expédition) | ✅ Cotation et expédition | ✅ Cotation et expédition |
| `pallet` | ⚠️ Simulation uniquement (non éligible à l'expédition) | ✅ Cotation et expédition | ✅ Cotation et expédition |

### `tax` — qui paie les taxes
| Valeur | Signification |
|---|---|
| `sender` | DDP : l'expéditeur paie les taxes et droits à destination |
| `receiver` | DDU : le destinataire paie les taxes et droits à destination |

> Si votre compte dispose d'un **préréglage de taxe** configuré, celui-ci **remplace** la valeur envoyée dans `tax`.
> Voir [Préréglages appliqués à la cotation](api-cotacao-v1.md#préréglages-appliqués-à-la-cotation).

### `measurement` — système de mesure
| Valeur | Dimensions | Poids | Volume |
|---|---|---|---|
| `metric` (défaut) | centimètres | kilogrammes | m³ |
| `imperial` | pouces | livres | ft³ |

### `level` — niveau de détail du résultat
| Valeur | Retour |
|---|---|
| `simple` (défaut) | Liste concise de transporteurs avec les totaux |
| `detailed` | Inclut le taux de change et la composition détaillée du fret, de l'assurance et de la taxe, avec les règles de subvention appliquées |

Envoyé en **query string** : `POST /v2/quotation?level=detailed`.

## Endpoint : créer une cotation

- **Méthode et route :** `POST /v2/quotation`
- **Query string optionnelle :** `level=simple|detailed`
- **Quand l'utiliser :** lorsque, au moment de coter, vous ne connaissez que le pays (et peut-être l'état) d'origine et de destination.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Défaut | Valeurs acceptées | Description |
|---|---|---|---|---|---|---|---|
| `object` | string | **oui** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Ce qui est envoyé |
| `type` | string | **oui** | - | - | - | `simple`, `advanced`, `items` | Comment le calcul est construit (`items` n'existe qu'en v2) |
| `tax` | string | **oui** | - | - | - | `sender`, `receiver` | Qui paie les taxes (peut être remplacé par le préréglage du compte) |
| `insurance` | boolean | **oui** | - | - | - | `true`, `false` | Souscrire une assurance (peut être activée automatiquement par le compte) |
| `currency_quote` | string | **oui** | 3 | 3 | - | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise dans laquelle la cotation est présentée et dans laquelle `unit_value`/`price` sont indiqués |
| `currency_payment` | string | **oui** | 3 | 3 | - | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise dans laquelle le paiement sera effectué |
| `address_sender` | object | **oui** | - | - | - | - | Origine (voir le tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | - | Destination (voir le tableau ci-dessous) |
| `boxes` | array | conditionnel | - | - | - | - | Obligatoire lorsque `type` n'est **pas** `items`. En `type: items` avec le regroupement `smartbox`, il est également obligatoire |
| `items` | array | conditionnel | - | - | - | - | Obligatoire lorsque `type` n'est **pas** `simple` et qu'aucune boîte n'utilise `kit` |
| `measurement` | string | non | - | - | préréglage du compte ou `metric` | `metric`, `imperial` | Système de mesure |
| `residential_delivery` | boolean | non | - | - | `false` | `true`, `false` | Livraison à une adresse résidentielle |
| `non_stackable` | boolean | non | - | - | `false` | `true`, `false` | Colis non empilables |
| `description` | string | non | - | 155 | `null` | - | Description libre de la cotation |
| `channel_code` | integer | non | - | - | préréglage du compte | - | Code du canal de vente enregistré et autorisé pour votre utilisateur |
| `channel_identifier_number` | string | non | - | - | `null` | - | Votre identifiant commercial (n° de commande). **Recommandé** |
| `channel_order_number` | string | non | - | 100 | `null` | - | Numéro de commande dans le canal de vente |
| `declared_freight` | object | non | - | - | préréglage du compte | - | Fret déclaré. Exige une permission spécifique |
| `manufacturer` | object | non | - | - | `null` | - | Fabricant affiché sur la Commercial Invoice |
| `level` | string | non (query) | - | - | `simple` | `simple`, `detailed` | Niveau de détail du résultat |

### Paramètres — `address_sender` et `address_receiver`

**C'est la principale différence de la v2.** Choisissez l'une des deux formes et ne les mélangez pas :

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code d'une adresse enregistrée dans le panneau de votre compte |

**Forme B — uniquement le pays (et l'état, si vous le connaissez) :**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **oui** | 2 | 2 | Pays en ISO 3166-1 alpha-2 (ex. : `BR`, `US`) |
| `state_code` | string | non | - | - | Sigle de l'état/province (ex. : `SP`, `FL`). Lorsqu'il est indiqué, il améliore la précision du calcul des taxes régionales et est réutilisé dans l'expédition |

> N'envoyez ni `city`, ni `postal_code`, ni `state` dans la cotation v2 : ces champs sont **ignorés** et doivent être
> indiqués dans l'[Expédition V2](api-remessa-v2.md).

> **Expéditeur enregistré obligatoire pour certains types de compte :** les comptes de personne physique et les comptes d'entreprise
> avec un seul centre de distribution doivent indiquer l'expéditeur via la **Forme A** (`address_code`).
> Sans cela, l'API répond `406` avec `Codigo de endereço do remetente não encontrado!`.

### Paramètres — `boxes[]`

Identique à la v1. Il existe **trois façons** de décrire le colis :

**Forme A — kit enregistré** (remplit automatiquement la boîte **et** les articles) :

| Champ | Type | Obligatoire | Max. | Description |
|---|---|---|---|---|
| `kit` | string | **oui** dans cette forme | 50 | SKU du kit enregistré. La boîte et les articles de cette position sont remplis à partir de l'enregistrement du kit |

**Forme B — boîte enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `box_code` | integer | **oui** dans cette forme | Code de la boîte enregistrée ([Boîte](api-caixa-v1.md)) |
| `box_identifier` | string | alternative à `box_code` | Identifiant de la boîte enregistrée (ex. : `CX0236`) |

**Forme C — boîte indiquée manuellement :**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `name` | string | non | 4 | 60 | Nom/étiquette du colis |
| `height` | number | **oui** (sauf palette avec `volume`) | 1 | - | Hauteur |
| `width` | number | **oui** (sauf palette avec `volume`) | 1 | - | Largeur |
| `depth` | number | **oui** (sauf palette avec `volume`) | 1 | - | Profondeur (longueur) |
| `volume` | number | conditionnel | 0.001 | - | **Uniquement lorsque `object` vaut `pallet`.** Alternative à hauteur/largeur/profondeur, en m³ ou ft³ |

**Champs valides dans les formes B et C :**

| Champ | Type | Obligatoire | Min. | Description |
|---|---|---|---|---|
| `weight` | number | **oui** lorsque `object` vaut `doc`/`package` **et** que `type` vaut `simple` | > 0 | Poids du colis. En `advanced`/`items`, lorsqu'il est omis, il correspond à la somme des articles |
| `price` | number | **oui** lorsque `object` vaut `doc`/`package` **et** que `type` vaut `simple` | > 0 | Valeur déclarée du colis. En `advanced`/`items`, lorsqu'il est omis, il correspond à la somme des articles |

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **oui** | 1 | 1000 | Quantité de l'article |
| `unit_value` | number | **oui** | > 0 | - | Valeur unitaire dans la devise de la cotation. Zéro ou une valeur négative est refusé |
| `box_id` | integer | **oui**, sauf lorsque `type` vaut `items` | 1 | nb de boîtes | **Position** de la boîte dans le tableau `boxes` (1 = première boîte). Ce n'est pas le `box_code` |
| `sku` | string | **oui** lorsque `type` vaut `items` | - | 50 | SKU de l'article enregistré ([Articles](api-itens-v1.md)). Les champs omis sont complétés à partir de l'enregistrement |
| `ean` | string | non | - | 50 | Code EAN de l'article |
| `nsmp` | string | non | - | 50 | *Non Standard Manufacturer Product Id* : identifiant du produit chez le fabricant. Imprimé sur l'invoice et utilisé dans les informations réglementaires exigées par certains transporteurs pour les destinations de l'Union européenne |
| `name` | string | **oui** lorsque `sku` n'est pas indiqué | 6 | 50 | Nom de l'article (apparaît sur l'invoice) |
| `description` | string | non | - | 155 | Description détaillée de l'article |
| `country_code` | string | **oui** lorsque `sku` n'est pas indiqué | 2 | 2 | Pays d'origine/de fabrication de l'article |
| `hscode` | string | **oui** lorsque `sku` n'est pas indiqué | 6 | 10 | NCM/HS Code. Définit la table de taxes appliquée |
| `weight` | number | **oui** lorsque `sku` n'est pas indiqué | 0.001 | - | Poids unitaire de l'article |
| `is_groupable` | boolean | non | - | - | Défaut `true`. Lorsque `false`, l'article reçoit une boîte exclusive dans les regroupements de `type: items` |

### Paramètres — `declared_freight`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | non | 0 | - | Valeur du fret déclaré |
| `currency` | string | **oui** lorsque `value` est indiqué | 3 | 3 | Devise du fret déclaré |

> Appliqué uniquement si votre profil dispose de la **permission de fret déclaré** ; sinon, il est ignoré silencieusement.

### Paramètres — `manufacturer`

Identique à la v1 : envoyez `address_code` **ou** les champs manuels (`name`, `country_code`, `state_code`, `state`,
`postal_code`, `city`, `street`, `number`, `district`, `complement`, `email`, `phone`, `phone_extension`, `type`,
`federal_tax_id`, `state_tax_id`). Limites et recommandations dans le
[tableau du fabricant de la v1](api-cotacao-v1.md#paramètres--manufacturer).

| Champ | Type | Recommandé | Description |
|---|---|---|---|
| `address_code` | integer | alternative aux champs manuels | Adresse enregistrée du fabricant |
| `name` | string | **oui** sans `address_code` | Raison sociale/nom du fabricant (5 à 30 caractères) |

### Payload minimal

Le plus petit payload accepté par l'endpoint (cotation de document) :

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 30,
      "width": 25,
      "depth": 2,
      "weight": 0.5,
      "price": 10
    }
  ]
}
```

### Payload maximal

Tous les champs acceptés par l'endpoint, avec les trois formes de boîte :

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "description": "Commande du marketplace, lot octobre",
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
    { "name": "Boite master", "height": 20, "width": 20, "depth": 20, "weight": 2.5, "price": 150 },
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton, manches courtes, couleur blanche",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Baskets de sport",
      "description": "Chaussures de course pour homme",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    },
    {
      "box_id": 3,
      "sku": "CNC-010-M",
      "quantity": 2,
      "unit_value": 15
    }
  ],
  "manufacturer": {
    "name": "Usine Exemple LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Entrepot 2",
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
        "url_image": "https://files.example.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.45",
        "insurance_final": "11.45",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "28153.894",
        "tax_final": "28153.894",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "url_image": "https://files.example.com/contratos/fedex.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.55",
        "insurance_final": "11.55",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "24924.09",
        "tax_final": "24924.09",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Champs de la réponse

Même structure que la v1 — description complète dans [Champs de la réponse de la v1](api-cotacao-v1.md#champs-de-la-réponse).
Les champs les plus utilisés :

| Champ | Type | Description |
|---|---|---|
| `data.quotation` | string (UUID) | **Code de la cotation.** À utiliser dans `POST /v2/shipping` |
| `data.finished` | boolean | `true` lorsque la cotation **a déjà été utilisée** pour créer une expédition |
| `data.warnings` | array | **Optionnel.** N'apparaît que lorsqu'il y a des alertes douanières |
| `carriers[].code` | integer | Code du contrat. À utiliser dans `carrier_code` lors de l'expédition |
| `carriers[].slug` | string | Identifiant textuel du contrat. Alternative à `carrier_code` |
| `carriers[].valid_until` | string | **Validité de la cotation.** Créez l'expédition avant cette date/heure |
| `carriers[].transit_days` | integer | Délai de transit estimé en jours |
| `carriers[].weight_details.type` | string | `real` ou `cubed` — le poids qui a été utilisé pour la tarification |
| `carriers[].freight` / `freight_final` | string | Fret avant / après les frais administratifs |
| `carriers[].insurance` / `insurance_final` | string | Assurance avant / après les frais administratifs |
| `carriers[].tax` / `tax_final` | string | Taxes avant / après les frais administratifs |
| `carriers[].currency_quote_amount` | string | Total à payer dans la devise de la cotation |
| `carriers[].currency_payment_amount` | string | Total à payer dans la devise de paiement |

## Scénarios de cotation

Chaque scénario ci-dessous présente ce qui change dans le payload, le payload minimal et le payload maximal **de ce scénario**.
Les champs racine communs (devises, taxe, assurance, canal, fret déclaré, etc.) suivent le
[tableau de la racine du payload](#paramètres--racine-du-payload).

| Scénario | `object` | `type` | `boxes[]` | `items[]` | Éligible à l'expédition |
|---|---|---|---|---|---|
| [Document](#cotation-de-document) | `doc` | `simple` | Dimensions + `weight` + `price` | Non utilisé | ✅ |
| [Colis — simple](#cotation-de-colis-boîte--simple) | `package` | `simple` | Dimensions + `weight` + `price` | Non utilisé | ❌ simulation uniquement |
| [Colis — avancé](#cotation-de-colis-boîte--avancée) | `package` | `advanced` | Dimensions (poids/valeur optionnels) | Obligatoire, avec `box_id` | ✅ |
| [Palette — par LLH](#cotation-de-palette--par-llh) | `pallet` | `advanced` | `height` + `width` + `depth` | Obligatoire, avec `box_id` | ✅ |
| [Palette — par m³](#cotation-de-palette--par-m³) | `pallet` | `advanced` | `volume` | Obligatoire, avec `box_id` | ✅ |
| [Articles](#cotation-darticles) — **exclusif à la v2** | `package` | `items` | Dépend du regroupement du compte | Obligatoire, avec `sku` | ✅ |

### Cotation de Document

- **Quand l'utiliser :** envoi de papiers/documents sans valeur commerciale significative (contrats, certificats, imprimés).
- **Comment le construire :** `object: doc` + `type: simple`. Chaque colis (enveloppe/boîte) indique les dimensions, `weight` et `price`.
- **N'utilise pas** `items[]` ni `manufacturer` : les documents ne passent pas par le calcul des taxes par article.
- **Est éligible à l'expédition.**

#### Payload minimal

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 }
  ]
}
```

#### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0002",
  "channel_order_number": "334456",
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Envoi de contrats signes",
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "declared_freight": { "value": 20, "currency": "BRL" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Enveloppe A4", "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 },
    { "box_code": 7, "weight": 1.2, "price": 15 }
  ]
}
```

> En `doc`, `weight` et `price` sont obligatoires **pour chaque colis**, y compris lorsque la boîte est enregistrée (`box_code`/`box_identifier`).

### Cotation de Colis (Boîte) — simple

- **Quand l'utiliser :** simulation rapide de fret lorsque vous ne connaissez que le poids et la valeur de chaque boîte.
- **Comment le construire :** `object: package` + `type: simple`. Chaque boîte indique les dimensions, `weight` et `price`. Sans articles.
- ⚠️ **N'est pas éligible à l'expédition.** Pour réellement expédier, refaites la cotation en
  [Colis — avancé](#cotation-de-colis-boîte--avancée) ou [Articles](#cotation-darticles).

#### Payload minimal

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 }
  ]
}
```

#### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "SIM-2026-0001",
  "channel_order_number": "334457",
  "object": "package",
  "type": "simple",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": true,
  "description": "Simulation de fret pour la vitrine",
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "declared_freight": { "value": 30, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Boite moyenne", "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 },
    { "box_identifier": "CX0236", "weight": 3.5, "price": 180 },
    { "box_code": 3, "weight": 1, "price": 60 },
    { "kit": "KIT-001" }
  ]
}
```

> Dans la boîte de `kit`, le poids et les dimensions proviennent de l'enregistrement de la boîte du kit et le `price` est la somme des valeurs des articles du kit.
> Si la boîte enregistrée a un **poids maximal** configuré, le `weight` indiqué ne peut pas le dépasser.

### Cotation de Colis (Boîte) — avancée

- **Quand l'utiliser :** c'est le **flux standard pour l'expédition** de marchandises. Vous déclarez les articles et indiquez dans quelle boîte
  chacun se trouve.
- **Comment le construire :** `object: package` + `type: advanced`. Les boîtes apportent les dimensions ; chaque article indique la
  position de sa boîte dans `box_id` (1 = première boîte).
- Le poids et la valeur de chaque boîte sont calculés par la somme des articles, s'ils ne sont pas indiqués.
- **Chaque boîte doit contenir au moins un article** ; un `box_id` en dehors de l'intervalle des boîtes est refusé.
- **Est éligible à l'expédition.**

#### Payload minimal

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

#### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0003",
  "channel_order_number": "334458",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Commande avec deux boites",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Boite vetements", "height": 30, "width": 25, "depth": 15, "weight": 1.2, "price": 120 },
    { "box_code": 3 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton, manches courtes, couleur blanche",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 4,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Baskets de sport",
      "description": "Chaussures de course pour homme",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9
    }
  ],
  "manufacturer": {
    "name": "Usine Exemple LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Entrepot 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

> Un article avec un `sku` enregistré peut être envoyé uniquement avec `box_id`, `sku`, `quantity` et `unit_value` : le reste
> (nom, pays, HS Code, poids, EAN, `nsmp`) est complété à partir de l'[enregistrement des articles](api-itens-v1.md).
> Ce que vous envoyez dans le payload a priorité sur l'enregistrement.

### Cotation de Palette — par LLH

- **Quand l'utiliser :** chargement palettisé dont vous connaissez les mesures — **L**ongueur (`depth`), **L**argeur (`width`)
  et **H**auteur (`height`).
- **Comment le construire :** `object: pallet` + `type: advanced`, avec `height`, `width` et `depth` pour chaque palette et les articles
  indiquant la palette dans `box_id`.
- La plateforme calcule le volume (`L × l × H ÷ 1 000 000` = m³) à partir des mesures.
- Le poids de la palette, s'il est omis, est la somme des poids des articles. Indiquez `weight` pour inclure le poids de la base/de l'emballage.
- `type: simple` est accepté pour la simulation, mais il est **non éligible à l'expédition**.

#### Payload minimal

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 150, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Piece de rechange industrielle",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0001",
  "channel_order_number": "990011",
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Deux palettes de pieces industrielles",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Palette 01", "height": 150, "width": 100, "depth": 120, "weight": 280, "price": 900 },
    { "name": "Palette 02", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Piece de rechange industrielle",
      "description": "Engrenage en acier pour machine emballeuse",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Roulement industriel",
      "description": "Roulement a billes en acier",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

### Cotation de Palette — par m³

- **Quand l'utiliser :** chargement palettisé dont vous ne connaissez que le **volume** (cubage), sans les mesures individuelles.
- **Comment le construire :** `object: pallet` + `type: advanced`, avec `volume` pour chaque palette (m³ en `metric`, ft³ en `imperial`).
- La plateforme convertit le volume en un cube équivalent pour la tarification :
  `côté (cm) = ∛volume (m³) × 100`. Ex. : `1.0` m³ → 100 × 100 × 100 cm ; `1.8` m³ → 121,64 × 121,64 × 121,64 cm.
- Si vous envoyez `volume` **et** les trois mesures, les mesures (LLH) ont la priorité.
- Vous pouvez mélanger des palettes par LLH et par m³ dans le même payload (chaque position de `boxes[]` décide de sa forme).

#### Payload minimal

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "volume": 1.8 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Piece de rechange industrielle",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0002",
  "channel_order_number": "990012",
  "object": "pallet",
  "type": "advanced",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Palette par volume et palette par mesures",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Palette volume", "volume": 1.8, "weight": 300, "price": 900 },
    { "name": "Palette mesuree", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Piece de rechange industrielle",
      "description": "Engrenage en acier pour machine emballeuse",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Roulement industriel",
      "description": "Roulement a billes en acier",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Une palette sans mesures **et** sans `volume` est refusée avec `422` (validation) ou avec
> `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³`.

### Cotation d'Articles

- **Exclusive à la v2.** C'est l'une des raisons de choisir la v2 : le payload n'a pas besoin de décrire le colis (boîte).
- **Quand l'utiliser :** vous avez déjà les produits dans l'[enregistrement des articles](api-itens-v1.md) et vous souhaitez que la plateforme
  constitue les boîtes elle-même.
- **Comment le construire :** `object: package` + `type: items`. Chaque article indique `sku`, `quantity` et `unit_value` ; n'indiquez
  **pas** `box_id`.
- La façon dont les boîtes sont constituées dépend du **regroupement** configuré sur votre compte
  ([Premiers Pas](api-primeiros-passos.md#regroupement-de-boîtes)) :

| Regroupement | `boxes[]` dans le payload | Prérequis dans l'enregistrement de l'article | Comment les boîtes sont constituées |
|---|---|---|---|
| `default` | Non utilisé | Article lié à une boîte enregistrée (`box_code` ou `box_identifier`) | Les articles regroupables qui utilisent la même boîte sont réunis ; les articles non regroupables reçoivent une boîte par unité |
| `all_in_one` | Non utilisé | Article avec dimensions enregistrées | Les articles regroupables vont dans **une** boîte virtuelle avec le volume total ; les non regroupables reçoivent une boîte par unité |
| `smartbox` | **Obligatoire** (boîtes candidates) | Article avec dimensions enregistrées | Les articles sont placés dans les boîtes envoyées, en tenant compte du volume et des dimensions avec 10 % de marge et de `is_groupable`/`is_foldable` |

#### Payload minimal

Regroupement `default` ou `all_in_one` :

```json
{
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "items": [
    { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
  ]
}
```

#### Payload maximal

Regroupement `smartbox` (boîtes candidates indiquées) :

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0004",
  "channel_order_number": "334459",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Commande constituee via le catalogue des articles",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 25, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "name": "Grande boite", "height": 40, "width": 40, "depth": 40 }
  ],
  "items": [
    {
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton, manches courtes, couleur blanche",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "sku": "TEN-042-M",
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Erreurs typiques de ce scénario : `O item com o sku :name não existe!` (`404`),
> `A caixa com o box_code/box_identifier :name não existe ou está desativada` (`404`),
> `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (`400`) et
> `Nenhuma caixa comporta o item :name (Smart Box).` (`400`).

### Cotation avec Kit

Un **kit** est un ensemble préenregistré dans le panneau (une boîte + des articles avec leurs quantités). En indiquant
`boxes[].kit`, cette position est remplacée par la boîte du kit et les articles du kit sont inclus automatiquement.

- Il peut être combiné avec des boîtes ordinaires dans le même payload.
- Si **toutes** les boîtes sont des kits, `items[]` peut être omis.
- En `type: simple`, le kit devient une boîte avec `price` = somme des articles du kit ; en `advanced`, les articles du kit
  entrent dans la cotation en pointant vers la position du kit.

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "kit": "KIT-001" }
  ]
}
```

| Statut | Message | Cause |
|---|---|---|
| `400` | `O kit :sku é inválido!` | Kit inexistant, inactif ou appartenant à un autre compte |
| `400` | `A caixa do kit :sku é inválida!` | La boîte liée au kit n'existe pas ou a été désactivée |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Un article du kit a été supprimé de l'enregistrement |

## Cotations non éligibles à l'expédition

Les mêmes conditions que la v1 s'appliquent à la v2, vérifiées dans le `POST /v2/shipping` dans cet ordre :

| # | Condition | Statut | Message en cas d'échec |
|---|---|---|---|
| 1 | La cotation existe et a été créée en **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La cotation appartient à votre compte | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Compte avec taxe verrouillée en DDP : cotation DDU (`tax: receiver`) sans paiement approuvé | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | La cotation **n'a pas encore été utilisée** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | Ce n'est pas un `type: simple` avec `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | La cotation **n'a pas expiré** (`valid_until` du transporteur) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | Le transporteur choisi figure dans `carriers[]` de cette cotation | `404` | `Contrato especificado não foi listado para esta cotação!` |

> En v2, l'adresse complète n'est validée que dans l'expédition. Un code postal ou une adresse invalide indiqué dans
> l'[Expédition V2](api-remessa-v2.md#validation-de-ladresse-complète) empêche également la création.

## Règles métier

### Préréglages et restrictions de chargement
Les mêmes règles que la v1 s'appliquent :
[Préréglages appliqués à la cotation](api-cotacao-v1.md#préréglages-appliqués-à-la-cotation),
[Restrictions de chargement](api-cotacao-v1.md#restrictions-de-chargement),
[Alertes de la cotation](api-cotacao-v1.md#alertes-de-la-cotation-warnings) et
[Enregistrements référencés](api-cotacao-v1.md#enregistrements-référencés).

### Adresse sans validation dans la cotation
- En v2, la cotation ne valide **pas** le code postal, la ville ni l'état ; uniquement le pays (et le `state_code`, lorsqu'il est envoyé).
- Le code postal n'est pas exigé pour les adresses au Brésil dans la cotation v2.
- Le `state_code` indiqué dans la cotation est réutilisé dans l'expédition v2.

### Pas d'endpoint de consultation
La v2 **ne dispose pas** de `GET /v2/quotation/{quotation}`. Le résultat complet est fourni dans la réponse du `POST`,
**stockez donc la réponse** si vous devez la présenter à nouveau à l'utilisateur.

## Erreurs courantes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier (chargement, boîte, article, kit) | Lire `message` et ajuster le scénario |
| `403` | Sans permission de cotation ou canal d'un autre compte | Demander la permission au support / revoir `channel_code` |
| `404` | Article/boîte non trouvé dans le regroupement ou aucun contrat éligible | Revoir les enregistrements et le scénario |
| `406` | Le chargement exige un traitement personnalisé, ou un expéditeur enregistré est obligatoire pour le type de compte | Contacter l'équipe commerciale / utiliser `address_sender.address_code` |
| `422` | Validation des champs ou HS Code sans table de taxe | Corriger les champs listés dans `data` / revoir `hscode` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- La v2 échange de la précision contre de la simplicité : l'adresse complète n'intervient que dans l'expédition.
- Les flux opérationnels de **statut, documents, annulation et planification d'enlèvement** utilisent les mêmes
  endpoints `/v1/...`, même pour les expéditions créées en v2 — voir
  [Suivi](api-rastreio.md), [Documents](api-documentos.md) et [Planification](api-agendamento.md).
- Conservez la réponse du `POST` : il n'existe pas d'endpoint de nouvelle consultation en v2.
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
