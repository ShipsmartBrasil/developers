# Manifeste

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Exécute **cotation + expédition + génération de documents en un seul appel synchrone**.
- La réponse contient déjà le numéro de suivi et les documents (AWB, et les autres le cas échéant).
- C'est le flux indiqué pour les opérations qui ont besoin de l'étiquette **immédiatement**, sans attendre un
  traitement asynchrone.

> ⚠️ **Le manifeste est synchrone et irréversible.** Comme il cote et expédie dans la même requête, il n'y a pas
> d'étape intermédiaire de vérification. Validez bien le payload avant l'envoi.
>
> ⚠️ **N'accepte que le paiement facturé** (`payment.type: billed`).

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

## Manifeste vs. cotation + expédition

| Thème | Cotation + Expédition | Manifeste |
|---|---|---|
| Appels | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Choix du transporteur | Vous choisissez dans la liste retournée | La plateforme décide (ou vous le fixez avec `carrier_code`/`slug`) |
| Documents | Consultation ultérieure sur `/docs` | **Arrivent dans la réponse** |
| Paiement | `billed`, `gateway`, `afterward` | **Uniquement `billed`** |
| Type de réponse | Asynchrone | **Synchrone** |
| Vérification avant l'expédition | Oui | Non |
| Champs obligatoires | Moins (répartis sur 2 étapes) | **Plus** (tout d'un coup) |

Utilisez le manifeste quand : l'opération est à fort volume, le transporteur est déjà défini par une règle
métier et vous devez imprimer l'étiquette sur le champ.

## Flux d'utilisation
1. Construisez le payload complet (adresses, boîtes, articles, paiement).
2. `POST /v1/manifest?format=pdf` (ou `zpl`).
3. Imprimez l'étiquette reçue dans `data.docs`.
4. Utilisez le `data.shipping` pour le [suivi](api-rastreio.md), les [documents](api-documentos.md) et la
   [planification d'enlèvement](api-agendamento.md).

## Endpoint : créer un manifeste

- **Méthode et route :** `POST /v1/manifest`

### Paramètres de query

| Nom | Emplacement | Type | Obligatoire | Par défaut | Valeurs | Description |
|---|---|---|---|---|---|---|
| `format` | query | string | non | tous | `pdf`, `zpl` | Format des documents dans la réponse. Vide, tous les formats disponibles sont retournés |

> `format` peut aussi être envoyé **dans le corps** du payload. Lorsqu'il est envoyé dans les deux endroits,
> **la valeur du corps prévaut**.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Valeurs / Par défaut | Description |
|---|---|---|---|---|---|---|
| `object` | string | **oui** | - | - | `doc`, `package`, `pallet`, `not_doc` | Ce qui est envoyé |
| `type` | string | **oui** | - | 30 | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalité de l'envoi |
| `tax` | string | **oui** | - | - | `sender`, `receiver` | Qui paie les taxes |
| `insurance` | boolean | **oui** | - | - | - | Souscrire une assurance |
| `residential_delivery` | boolean | **oui** | - | - | - | Livraison à une adresse résidentielle |
| `non_stackable` | boolean | **oui** | - | - | - | Colis non empilables |
| `measurement` | string | **oui** | - | - | `metric`, `imperial` | Système de mesure |
| `ior` | boolean | **oui** | - | - | - | Importer of Record |
| `currency_quote` | string | **oui** | 3 | 3 | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise de la cotation |
| `currency_payment` | string | **oui** | 3 | 3 | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise du paiement |
| `address_sender` | object | **oui** | - | - | - | Expéditeur (voir tableau) |
| `address_receiver` | object | **oui** | - | - | - | Destinataire (voir tableau) |
| `boxes` | array | **oui** | 1 élément | - | - | Colis (voir tableau) |
| `payment` | object | **oui** | - | - | - | Paiement (voir tableau) |
| `items` | array | conditionnel | - | - | - | Obligatoire quand `object` n'est **pas** `doc` et qu'aucune boîte n'utilise `kit`. **Interdit quand `object` vaut `doc`** |
| `carrier_code` | integer | non | - | - | - | Fixe le contrat à utiliser |
| `slug` | string | non | - | 20 | - | Fixe le contrat par son identifiant textuel |
| `channel_code` | integer | non | - | - | - | Code du canal de vente enregistré |
| `channel_identifier_number` | string | non | - | - | - | Votre identifiant commercial. **Recommandé** |
| `nfkey` | string | non | 44 | 44 | - | Clé de la NF-e |
| `description` | string | non | 5 | 155 | - | Description libre du manifeste |
| `declared_freight` | object | non | - | - | - | Fret déclaré (voir tableau) |
| `biller` | object | non | - | - | - | Facturant affiché sur l'invoice |
| `manufacturer` | object | non | - | - | - | Fabricant affiché sur l'invoice |
| `extra` | object | non | - | - | - | Données auxiliaires en paires clé/valeur (voir tableau) |
| `format` | string | non | - | - | `pdf`, `zpl` | Alternative à la query string |
| `size` | string | non | - | - | `label`, `letter` | Taille de l'étiquette |

### Paramètres — `address_sender` et `address_receiver`

Contrairement à l'expédition v1/v2, ici l'adresse est indiquée **complète**, parce que la cotation a lieu
dans le même appel.

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code de l'adresse enregistrée. **N'envoyez aucun autre champ d'adresse avec lui** |

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `country_code` | string | **oui** | 2 | 2 | - | Pays en ISO 3166-1 alpha-2 |
| `city` | string | conditionnel | 1 | 100 | - | Ville. Obligatoire si `postal_code` n'est pas indiqué |
| `postal_code` | string | conditionnel | - | 12 | - | Code postal/ZIP. Obligatoire si `state` et `state_code` ne sont pas indiqués |
| `state` | string | conditionnel | - | 100 | - | Nom de l'état. Obligatoire si `state_code` et `postal_code` ne sont pas indiqués |
| `state_code` | string | conditionnel | 1 | 10 | - | Sigle de l'état, validé contre le pays. Obligatoire si `state` et `postal_code` ne sont pas indiqués |
| `street` | string | **oui** | 5 | 70 | - | Rue |
| `name` | string | **oui** | 1 | 70 | - | Nom du contact/raison sociale |
| `email` | string | **oui** | - | 60 | - | E-mail valide |
| `type` | string | **oui** | - | - | `pj` | `pf` ou `pj` |
| `foreign` | boolean | **oui** | - | - | - | Indique si la personne est étrangère (utilisé pour valider le `federal_tax_id`) |
| `federal_tax_id` | string | conditionnel | - | 20 | - | **Obligatoire quand le pays est `BR` ou `AR`** |
| `phone` | string | non | 1 | 25 | - | Téléphone |
| `phone_extension` | string | non | - | 5 | - | Poste |
| `number` | string | non | 1 | 6 | - | Numéro du bâtiment |
| `district` | string | non | 1 | 30 | - | Quartier |
| `complement` | string | non | 1 | 20 | - | Complément |
| `description` | string | non | 1 | 155 | - | Libellé interne de l'adresse |
| `state_tax_id` | string | non | - | 20 | - | Numéro d'immatriculation de l'état |
| `company_name` | string | non | 5 | 50 | - | **Uniquement dans `address_sender`.** Raison sociale exigée par certains envois FedEx |

### Paramètres — `boxes[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `kit` | string | alternative aux autres formes | - | 50 | SKU du kit enregistré. Remplit la boîte **et** les articles de cette position |
| `box_code` | integer | alternative | - | - | Code de la boîte enregistrée ([Boîte](api-caixa-v1.md)) |
| `box_identifier` | string | alternative | - | 50 | Identifiant de la boîte enregistrée |
| `name` | string | **oui** sans `kit`/`box_code`/`box_identifier` | 5 | 155 | Nom de la boîte |
| `height` | number | **oui** sans `kit`/`box_code`/`box_identifier` | 1 | - | Hauteur (cm en `metric`, in en `imperial`) |
| `width` | number | **oui** sans `kit`/`box_code`/`box_identifier` | 1 | - | Largeur |
| `depth` | number | **oui** sans `kit`/`box_code`/`box_identifier` | 1 | - | Profondeur |
| `weight` | number | **oui** quand `object` vaut `doc` | 0.001 | - | Poids du colis (kg en `metric`, lb en `imperial`) |
| `price` | number | **oui** quand `object` vaut `doc` | 0 | - | Valeur déclarée du colis |

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `box_id` | integer | **oui** | - | - | **Position** de la boîte dans le tableau `boxes` (1 = la première) |
| `quantity` | integer | **oui** | 1 | 99 | Quantité de l'article |
| `sku` | string | alternative aux champs manuels | 1 | 50 | SKU de l'article enregistré ([Articles](api-itens-v1.md)) |
| `name` | string | **oui** sans `sku` | 6 | 50 | Nom de l'article |
| `country_code` | string | **oui** sans `sku` | 2 | 2 | Pays d'origine de l'article |
| `hscode` | string | **oui** sans `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **oui** sans `sku` | 0.001 | 99 | Poids unitaire |
| `unit_value` | number | **oui** sans `sku` | 0 | - | Valeur unitaire |
| `description` | string | non | 5 | 450 | Description détaillée de l'article |
| `selling_unit_value` | number | conditionnel | 0 | - | Valeur de vente unitaire. **Nécessaire quand `ior` vaut `true`** |

> **Attention aux différences par rapport à la cotation :** ici `quantity` a un maximum de **99** (et non 1000) et
> `description` accepte jusqu'à **450** caractères (et non 155).

### Paramètres — `payment`

| Champ | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|
| `payment` | object | **oui** | - | Enveloppe |
| `payment.type` | string | **oui** | **`billed`** | Le manifeste n'accepte **que** le paiement facturé |

### Paramètres — `declared_freight`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `declared` | boolean | non | - | - | Active le fret déclaré. Avec `true`, la valeur peut être `0` et les taxes sont calculées avec un fret déclaré nul |
| `value` | number | non | 0 | - | Montant du fret déclaré |
| `currency` | string | **oui** quand `value` est indiqué | 3 | 3 | Devise du fret déclaré |

### Paramètres — `extra`

Paires clé/valeur auxiliaires, transmises au processus du manifeste.

| Règle | Détail |
|---|---|
| Structure | Objet **plat** (un seul niveau, sans imbrication) |
| Clés | Uniquement des lettres minuscules et `_` |
| Valeurs | Texte, entier ou décimal |

| Clé connue | Type | Règle | Description |
|---|---|---|---|
| `federal_tax_provision` | number | - | Provision de taxe fédérale |
| `state_tax_provision` | number | - | Provision de taxe de l'état |
| `usd_rate_provision` | number | > 0 | Provision de change du dollar |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Paramètres — `biller` et `manufacturer`

- `biller` : même format que celui décrit dans [Expédition (v1)](api-remessa-v1.md#paramètres--biller-facturant).
  Dans le manifeste, `biller.name` accepte jusqu'à **355** caractères.
- `manufacturer` : même format que celui décrit dans [Cotation (v1)](api-cotacao-v1.md#paramètres--manufacturer).

### Payload minimal

Manifeste d'un document (`object: doc`) — `items` est interdit dans ce cas :

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Enveloppe documents",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Payload maximal

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "slug": "fedex-br",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifeste du lot de mars",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centre de distribution SP",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Hangar 2",
    "company_name": "Boutique Exemple Commerce LTDA",
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
    "country_code": "US",
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
  "boxes": [
    {
      "name": "Boite master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton, manches courtes, couleur blanche, taille S",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Baskets de sport",
      "description": "Chaussures de course pour homme taille 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "number": "100",
    "name": "Holding Exemple LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Usine Exemple LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "type": "pj"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Exemple de requête

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rue Principale", "name": "Boutique Exemple LTDA", "email": "contato@example.com", "type": "pj", "federal_tax_id": "12345678000199", "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "type": "pf", "foreign": true},
    "boxes": [{"name": "Enveloppe documents", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Test",
    "tracking_id": "271249639538",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.shipping` | string (UUID) | **Identifiant de l'expédition.** À utiliser pour le suivi, les documents et la planification |
| `data.carrier` | string | Transporteur qui a traité l'envoi |
| `data.tracking_id` | string | Numéro de suivi (AWB) |
| `data.docs` | object | Documents générés, au format `nom → { url, base64 }` |

> Des documents supplémentaires (invoice, packing list, déclaration) peuvent être générés en arrière-plan.
> Consultez [Documents de l'Expédition](api-documentos.md) quelques instants plus tard pour obtenir la liste complète.

## Erreur du transporteur

Quand le transporteur refuse l'envoi, la réponse est `400` et contient son retour brut :

```json
{
  "status": "error",
  "message": "[1234] CARRIER - Carrier error message",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Champ | Description |
|---|---|
| `message` | `[code] TRANSPORTEUR - message` retourné par le transporteur |
| `data` | Payload brut du transporteur. **La structure varie selon le transporteur** |

> Enregistrez `message` et `data` dans vos logs : c'est la seule source du motif technique du refus.

## Règles métier
- **Paiement :** uniquement `billed`. Toute autre valeur est rejetée à la validation.
- **`items` et `object: doc` :** avec `object: doc`, le champ `items` **ne peut pas** être envoyé.
  Avec tout autre `object`, `items` est obligatoire — sauf lorsqu'une boîte utilise `kit`.
- **Kits :** quand une boîte indique `kit`, la boîte et les articles de cette position viennent de l'enregistrement,
  et `boxes[].weight`/`boxes[].price` sont ignorés.
- **`state_code` :** validé contre le `country_code` de la même adresse.
- **`federal_tax_id` :** obligatoire pour `BR` et `AR`.
- **`address_code` :** lorsqu'il est utilisé, **aucun** autre champ d'adresse ne peut être envoyé dans le même objet.
- **Les préréglages du compte** sont appliqués de la même façon que dans le brouillon : le payload gagne toujours sur le préréglage.
  Indiquer `declared_freight` annule le préréglage de fret déclaré ; indiquer
  `address_sender.country_code` annule le préréglage d'adresse de l'expéditeur.
- **`not_doc`** est accepté par compatibilité et converti en interne en `package`.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Erreur du transporteur ou règle métier | Lire `message` et `data` ; ajuster le payload ou contacter le support |
| `403` | Sans permission de créer un manifeste | Demander la permission au support |
| `422` | Validation des champs | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | **Avant de réessayer**, vérifiez dans [Suivi](api-rastreio.md) si l'expédition a été créée |

> ⚠️ **Ne répétez jamais un manifeste automatiquement après un `500`.** Comme l'appel crée réellement l'expédition,
> une répétition peut générer un envoi dupliqué. Vérifiez d'abord via le `channel_identifier_number`.

## Remarques finales
- Indiquez toujours `channel_identifier_number` : c'est votre protection contre les doublons et le moyen de localiser
  l'expédition si la réponse est perdue.
- Le manifeste concentre beaucoup de responsabilités dans un seul appel. Pour les intégrations qui peuvent attendre,
  le flux [cotation](api-cotacao-v1.md) + [expédition](api-remessa-v1.md) est plus sûr.
- Pour le flux inverse (payload partiel, validé plus tard), voir [Brouillon](api-rascunho.md).
