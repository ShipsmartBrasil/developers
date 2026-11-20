# Cotation (v1)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Calcule le fret, l'assurance et les taxes d'un envoi international, auprès de tous les transporteurs activés sur votre compte.
- C'est la **première étape obligatoire** du flux : l'expédition, le suivi, les documents et la planification d'enlèvement
  sont tous rattachés au code de cotation (`quotation`).
- La v1 est la version complète : elle reçoit l'adresse (pays, ville, code postal/état) dès la cotation, valide cette adresse
  et dispose d'un endpoint de nouvelle consultation du résultat.
- Elle couvre cinq scénarios de chargement : document, colis simple, colis avancé, palette par LLH et palette par m³ —
  voir [Scénarios de cotation](#scénarios-de-cotation). La cotation **par articles** (uniquement `sku`, sans boîtes) est exclusive à la
  [Cotation (v2)](api-cotacao-v2.md#cotation-darticles).

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
2. `POST /v1/quotation` avec l'origine, la destination, l'objet, le type et les boîtes/articles.
3. La réponse contient déjà le résultat calculé (`data.carriers`) et le code `data.quotation`.
4. Si vous devez consulter à nouveau (ou changer le niveau de détail), utilisez `GET /v1/quotation/{quotation}/{level}`.
5. Vérifiez que la cotation est [éligible à l'expédition](#cotations-non-éligibles-à-lexpédition).
6. Choisissez un transporteur dans la liste et utilisez `code` (ou `slug`) + `quotation` pour créer
   l'[expédition](api-remessa-v1.md) **avant** le `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Concepts essentiels

### `object` — ce qui est envoyé
| Valeur | Signification | Remarques |
|---|---|---|
| `doc` | Document | Coté par colis, en indiquant `weight` et `price` de chaque enveloppe/boîte |
| `package` | Colis/boîte | Flux le plus courant |
| `pallet` | Palette | Accepte des dimensions (LLH) **ou** un `volume` en m³/ft³ |
| `not_doc` | Hérité | Conservé pour compatibilité ; il est converti en interne en `package`. **Ne l'utilisez pas dans les nouvelles intégrations** |

### `type` — comment le calcul est construit
| Valeur | Ce que vous envoyez | Quand l'utiliser |
|---|---|---|
| `simple` | Uniquement `boxes[]` avec `weight` et `price` | Documents et simulations rapides de fret. **N'est pas éligible à l'expédition lorsque `object` vaut `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` avec `box_id` indiquant la position de la boîte | Flux standard pour l'expédition : vous contrôlez la répartition des articles dans les boîtes |
| `items` | **Non disponible en v1** | Exclusif à la [Cotation (v2)](api-cotacao-v2.md#cotation-darticles) |

### Matrice d'éligibilité

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Cotation et expédition | Non recommandé | ❌ Uniquement en v2 |
| `package` | ⚠️ Simulation uniquement (non éligible à l'expédition) | ✅ Cotation et expédition | ❌ Uniquement en v2 |
| `pallet` | ⚠️ Simulation uniquement (non éligible à l'expédition) | ✅ Cotation et expédition | ❌ Uniquement en v2 |

> **Attention :** `type: simple` avec `object: package` ou `object: pallet` sert uniquement à la simulation.
> Lors de la tentative de création de l'expédition, l'API répond `403` avec `Cotação Simples é inelegível para remessa!`.
> Détails dans [Cotations non éligibles à l'expédition](#cotations-non-éligibles-à-lexpédition).

### `tax` — qui paie les taxes
| Valeur | Signification |
|---|---|
| `sender` | DDP : l'expéditeur paie les taxes et droits à destination |
| `receiver` | DDU : le destinataire paie les taxes et droits à destination |

> Si votre compte dispose d'un **préréglage de taxe** configuré, celui-ci **remplace** la valeur envoyée dans `tax`.
> Voir [Préréglages appliqués à la cotation](#préréglages-appliqués-à-la-cotation).

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

Envoyé en **query string** dans le `POST` et en **segment de path** dans le `GET`.

## Endpoint : créer une cotation

- **Méthode et route :** `POST /v1/quotation`
- **Query string optionnelle :** `level=simple|detailed`
- **Quand l'utiliser :** chaque fois que vous avez besoin du prix, du délai et des taxes, et avant de créer toute expédition.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Défaut | Valeurs acceptées | Description |
|---|---|---|---|---|---|---|---|
| `object` | string | **oui** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Ce qui est envoyé |
| `type` | string | **oui** | - | - | - | `simple`, `advanced` | Comment le calcul est construit (`items` est exclusif à la v2) |
| `tax` | string | **oui** | - | - | - | `sender`, `receiver` | Qui paie les taxes (peut être remplacé par le préréglage du compte) |
| `insurance` | boolean | **oui** | - | - | - | `true`, `false` | Souscrire une assurance (peut être activée automatiquement par le compte) |
| `currency_quote` | string | **oui** | 3 | 3 | - | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise dans laquelle la cotation est présentée et dans laquelle `unit_value`/`price` sont indiqués |
| `currency_payment` | string | **oui** | 3 | 3 | - | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise dans laquelle le paiement sera effectué |
| `address_sender` | object | **oui** | - | - | - | - | Origine (voir le tableau ci-dessous) |
| `address_receiver` | object | **oui** | - | - | - | - | Destination (voir le tableau ci-dessous) |
| `boxes` | array | **oui** | - | - | - | - | Colis de la cotation (voir le tableau ci-dessous) |
| `items` | array | conditionnel | - | - | - | - | Obligatoire lorsque `type` n'est **pas** `simple` et qu'aucune boîte n'utilise `kit` |
| `measurement` | string | non | - | - | préréglage du compte ou `metric` | `metric`, `imperial` | Système de mesure des dimensions, poids et volumes |
| `ior` | boolean | non | - | - | `false` | `true`, `false` | Importer of Record. Lorsque `true`, `items[].selling_unit_value` est pris en compte |
| `residential_delivery` | boolean | non | - | - | `false` | `true`, `false` | Livraison à une adresse résidentielle |
| `non_stackable` | boolean | non | - | - | `false` | `true`, `false` | Colis non empilables |
| `description` | string | non | - | 155 | `null` | - | Description libre de la cotation, pour votre référence |
| `channel_code` | integer | non | - | - | préréglage du compte | - | Code du canal de vente enregistré et autorisé pour votre utilisateur |
| `channel_identifier_number` | string | non | - | - | `null` | - | Votre identifiant commercial (n° de commande). **Recommandé** : permet de consulter le statut/les documents et d'annuler par canal |
| `channel_order_number` | string | non | - | 100 | `null` | - | Numéro de commande dans le canal de vente |
| `declared_freight` | object | non | - | - | préréglage du compte | - | Fret déclaré (voir le tableau ci-dessous). Exige une permission spécifique |
| `manufacturer` | object | non | - | - | `null` | - | Fabricant affiché sur la Commercial Invoice (voir le tableau ci-dessous) |
| `level` | string | non (query) | - | - | `simple` | `simple`, `detailed` | Niveau de détail du résultat |

### Paramètres — `address_sender` et `address_receiver`

Il existe **deux façons** d'indiquer l'adresse. Choisissez-en une et ne les mélangez pas :

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code d'une adresse enregistrée dans le panneau de votre compte. Lorsqu'il est utilisé, n'envoyez pas les autres champs |

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **oui** | 2 | 2 | Pays en ISO 3166-1 alpha-2 (ex. : `BR`, `US`) |
| `city` | string | **oui** | - | 100 | Ville |
| `postal_code` | string | conditionnel | - | 12 | Code postal/ZIP. Obligatoire si `state` n'est pas indiqué. **Pour les adresses au Brésil, il est toujours obligatoire** |
| `state` | string | conditionnel | - | - | Nom de l'état/province. Obligatoire si `state_code` et `postal_code` ne sont pas indiqués |
| `state_code` | string | conditionnel | - | - | Sigle de l'état/province (ex. : `SP`, `FL`). Obligatoire si `state` et `postal_code` ne sont pas indiqués |

> **Règle pratique :** indiquez toujours `country_code` + `city` + `postal_code` (+ `state_code`, lorsque vous le connaissez).
> Cette combinaison convient à tous les pays et c'est celle qui génère le calcul le plus précis.

> **Expéditeur enregistré obligatoire pour certains types de compte :** les comptes de personne physique et les comptes d'entreprise
> avec un seul centre de distribution doivent indiquer l'expéditeur via la **Forme A** (`address_code`).
> Sans cela, l'API répond `406` avec `Codigo de endereço do remetente não encontrado!`.

### Paramètres — `boxes[]`

Chaque élément du tableau représente un colis (boîte, enveloppe ou palette). Il existe **trois façons** de décrire le colis :

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
| `name` | string | non | 4 | 60 | Nom/étiquette du colis (utilisé dans les messages d'erreur) |
| `height` | number | **oui** (sauf palette avec `volume`) | 1 | - | Hauteur |
| `width` | number | **oui** (sauf palette avec `volume`) | 1 | - | Largeur |
| `depth` | number | **oui** (sauf palette avec `volume`) | 1 | - | Profondeur (longueur) |
| `volume` | number | conditionnel | 0.001 | - | **Uniquement lorsque `object` vaut `pallet`.** Alternative à hauteur/largeur/profondeur, en m³ (`metric`) ou ft³ (`imperial`) |

**Champs valides dans les formes B et C :**

| Champ | Type | Obligatoire | Min. | Description |
|---|---|---|---|---|
| `weight` | number | **oui** lorsque `object` vaut `doc`/`package` **et** que `type` vaut `simple` | > 0 | Poids du colis. En `advanced`, lorsqu'il est omis, il correspond à la somme des poids des articles de cette boîte |
| `price` | number | **oui** lorsque `object` vaut `doc`/`package` **et** que `type` vaut `simple` | > 0 | Valeur déclarée du colis. En `advanced`, lorsqu'il est omis, il correspond à la somme de `unit_value × quantity` des articles de cette boîte |

> `weight` et `price` sont **ignorés** lorsque la boîte utilise `kit`.

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **oui** | 1 | 1000 | Quantité de l'article |
| `unit_value` | number | **oui** | > 0 | - | Valeur unitaire dans la devise de la cotation. Zéro ou une valeur négative est refusé |
| `box_id` | integer | **oui** | 1 | nb de boîtes | **Position** de la boîte dans le tableau `boxes` (1 = première boîte, 2 = deuxième...). Ce n'est pas le `box_code` |
| `sku` | string | non | - | 50 | SKU de l'article enregistré ([Articles](api-itens-v1.md)). Les champs omis sont complétés à partir de l'enregistrement |
| `ean` | string | non | - | 50 | Code EAN de l'article |
| `nsmp` | string | non | - | 50 | *Non Standard Manufacturer Product Id* : identifiant du produit chez le fabricant. Imprimé sur l'invoice et utilisé dans les informations réglementaires exigées par certains transporteurs pour les destinations de l'Union européenne |
| `name` | string | **oui** lorsque `sku` n'est pas indiqué | 6 | 50 | Nom de l'article (apparaît sur l'invoice) |
| `description` | string | non | - | 155 | Description détaillée de l'article |
| `country_code` | string | **oui** lorsque `sku` n'est pas indiqué | 2 | 2 | Pays d'origine/de fabrication de l'article |
| `hscode` | string | **oui** lorsque `sku` n'est pas indiqué | 6 | 10 | NCM/HS Code de l'article. Définit la table de taxes appliquée |
| `weight` | number | **oui** lorsque `sku` n'est pas indiqué | 0.001 | - | Poids unitaire de l'article |
| `selling_unit_value` | number | non | - | - | Valeur de vente unitaire. Prise en compte lorsque `ior` vaut `true` ; si elle est omise, la valeur de l'enregistrement de l'article est utilisée et, à défaut, le `unit_value` |
| `is_groupable` | boolean | non | - | - | Défaut `true`. Utilisé dans les regroupements automatiques de boîtes de la cotation par articles (v2) |

> **Les articles ayant le même `name`** génèrent un `warning` dans la réponse
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> La cotation est effectuée, mais il est conseillé de différencier les noms pour éviter une rétention en douane.

### Paramètres — `declared_freight`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | non | 0 | - | Valeur du fret déclaré |
| `currency` | string | **oui** lorsque `value` est indiqué | 3 | 3 | Devise du fret déclaré. Même liste que `currency_quote` |

> Le fret déclaré n'est appliqué que si votre profil dispose de la **permission de fret déclaré**. Sans elle, le champ est
> ignoré silencieusement (la cotation est effectuée normalement, sans fret déclaré).

### Paramètres — `manufacturer`

Données du fabricant imprimées sur la Commercial Invoice. Envoyez `address_code` **ou** les champs manuels.

| Champ | Type | Recommandé | Min. | Max. | Description |
|---|---|---|---|---|---|
| `address_code` | integer | alternative aux champs manuels | - | - | Code d'adresse enregistrée. Lorsqu'il est utilisé, les données de l'enregistrement remplacent les champs manuels |
| `name` | string | **oui** sans `address_code` | 5 | 30 | Raison sociale/nom du fabricant |
| `country_code` | string | **oui** sans `address_code` | 2 | 2 | Pays |
| `state_code` | string | **oui** sans `address_code` | 1 | 10 | Sigle de l'état |
| `state` | string | non | - | 155 | Nom de l'état |
| `postal_code` | string | **oui** sans `address_code` | - | 12 | Code postal/ZIP |
| `city` | string | **oui** sans `address_code` | - | 100 | Ville |
| `street` | string | **oui** sans `address_code` | - | 70 | Voie |
| `number` | string | non | - | 6 | Numéro |
| `district` | string | non | - | 30 | Quartier |
| `complement` | string | non | - | 20 | Complément |
| `email` | string | **oui** sans `address_code` | - | 60 | E-mail |
| `phone` | string | **oui** sans `address_code` | - | 25 | Téléphone |
| `phone_extension` | string | non | - | 6 | Poste |
| `type` | string | **oui** sans `address_code` | - | - | `pf` ou `pj` |
| `federal_tax_id` | string | non | - | 20 | CPF/CNPJ ou équivalent |
| `state_tax_id` | string | non | - | 20 | Inscription fiscale de l'état |

> L'API **ne rejette pas** un `manufacturer` incomplet dans la cotation, mais les données sont imprimées sur l'invoice :
> envoyez tous les champs marqués comme recommandés pour éviter des problèmes lors du dédouanement.
> Pour les articles dont le `hscode` commence par `64` (chaussures), le nom du fabricant est exigé par la douane.

### Payload minimal

Le plus petit payload accepté par l'endpoint (cotation de document). Les payloads minimaux de chaque type de chargement se trouvent
dans [Scénarios de cotation](#scénarios-de-cotation).

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "postal_code": "33101"
  },
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
  "ior": true,
  "description": "Commande du marketplace, lot octobre",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    {
      "name": "Boite master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 2.5,
      "price": 150
    },
    {
      "box_identifier": "CX0236"
    },
    {
      "box_code": 3
    },
    {
      "kit": "KIT-001"
    }
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
      "selling_unit_value": 49.9,
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
      "selling_unit_value": 349.9,
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

> En `type: advanced`, **chaque boîte doit contenir au moins un article** (sauf la boîte de `kit`, qui apporte déjà les siens).
> C'est pourquoi l'exemple associe un article aux boîtes 1, 2 et 3.

### Exemple de requête

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/quotation?level=detailed' \
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
    "address_sender": {"country_code": "BR", "city": "Diadema", "postal_code": "09941070"},
    "address_receiver": {"country_code": "US", "city": "Miami", "state_code": "FL", "postal_code": "33101"},
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
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
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
      }
    ]
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.quotation` | string (UUID) | **Code de la cotation.** Conservez-le : c'est la clé de tout le flux |
| `data.finished` | boolean | `true` lorsque la cotation **a déjà été utilisée** pour créer une expédition ; `false` tant qu'elle est encore disponible |
| `data.date` | string | Date/heure de la cotation (`AAAA-MM-JJ HH:MM:SS`) |
| `data.description` | string | Description envoyée dans `description` |
| `data.channel` | string | Nom du canal de vente, lorsqu'il y a un `channel_code` |
| `data.channel_identifier_number` | string | Identifiant commercial envoyé dans la cotation |
| `data.object` / `data.type` / `data.tax` / `data.insurance` | - | Valeurs effectivement utilisées dans le calcul (après les préréglages du compte) |
| `data.currency_quote` / `data.currency_payment` | string | Devises de la cotation et du paiement |
| `data.currency_quote_exchange_value` | string | **Uniquement en `detailed`.** Taux de change appliqué à la devise de la cotation |
| `data.currency_payment_exchange_value` | string | **Uniquement en `detailed`.** Taux de change appliqué à la devise de paiement |
| `data.warnings` | array | **Optionnel.** N'apparaît que lorsqu'il y a des alertes (voir [Alertes de la cotation](#alertes-de-la-cotation-warnings)) |
| `data.carriers[]` | array | Options d'envoi calculées |
| `carriers[].code` | integer | Code du contrat. À utiliser dans `carrier_code` lors de l'expédition |
| `carriers[].slug` | string | Identifiant textuel du contrat. Alternative à `carrier_code` lors de l'expédition |
| `carriers[].name` | string | Nom commercial du transporteur/contrat |
| `carriers[].url_image` | string | Logo du transporteur |
| `carriers[].valid_until` | string | **Validité de la cotation.** L'expédition doit être créée avant cette date/heure |
| `carriers[].transit_days` | integer | Délai de transit estimé en jours |
| `carriers[].weight_details.type` | string | `real` (poids réel) ou `cubed` (poids volumétrique) — celui qui a été utilisé pour la tarification |
| `carriers[].weight_details.weight` | string | Poids pris en compte |
| `carriers[].freight` / `freight_final` | string | Fret avant / après les frais administratifs |
| `carriers[].insurance` / `insurance_final` | string | Assurance avant / après les frais administratifs |
| `carriers[].tax` / `tax_final` | string | Taxes avant / après les frais administratifs |
| `carriers[].*_subsidy_rule_code` | integer | **Uniquement en `detailed`.** Code de la règle de subvention appliquée au fret, à l'assurance ou à la taxe (`null` lorsqu'il n'y en a pas) |
| `carriers[].*_details` | object | **Uniquement en `detailed`.** Composition détaillée du fret, de l'assurance ou de la taxe (`null` lorsqu'il n'y en a pas) |
| `carriers[].currency_quote_price` | string | Total dans la devise de la cotation, avant les frais administratifs |
| `carriers[].currency_quote_amount` | string | Total à payer dans la devise de la cotation |
| `carriers[].currency_payment_amount` | string | Total à payer dans la devise de paiement |

## Endpoint : consulter le résultat de la cotation

- **Méthode et route :** `GET /v1/quotation/{quotation}/{level}`
- **Quand l'utiliser :** pour consulter à nouveau un résultat déjà calculé, changer le niveau de détail ou savoir si la cotation
  a déjà été utilisée (`finished`).

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|---|
| `quotation` | path | string | **oui** | UUID | Code renvoyé lors de la création |
| `level` | path | string | non | `simple`, `detailed` | Défaut `simple` |

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

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

> La réponse a la même structure de `data` que le `POST` (sauf `warnings`).

| Statut | Message | Cause |
|---|---|---|
| `400` | `Não foi encontrada a cotação especificada!` | Le `quotation` n'existe pas ou n'appartient pas à votre compte |
| `400` | `Nivel de detalhe da cotação não existe` | `level` différent de `simple`/`detailed` |
| `403` | `Has no permission!` | Votre profil n'a pas la permission de consultation de cotation |

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
| [Articles](#cotation-darticles) | `package` | `items` | - | - | **Exclusif à la v2** |

### Cotation de Document

- **Quand l'utiliser :** envoi de papiers/documents sans valeur commerciale significative (contrats, certificats, imprimés).
- **Comment le construire :** `object: doc` + `type: simple`. Chaque colis (enveloppe/boîte) indique les dimensions, `weight` et `price`.
- **N'utilise pas** `items[]`, `ior` ni `manufacturer` : les documents ne passent pas par le calcul des taxes par article.
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  [Colis — avancé](#cotation-de-colis-boîte--avancée) (ou, en v2, [Articles](api-cotacao-v2.md#cotation-darticles)).

#### Payload minimal

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": true,
  "description": "Commande avec deux boites",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
      "selling_unit_value": 49.9,
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
      "selling_unit_value": 349.9
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Deux palettes de pieces industrielles",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Palette par volume et palette par mesures",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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

> **Exclusive à la [Cotation (v2)](api-cotacao-v2.md#cotation-darticles).** La cotation par articles (`type: items`), dans laquelle vous
> envoyez uniquement le `sku` des articles enregistrés et la plateforme constitue les boîtes, ne fait pas partie du contrat de la v1.
> En v1, indiquez toujours les boîtes et utilisez [Colis — avancé](#cotation-de-colis-boîte--avancée) avec `items[].sku`
> pour tirer parti de l'enregistrement des articles.

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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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

Une cotation calculée avec succès ne peut **pas toujours** devenir une expédition. Avant d'appeler `POST /v1/shipping`,
vérifiez les conditions ci-dessous. Elles sont vérifiées dans cet ordre au moment de l'expédition :

| # | Condition | Statut | Message en cas d'échec |
|---|---|---|---|
| 1 | La cotation existe et a été créée en **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La cotation appartient à votre compte | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Compte avec taxe verrouillée en DDP : cotation DDU (`tax: receiver`) sans paiement approuvé | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | La cotation **n'a pas encore été utilisée** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | Ce n'est pas un `type: simple` avec `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | La cotation **n'a pas expiré** (`valid_until` du transporteur) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | Le transporteur choisi figure dans `carriers[]` de cette cotation | `404` | `Contrato especificado não foi listado para esta cotação!` |

> **Comment l'éviter :** utilisez `advanced` pour colis/palette, créez l'expédition juste après la cotation,
> utilisez le `code`/`slug` renvoyé **dans cette** cotation et faites **une cotation par expédition**.
> Toutes les validations de l'expédition se trouvent dans [Expédition (v1)](api-remessa-v1.md#cotations-non-éligibles-à-lexpédition).

## Règles métier

### Préréglages appliqués à la cotation
Les [préréglages du compte](api-primeiros-passos.md#préréglages-du-compte-presets) influent sur le calcul :

| Préréglage | Effet sur la cotation |
|---|---|
| Taxe (`tax`) | Lorsqu'il est configuré, il **remplace** le `tax` envoyé. Les pays de destination listés comme exception reçoivent l'inverse du défaut. Si le compte est verrouillé en DDP, la taxe est toujours `sender` |
| Assurance automatique | Si elle est active, l'assurance est souscrite automatiquement lorsque la valeur déclarée (convertie en USD) atteint le minimum configuré, même avec `insurance: false` |
| Fret déclaré | Utilisé lorsque `declared_freight` n'est pas envoyé. Exige la permission de fret déclaré |
| Système de mesure | Utilisé lorsque `measurement` n'est pas envoyé. À défaut, `metric` |
| Livraison résidentielle / non empilable | Utilisés lorsque `residential_delivery` / `non_stackable` ne sont pas envoyés |
| Canal de vente | Utilisé lorsque `channel_code` n'est pas envoyé |
| Regroupement de boîtes | Ne s'applique pas à la v1 (utilisé uniquement dans la cotation par articles de la v2) |

### Validation de l'adresse
En v1, l'adresse manuelle est **validée dès la cotation** :

| Cas | Statut | Message |
|---|---|---|
| Adresse au Brésil sans `postal_code` | `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` |
| Code postal brésilien inexistant/invalide (lorsque `state_code` n'est pas envoyé) | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Adresse internationale introuvable | `404` | `Endereço internacional enviado está errado ou não existe!` |
| État/province non reconnu | `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |
| `address_code` inexistant ou appartenant à un autre compte | `400` | `Nenhum endereco encontrado para o usuário especificado!` |
| Expéditeur et destinataire identiques | `400` | `Endereço de Remetente e Destinatário são iguais!` |

### Restrictions de chargement
La cotation peut être refusée lorsque le chargement ne correspond pas aux contrats de votre compte :

| Statut | Message | Cause |
|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Boîte sans article en `advanced` |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` supérieur au nombre de boîtes ou inférieur à 1 |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Article manuel sans `box_id` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | `weight` supérieur au poids maximal de la boîte enregistrée (`simple`) |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Somme des articles supérieure au poids maximal de la boîte enregistrée |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inexistant ou appartenant à un autre compte |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` non enregistré et données manuelles incomplètes |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...` | `unit_value` inférieur ou égal à zéro |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` d'un autre compte |
| `404` | `Nenhum transportador encontrado para este usuário!` | Compte sans contrat activé pour la route/l'objet |
| `404` | `Não há contratos que atendem as especificações do pacote enviado...` | Aucun contrat ne couvre la combinaison poids/dimension/route |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Chargement hors de la plage automatique — contactez l'équipe commerciale |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Aucun transporteur n'a renvoyé de résultat |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item)...` | HS Code sans table de taxe à destination — révisez le `hscode` |

> Les refus liés au **poids** peuvent arriver avec `status: "error"` au lieu de `"warning"`, en conservant le code HTTP
> du tableau. Traitez-les selon le code HTTP et le `message`.

### Alertes de la cotation (`warnings`)
La cotation est effectuée, mais `data.warnings` apparaît avec :

| Alerte | Quand |
|---|---|
| `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` | Deux articles ou plus avec le même `name` |
| `Valores declarados dos itens podem ocasionar problemas aduaneiros` | Valeur déclarée supérieure à 800 USD avec un transporteur express (DHL/FedEx) dans le résultat |

### Enregistrements référencés
Lorsque vous utilisez des codes d'enregistrement, ils doivent exister, être actifs et appartenir à votre compte :

| Champ | Enregistrement | Où le gérer |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Boîte | [Boîte](api-caixa-v1.md) |
| `items[].sku` | Article | [Articles](api-itens-v1.md) |
| `boxes[].kit` | Kit | Panneau de Shipsmart Global |
| `address_*.address_code` / `manufacturer.address_code` | Adresse | Panneau de Shipsmart Global |
| `channel_code` | Canal de vente | Panneau de Shipsmart Global |

## Erreurs courantes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier (chargement, boîte, article, kit, adresse) | Lire `message` et ajuster le scénario |
| `403` | Sans permission de cotation ou canal d'un autre compte | Demander la permission au support / revoir `channel_code` |
| `404` | Adresse introuvable ou aucun contrat éligible | Revoir l'adresse et le scénario |
| `406` | Le chargement exige un traitement personnalisé, ou un expéditeur enregistré est obligatoire pour le type de compte | Contacter l'équipe commerciale / utiliser `address_sender.address_code` |
| `422` | Validation des champs ou HS Code sans table de taxe | Corriger les champs listés dans `data` / revoir `hscode` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- La réponse du `POST` contient déjà le résultat calculé — il n'est pas obligatoire d'appeler le `GET` ensuite.
- `data.warnings` est informatif : la cotation a été effectuée, mais il existe un risque douanier à examiner.
- Pour un payload de cotation encore plus concis (uniquement pays et état), voir [Cotation V2](api-cotacao-v2.md).
- Pour choisir entre les versions, voir [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md).
- Pour cotation + expédition en un seul appel synchrone, voir [Manifeste](api-manifesto.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
