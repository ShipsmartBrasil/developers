# Cotation (v1)

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Calcule le fret, l'assurance et les taxes pour un envoi international, chez tous les transporteurs activés sur votre compte.
- C'est la **première étape obligatoire** du flux : l'expédition, le suivi, les documents et la planification d'enlèvement
  sont tous rattachés au code de cotation (`quotation`).
- La v1 est la version complète : elle accepte une adresse détaillée à l'origine/destination et possède un endpoint de reconsultation du résultat.

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
1. `POST /v1/quotation` avec l'origine, la destination, l'objet, le type et les boîtes/articles.
2. La réponse contient déjà le résultat calculé (`data.carriers`) et le code `data.quotation`.
3. Si vous devez reconsulter (ou changer le niveau de détail), utilisez `GET /v1/quotation/{quotation}/{level}`.
4. Choisissez un transporteur dans la liste et utilisez `code` (ou `slug`) + `quotation` pour créer
   l'[expédition](api-remessa-v1.md) **avant** la date `valid_until`.

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
| `doc` | Document | Cotation par boîte, en indiquant `weight` et `price` |
| `package` | Colis/boîte | Flux le plus courant |
| `pallet` | Palette | Permet d'indiquer `volume` au lieu de hauteur/largeur/profondeur |
| `not_doc` | Obsolète | Conservé pour compatibilité ; converti en interne en `package`. **Ne l'utilisez pas dans de nouvelles intégrations** |

### `type` — comment le calcul est construit
| Valeur | Ce que vous envoyez | Quand l'utiliser |
|---|---|---|
| `simple` | Uniquement `boxes[]` avec `weight` et `price` | Simulation rapide de fret. **Non éligible à l'expédition quand `object` vaut `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensions) + `items[]` avec `box_id` indiquant la position de la boîte | Flux standard pour l'expédition : vous contrôlez la répartition des articles dans les boîtes |
| `items` | Uniquement `items[]` avec `sku` (les boîtes sont choisies par la plateforme) | Quand les articles et les boîtes sont déjà enregistrés et que vous voulez que la plateforme fasse l'agencement |

> **Attention :** `type: simple` avec `object: package` ou `object: pallet` ne sert qu'à la simulation.
> En tentant de créer l'expédition, l'API répond `Cotação Simples é inelegível para remessa!`.
> Pour l'expédition, utilisez `advanced` ou `items`.

### `tax` — qui paie les taxes
| Valeur | Signification |
|---|---|
| `sender` | DDP : l'expéditeur paie les taxes et les frais à destination |
| `receiver` | DDU : le destinataire paie les taxes et les frais à destination |

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

Envoyé en **query string** dans le `POST` et en **segment de path** dans le `GET`.

## Endpoint : créer une cotation

- **Méthode et route :** `POST /v1/quotation`
- **Query string optionnelle :** `level=simple|detailed`
- **Quand l'utiliser :** chaque fois que vous avez besoin du prix, du délai et des taxes, et avant de créer toute expédition.

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
| `ior` | boolean | non | - | - | `false` | `true`, `false` | Importer of Record. Quand `true`, `items[].selling_unit_value` devient nécessaire |
| `residential_delivery` | boolean | non | - | - | `false` | `true`, `false` | Livraison à une adresse résidentielle |
| `non_stackable` | boolean | non | - | - | `false` | `true`, `false` | Colis non empilables |
| `description` | string | non | - | 155 | `null` | - | Description libre de la cotation, pour votre référence |
| `channel_code` | integer | non | - | - | préréglage du compte | - | Code du canal de vente enregistré sur votre compte |
| `channel_identifier_number` | string | non | - | - | `null` | - | Votre identifiant commercial (n° de commande). **Recommandé** : permet de consulter le statut/les documents par canal |
| `channel_order_number` | string | non | - | 100 | `null` | - | Numéro de la commande dans le canal de vente |
| `declared_freight` | object | non | - | - | préréglage du compte | - | Fret déclaré (voir tableau ci-dessous) |
| `manufacturer` | object | non | - | - | `null` | - | Fabricant affiché sur la Commercial Invoice (voir tableau ci-dessous) |
| `level` | string | non (query) | - | - | `simple` | `simple`, `detailed` | Niveau de détail du résultat |

### Paramètres — `address_sender` et `address_receiver`

