# Brouillon

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Permet d'envoyer un envoi **incomplet** : vous transmettez ce que vous avez déjà, et la plateforme valide ensuite.
- Le brouillon est traité de façon **asynchrone** et, lorsqu'il est complet, évolue vers cotation + expédition
  automatiquement.
- C'est le flux indiqué quand l'intégration reçoit les données de la commande par étapes (ex. : la commande arrive avant
  l'adresse finale ou la facture fiscale).

> **Pratiquement tous les champs sont optionnels à l'envoi.** La validation stricte a lieu au moment de
> l'expédition. Tant que des données manquent, le brouillon reste au statut `incomplete` et vous pouvez le compléter.

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

## Brouillon vs. autres flux

| Thème | Cotation + Expédition | Manifeste | Brouillon |
|---|---|---|---|
| Payload | Complet en 2 étapes | Complet en 1 appel | **Partiel, en autant d'étapes que nécessaire** |
| Validation | Synchrone, par étape | Synchrone, totale | **Asynchrone, à l'expédition** |
| Réponse | Cotation avec les prix | Documents prêts | Confirmation de réception |
| Paiement | `billed`, `gateway`, `afterward` | `billed` | **`billed` ou `afterward`** |
| Correction ultérieure | Nouvelle cotation | Non | **Oui, via `PUT`** |

## Cycle de vie du brouillon

```
POST /v1/draft
     │
     ▼
  received  ──►  validating  ──►  incomplete  ──┐
                      │                          │ PUT /v1/draft/{code}
                      ▼                          │
                   complete  ◄───────────────────┘
                      │
                      ▼
                 processing  ──►  cotation + expédition
                      │
                      └──►  error
```

| Statut | Signification | Modifiable ? |
|---|---|---|
| `received` | Brouillon reçu, en attente de validation | Non |
| `validating` | En cours de validation | Non |
| `incomplete` | **Des données manquent.** Le motif est dans `status_descricao` | **Oui** |
| `edited` | Brouillon mis à jour via `PUT`, en attente d'une nouvelle validation | Non |
| `complete` | Toutes les données sont présentes, prêt pour l'expédition | Non |
| `processing` | En traitement : génération de la cotation et de l'expédition | Non |
| `error` | Échec du traitement | Non |

> **Seuls les brouillons au statut `incomplete` peuvent être modifiés.** Tenter de modifier dans un autre statut
> retourne `400 Apenas rascunhos com o status incomplete podem ser editados`.

## Flux d'utilisation
1. `POST /v1/draft` avec les données que vous avez déjà.
2. Attendez le traitement asynchrone.
3. `GET /v1/draft` pour voir le statut et, si `incomplete`, lire le `status_descricao`.
4. `PUT /v1/draft/{code}` pour compléter ce qui manquait.
5. Lorsqu'il devient `complete`, la plateforme génère la cotation et l'expédition.
6. Suivez via le [suivi](api-rastreio.md) et téléchargez les [documents](api-documentos.md).
7. `DELETE /v1/draft` pour supprimer les brouillons qui ne seront pas utilisés.

## Endpoint : créer un brouillon

- **Méthode et route :** `POST /v1/draft`
- **Traitement :** asynchrone.

### Paramètres — racine du payload

Aucun champ n'est obligatoire, mais **au moins un** doit être envoyé.

| Champ | Type | Min. | Max. | Par défaut | Valeurs | Description |
|---|---|---|---|---|---|---|
| `object` | string | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` | Ce qui est envoyé |
| `type` | string | - | 30 | `Commercial` | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalité de l'envoi |
| `tax` | string | - | - | préréglage du compte | `sender`, `receiver` | Qui paie les taxes |
| `insurance` | boolean | - | - | préréglage du compte | - | Souscrire une assurance |
| `measurement` | string | - | - | `metric` | `metric`, `imperial` | Système de mesure |
| `ior` | boolean | - | - | `false` | - | Importer of Record |
| `residential_delivery` | boolean | - | - | `false` | - | Livraison résidentielle |
| `non_stackable` | boolean | - | - | `false` | - | Colis non empilables |
| `currency_quote` | string | 3 | 3 | préréglage du compte | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise de la cotation |
| `currency_payment` | string | 3 | 3 | préréglage du compte | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise du paiement |
| `nfkey` | string | 44 | 44 | - | - | Clé de la NF-e |
| `description` | string | - | 155 | - | - | Description libre du brouillon |
| `channel_code` | integer | - | - | préréglage du compte | - | Code du canal de vente enregistré |
| `channel_identifier_number` | string | - | - | - | - | Votre identifiant commercial. **Recommandé** |
| `carrier_code` | integer | - | - | - | - | Contrat préféré, s'il est disponible dans la cotation |
| `address_sender` | object | - | - | préréglage du compte | - | Expéditeur (voir tableau) |
| `address_receiver` | object | - | - | - | - | Destinataire (voir tableau) |
| `boxes` | array | - | - | - | - | Colis (voir tableau) |
| `items` | array | - | - | - | - | Articles (voir tableau) |
| `payment` | object | - | - | - | - | Paiement (voir tableau) |
| `biller` | object | - | - | préréglage du compte | - | Facturant affiché sur l'invoice |
| `manufacturer` | object | - | - | - | - | Fabricant affiché sur l'invoice |

### Paramètres — `address_sender` et `address_receiver`

Tous optionnels à l'envoi. Les colonnes « Obligatoire à l'expédition » indiquent ce que la plateforme exige pour que le
brouillon devienne `complete`.

| Champ | Type | Min. | Max. | Obligatoire à l'expédition | Description |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Code d'adresse enregistrée (uniquement dans `address_sender`) |
| `country_code` | string | 2 | 2 | **Oui** | Pays en ISO 3166-1 alpha-2 |
| `city` | string | - | 100 | **Oui** | Ville |
| `postal_code` | string | - | 12 | Oui, si `state` n'est pas indiqué | Code postal/ZIP |
| `state` | string | - | - | Oui, si `state_code`/`postal_code` ne sont pas indiqués | Nom de l'état |
| `state_code` | string | - | - | Oui, si `state`/`postal_code` ne sont pas indiqués | Sigle de l'état, validé contre le pays |
| `street` | string | - | 70 | **Oui** | Rue |
| `name` | string | 5 | 35 | **Oui** | Nom du contact/raison sociale |
| `email` | string | - | 60 | **Oui** | E-mail valide |
| `phone` | string | - | 25 | **Oui** | Téléphone |
| `type` | string | - | - | **Oui** | `pf` ou `pj` (par défaut `pj`) |
| `federal_tax_id` | string | - | 20 | Oui quand le pays est `BR` | CPF/CNPJ ou document fédéral |
| `state_tax_id` | string | - | 20 | Oui quand le pays est `BR` | Numéro d'immatriculation de l'état |
| `number` | string | - | 100 | - | Numéro du bâtiment |
| `district` | string | - | 30 | - | Quartier |
| `complement` | string | - | 20 | - | Complément |
| `description` | string | - | 155 | - | Libellé interne de l'adresse |
| `phone_extension` | string | - | 5 | - | Poste |
| `foreign` | boolean | - | - | - | Indique si la personne est étrangère |

### Paramètres — `boxes[]`

| Champ | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU du kit enregistré. Remplit la boîte **et** les articles de cette position à l'expédition |
| `box_code` | integer | - | - | Code de la boîte enregistrée ([Boîte](api-caixa-v1.md)) |
| `box_identifier` | string | - | - | Identifiant de la boîte enregistrée |
| `name` | string | 4 | 155 | Nom de la boîte |
| `height` | number | 1 | - | Hauteur |
| `width` | number | 1 | - | Largeur |
| `depth` | number | 1 | - | Profondeur |
| `volume` | number | 0.001 | - | Volume (alternative aux dimensions, pour `pallet`) |
| `weight` | number | 0.1 | - | Poids du colis. **Obligatoire quand `object` vaut `doc`.** Jusqu'à 3 décimales. Ignoré quand la boîte utilise `kit` |
| `price` | number | 0 | - | Valeur déclarée. **Obligatoire quand `object` vaut `doc`.** Ignoré quand la boîte utilise `kit` |

### Paramètres — `items[]`

| Champ | Type | Min. | Max. | Obligatoire à l'expédition | Description |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU de l'article enregistré ([Articles](api-itens-v1.md)) |
| `ean` | string | - | 50 | - | Code EAN |
| `box_id` | integer | - | - | - | **Position** de la boîte dans le tableau `boxes` (1 = la première) |
| `name` | string | 6 | 35 | Oui, sans `sku` | Nom de l'article |
| `description` | string | 4 | 155 | - | Description de l'article |
| `country_code` | string | 2 | 2 | Oui, sans `sku` | Pays d'origine de l'article |
| `hscode` | string | 6 | 10 | Oui, sans `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Oui, sans `sku` | Poids unitaire |
| `quantity` | integer | 1 | 1000 | **Oui** | Quantité |
| `unit_value` | number | - | - | **Oui** | Valeur unitaire |
| `selling_unit_value` | number | - | - | Oui quand `ior` vaut `true` | Valeur de vente unitaire |
| `is_groupable` | boolean | - | - | - | Permet de regrouper l'article avec d'autres |
| `height`, `width`, `depth` | number | 1 | - | - | Dimensions de l'article |
| `is_foldable` | boolean | - | - | - | Article pliable (utilisé dans le regroupement `smartbox`) |

> `items` devient **obligatoire à l'expédition** quand `object` vaut `package`, `pallet` ou `not_doc`.

### Paramètres — `payment`

| Champ | Type | Obligatoire à l'expédition | Valeurs | Description |
|---|---|---|---|---|
| `payment.type` | string | **Oui** | `billed`, `afterward` | Le brouillon **n'accepte pas** `gateway` |

### Payload minimal

Le plus petit payload utile — n'importe quel champ convient, mais le brouillon doit en avoir au moins un :

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package"
}
```

Un exemple plus réaliste, avec ce que l'on a normalement déjà au moment de la commande :

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
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
  ],
  "payment": { "type": "billed" }
}
```

### Payload maximal

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "carrier_code": 1,
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
  "description": "Brouillon de la commande du marketplace",
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
    "state_tax_id": "987654321",
    "foreign": true
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
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton manches courtes blanc",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
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
  "payment": { "type": "billed" }
}
```

### Exemple de requête

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt coton", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

| Statut | Message | Cause |
|---|---|---|
| `400` | `Necessário informar pelo menos um parâmetro para criar o rascunho` | Payload vide |

## Endpoint : lister les brouillons

- **Méthode et route :** `GET /v1/draft`
- **Éléments par page :** 20

| Champ | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `drafts` | body | object | non | Enveloppe du filtre |
| `drafts.drafts_ids` | body | array | non | Liste de codes de brouillon. **Vide ou absente, tous sont retournés** |
| `page` | query | integer | non | Page de la liste |

> Ce `GET` accepte un corps JSON pour le filtre. Si votre client HTTP ne prend pas en charge le body en `GET`,
> listez tout et filtrez de votre côté.

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "object": "package",
          "type": "Commercial",
          "tax": "sender",
          "insurance": true,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "payment": { "type": "billed" },
          "address_receiver": {
            "country_code": "US",
            "postal_code": "33101",
            "city": "Miami",
            "street": "Main Avenue",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf"
          },
          "items": [
            {
              "name": "T-shirt coton",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9,
              "box_id": 1
            }
          ]
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender.country code é obrigatório. O campo address sender.street é obrigatório.",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `codigo` | integer | **Code du brouillon.** À utiliser dans `PUT /v1/draft/{code}` et dans `drafts.drafts_ids` |
| `data` | object | Le payload du brouillon, déjà fusionné avec les préréglages du compte |
| `status` | string | Statut actuel (voir cycle de vie) |
| `status_descricao` | string | **Quand il est `incomplete`, liste exactement ce qui manque.** C'est le champ le plus important pour corriger le brouillon |
| `created_at`, `updated_at` | string | Dates de création et de dernière modification |

## Endpoint : mettre à jour un brouillon

- **Méthode et route :** `PUT /v1/draft/{code}`
- **Restriction :** uniquement les brouillons au statut `incomplete`.
- **Structure du payload :** identique au `POST`.

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `code` | path | integer | **oui** | Code du brouillon (`codigo` dans la liste) |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "address_sender": {
      "country_code": "BR",
      "postal_code": "09941070",
      "city": "Diadema",
      "street": "Rue Principale",
      "name": "Boutique Exemple LTDA",
      "email": "contato@example.com",
      "phone": "11989677465",
      "type": "pj",
      "federal_tax_id": "12345678000199",
      "state_tax_id": "123456789012"
    }
  }'
```

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso."
}
```

| Statut | Message | Cause |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | Le brouillon est dans un autre statut |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | `code` invalide ou appartenant à un autre compte |

## Endpoint : supprimer des brouillons

- **Méthode et route :** `DELETE /v1/draft`

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `drafts` | object | **oui** | Enveloppe |
| `drafts.drafts_ids` | array | **oui** | Liste des codes de brouillon à supprimer |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Seuls les brouillons de votre propre compte sont supprimés. Si aucun code ne correspond, la réponse est
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Règles métier

### Validation à l'expédition
Quand le brouillon est expédié, la plateforme applique un ensemble **strict** de règles. Ce qui manque apparaît
dans `status_descricao`. Résumé des champs exigés :

| Groupe | Champs exigés |
|---|---|
| Général | `tax`, `insurance` |
| Expéditeur | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`state`, `state_code` ou `postal_code`) |
| Expéditeur BR | `federal_tax_id` et `state_tax_id` |
| Destinataire | Les mêmes champs que l'expéditeur |
| Articles (`object` ≠ `doc`) | `items` avec `quantity` et `unit_value` ; et `name`, `country_code`, `hscode`, `weight` quand il n'y a pas de `sku` |
| Boîtes (`object` = `doc`) | `boxes[].weight` et `boxes[].price` |
| Paiement | `payment.type` (`billed` ou `afterward`) |

### Fabricant
Si vous envoyez `manufacturer` **sans** `address_code`, les champs `name`, `country_code`, `state_code`,
`postal_code`, `city`, `street`, `email`, `phone` et `type` deviennent obligatoires à l'expédition.

Pour les articles dont le `hscode` commence par **`64`** (chaussures), le `manufacturer.name` est obligatoire.

### Préréglages du compte
Les préréglages sont fusionnés avec le brouillon. **Le payload gagne toujours sur le préréglage**, avec ces exceptions :
- Indiquer `address_sender.country_code` annule le préréglage d'adresse de l'expéditeur.
- Si le compte a la taxe verrouillée (`tax locked`), la valeur est forcée à `sender`.
- Le préréglage de facturant n'est appliqué que lorsque vous envoyez un `biller` vide.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Brouillon vide, non trouvé ou non modifiable | Envoyer au moins un champ, vérifier le `code` et le statut |
| `403` | Sans permission pour l'action | Demander la permission au support |
| `422` | Validation des champs | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Le champ **`status_descricao` est votre guide** : il indique exactement ce qui manque pour que le brouillon avance.
- Indiquez toujours `channel_identifier_number` : c'est ce qui relie le brouillon à votre commande et permet de
  consulter l'expédition plus tard.
- Le brouillon **n'accepte pas** le paiement via `gateway`. Pour ce scénario, utilisez
  [cotation](api-cotacao-v1.md) + [expédition](api-remessa-v1.md).
- Pour le flux inverse (tout validé d'un coup, documents immédiats), voir [Manifeste](api-manifesto.md).