Il y a **deux façons** d'indiquer l'adresse. Choisissez-en une et ne les mélangez pas :

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code d'une adresse enregistrée sur votre compte. Lorsqu'il est utilisé, n'envoyez pas les autres champs |

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `country_code` | string | **oui** | 2 | 2 | Pays en ISO 3166-1 alpha-2 (ex. : `BR`, `US`) |
| `city` | string | **oui** | - | 100 | Ville |
| `postal_code` | string | conditionnel | - | 12 | Code postal/ZIP. Obligatoire si `state` n'est pas indiqué. **Pour les adresses au Brésil, il est toujours obligatoire** |
| `state` | string | conditionnel | - | - | Nom de l'état/province. Obligatoire si `state_code` et `postal_code` ne sont pas indiqués |
| `state_code` | string | conditionnel | - | 10 | Sigle de l'état/province (ex. : `SP`, `FL`). Obligatoire si `state` et `postal_code` ne sont pas indiqués |

> **Règle pratique :** indiquez toujours `country_code` + `city` + (`postal_code` **ou** `state_code`).
> Cette combinaison couvre tous les pays.

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

> `weight` et `price` sont **ignorés** quand la boîte utilise `kit`.

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `quantity` | integer | **oui** | 1 | 1000 | Quantité de l'article |
| `unit_value` | number | **oui** | - | - | Valeur unitaire dans la devise de la cotation |
| `box_id` | integer | **oui**, sauf quand `type` vaut `items` | - | - | **Position** de la boîte dans le tableau `boxes` (1 = première boîte, 2 = deuxième...). Ce n'est pas le `box_code` |
| `sku` | string | **oui** quand `type` vaut `items` | - | 50 | SKU de l'article enregistré ([Articles](api-itens-v1.md)). Lorsqu'il est indiqué, les données de l'enregistrement sont utilisées et les autres champs peuvent être omis |
| `ean` | string | non | - | 50 | Code EAN de l'article |
| `name` | string | **oui** quand `sku` n'est pas indiqué | 6 | 50 | Nom de l'article (apparaît sur l'invoice) |
| `description` | string | non | - | 155 | Description détaillée de l'article |
| `country_code` | string | **oui** quand `sku` n'est pas indiqué | 2 | 2 | Pays d'origine/de fabrication de l'article |
| `hscode` | string | **oui** quand `sku` n'est pas indiqué | 6 | 10 | NCM/HS Code de l'article. Définit la table de taxes appliquée |
| `weight` | number | **oui** quand `sku` n'est pas indiqué | 0.001 | - | Poids unitaire de l'article |
| `selling_unit_value` | number | conditionnel | - | - | Valeur de vente unitaire. **Nécessaire quand `ior` vaut `true`** |
| `is_groupable` | boolean | non | - | - | Par défaut `true`. Quand `false`, l'article reçoit une boîte exclusive dans le regroupement `smartbox` |

> **Des descriptions identiques sur des articles différents** génèrent un `warning` dans la réponse
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> La cotation est réalisée, mais il vaut mieux revoir les descriptions pour éviter une retenue en douane.

### Paramètres — `declared_freight`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | non | 0 | - | Montant du fret déclaré |
| `currency` | string | **oui** quand `value` est indiqué | 3 | 3 | Devise du fret déclaré. Même liste que `currency_quote` |

### Paramètres — `manufacturer`

Données du fabricant imprimées sur la Commercial Invoice. Envoyez `address_code` **ou** les champs manuels.

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `address_code` | integer | alternative aux champs manuels | - | - | Code d'adresse enregistrée. Lorsqu'il est utilisé, les autres champs sont optionnels |
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

> Pour les articles dont le `hscode` commence par `64` (chaussures), le nom du fabricant est exigé par la douane.

### Payload minimal

Cotation d'un document, type simple — le plus petit payload accepté :

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
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
    "state_code": "FL",
    "postal_code": "33101"
  },
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

Pour générer l'expédition d'un colis, il faut utiliser `advanced` (ou `items`) et déclarer les articles :

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
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

Tous les champs acceptés par l'endpoint :

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
  "description": "Commande du marketplace, lot d'octobre",
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
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
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
| `data.finished` | boolean | Indique si le traitement de la cotation est terminé |
| `data.date` | string | Date/heure de la cotation |
| `data.channel` | string | Nom du canal de vente, quand `channel_code` a été indiqué |
| `data.warnings` | array | **Optionnel.** N'apparaît que lorsqu'il y a des alertes (ex. : descriptions dupliquées, valeurs déclarées élevées) |
| `data.carriers[]` | array | Options d'envoi calculées |
| `carriers[].code` | integer | Code du contrat. À utiliser dans `carrier_code` de l'expédition |
| `carriers[].slug` | string | Identifiant textuel du contrat. Alternative à `carrier_code` dans l'expédition |
| `carriers[].name` | string | Nom commercial du transporteur |
| `carriers[].url_image` | string | Logo du transporteur |
| `carriers[].valid_until` | string | **Validité de la cotation.** L'expédition doit être créée avant cette date/heure |
| `carriers[].transit_days` | integer | Délai de transit estimé en jours |
| `carriers[].weight_details.type` | string | `real` (poids réel) ou `cubed` (poids volumétrique) — celui utilisé pour la tarification |
| `carriers[].weight_details.weight` | string | Poids retenu |
| `carriers[].freight` / `freight_final` | string | Fret avant / après les frais administratifs |
| `carriers[].insurance` / `insurance_final` | string | Assurance avant / après les frais administratifs |
| `carriers[].tax` / `tax_final` | string | Taxes avant / après les frais administratifs |
| `carriers[].currency_quote_price` | string | Total dans la devise de la cotation |
| `carriers[].currency_quote_amount` | string | Total à payer dans la devise de la cotation |
| `carriers[].currency_payment_amount` | string | Total à payer dans la devise de paiement |

## Endpoint : consulter le résultat de la cotation

- **Méthode et route :** `GET /v1/quotation/{quotation}/{level}`
- **Quand l'utiliser :** pour reconsulter un résultat déjà calculé ou changer le niveau de détail.

| Paramètre | Emplacement | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|---|
| `quotation` | path | string | **oui** | UUID | Code retourné à la création |
| `level` | path | string | non | `simple`, `detailed` | Par défaut `simple` |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

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

> Si le `quotation` n'existe pas ou n'appartient pas à votre compte, l'API répond `400` avec
> `Não foi encontrada a cotação especificada!`.

## Règles métier

### Éligibilité à l'expédition
Pour qu'une cotation puisse générer une expédition, toutes les conditions ci-dessous doivent être vraies :

1. Le `quotation` existe et appartient à votre compte.
2. La cotation **n'est pas expirée** (respectez `valid_until` du transporteur choisi).
3. La cotation **n'a pas encore été utilisée** dans une autre expédition (`Cotação já foi utilizada em outra remessa!`).
4. La cotation a été créée dans la **même version** de l'API que celle utilisée pour l'expédition (v1 → v1).
5. Ce n'est pas une cotation `simple` de `package`/`pallet` (`Cotação Simples é inelegível para remessa!`).
6. Le transporteur indiqué figure dans la liste retournée par la cotation
   (`Contrato especificado não foi listado para esta cotação!`).

### Restrictions de charge
La cotation peut être refusée avec `400` quand :

| Message | Cause |
|---|---|
| `A caixa :name tem a altura/largura/profundidade maior do que a permitida!` | Dimension au-delà de la limite des contrats |
| `A caixa :name está com o peso maior que o permitido!` | Poids par colis au-delà de la limite |
| `Remessa está com o peso acima do permitido!` | Poids total au-delà de la limite |
| `Nenhum item foi associado para a caixa :name !` | Boîte sans article en `advanced` |
| `A cotação excedeu o máximo de itens permitido!` | Nombre d'articles au-delà de la limite |
| `Nenhum transportador encontrado para este usuário!` | Compte sans contrat activé |
| `Não há contratos que atendem as especificações do pacote enviado...` | Aucun contrat ne couvre la combinaison poids/dimension/route |
| `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Charge hors de la plage automatique — contactez l'équipe commerciale |
| `Endereço de Remetente e Destinatário são iguais!` | Origine et destination identiques |
| `O país de origem é o mesmo de destino!` | L'API traite des envois internationaux |
| `Não foi passado um postal code para o endereço do Brasil no :name !` | Adresse brésilienne sans code postal |
| `Este país de Origem/Destino está temporariamente inabilitado para envios!` | Route bloquée pour le moment |
| `Não existe tabela de duty para o país de destino :name com o hscode :hscode...` | HS Code sans table de taxes à destination — revoyez le `hscode` |

### Enregistrements référencés
Quand vous utilisez des codes d'enregistrement, ils doivent exister et être actifs :

| Champ | Enregistrement | Page |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Boîte | [Boîte](api-caixa-v1.md) |
| `items[].sku` | Article | [Articles](api-itens-v1.md) |
| `address_*.address_code` | Adresse | Enregistrée dans le portail |
| `channel_code` | Canal de vente | Enregistré dans le portail |
| `boxes[].kit` | Kit | Enregistré dans le portail |

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier (restriction de charge, route, contrat, HS Code) | Lire `message` et ajuster le scénario |
| `403` | Sans permission de cotation | Demander la permission au support |
| `406` | Structure du payload incompatible | Revoir les types et l'imbrication |
| `422` | Validation des champs | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- La réponse du `POST` contient déjà le résultat calculé — il n'est pas obligatoire d'appeler le `GET` ensuite.
- `data.warnings` est informatif : la cotation a été réalisée, mais il y a un risque douanier à examiner.
- Pour un payload de cotation encore plus allégé (pays et état uniquement), voir [Cotation V2](api-cotacao-v2.md).
- Pour cotation + expédition en un seul appel synchrone, voir [Manifeste](api-manifesto.md).
