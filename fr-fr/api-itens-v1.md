# Articles

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre opération.

## Vue d'ensemble
- Gère l'enregistrement des produits que vous expédiez, avec poids, dimensions, HS Code, valeurs et identifiants (SKU, EAN et NSMP).
- Une fois l'article enregistré, la cotation peut le référencer uniquement par son `sku`, sans répéter toutes les données.
- C'est ce qui permet la [cotation par articles](api-cotacao-v2.md#cotation-darticles) (`type: items`, **exclusive à la v2**) et les regroupements automatiques de boîtes du compte.
- Toutes les opérations d'écriture (`POST`, `PUT`, `DELETE`) sont **asynchrones** et validées uniquement lors du traitement.
- La liste est **paginée (20 par page)** et reflète l'enregistrement en temps réel (sans cache).

> ⚠️ **Important :** `POST`, `PUT` et `DELETE` **ne valident pas les champs de manière synchrone**.
> L'API répond `200` en confirmant la réception et valide chaque article pendant le traitement.
> Les articles invalides sont **ignorés silencieusement** (sans erreur dans la réponse HTTP) et le motif est consigné dans le
> rapport d'importation des articles, dans le tableau de bord. Confirmez toujours le résultat avec `GET /v1/item`.

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

| Header | Obligatoire | Valeurs | Par défaut |
|---|---|---|---|
| `Authorization` | **oui** | `Bearer VOTRE_TOKEN` | - |
| `Content-Type` | oui en `POST`, `PUT` et `DELETE` | `application/json` | - |
| `lang` | non | `pt`, `en`, `es` | `pt` |

Chaque opération exige une permission spécifique dans votre profil (lister, enregistrer/mettre à jour et supprimer des articles).
Sans cette permission, la réponse est `403` avec `Has no permission!`.

## Flux d'utilisation
1. (Facultatif) Enregistrez les [boîtes](api-caixa-v1.md) et notez le `box_code` pour les lier aux articles.
2. Enregistrez les articles avec `POST /v1/item`.
3. Attendez le traitement asynchrone et vérifiez le rapport d'importation dans le tableau de bord.
4. Confirmez avec `GET /v1/item` que tous les SKU ont bien été intégrés.
5. Utilisez le `sku` dans les [cotations](api-cotacao-v1.md).
6. Mettez à jour avec `PUT /v1/item` et supprimez avec `DELETE /v1/item`.

## Concepts essentiels

### Pourquoi utiliser l'enregistrement des articles

**Sans enregistrement** — en répétant tout dans chaque cotation :
```json
"items": [
  { "box_id": 1, "name": "T-shirt coton P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Avec enregistrement** — en référençant le SKU :
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Avec `type: items` (**exclusif à la [Cotation v2](api-cotacao-v2.md#cotation-darticles)**), vous n'avez même pas besoin de placer l'article dans une boîte : la plateforme constitue les boîtes selon le
regroupement configuré sur le compte (voir [Comment utiliser les articles dans la cotation](#comment-utiliser-les-articles-dans-la-cotation)).

### Identifiants du produit

| Champ | Ce que c'est | Utilisation |
|---|---|---|
| `sku` | Votre code interne du produit | **Clé de l'enregistrement** : unique par compte, utilisée pour référencer, mettre à jour et supprimer |
| `ean` | Code-barres EAN/GTIN | Imprimé sur l'invoice et envoyé comme identifiant standard du fabricant dans les informations réglementaires |
| `nsmp` | *Non Standard Manufacturer Product Id* — identifiant de produit **non standardisé** du fabricant (ex. : le code de pièce/modèle du fabricant) | Imprimé sur l'invoice (ligne `Mfr:`) et envoyé dans les informations réglementaires exigées par certains transporteurs pour les destinations de l'**Union européenne** |

### Unités de mesure

Les dimensions et le poids de l'enregistrement sont toujours interprétés en **centimètres** et en **kilogrammes**.
La conversion du système `imperial` effectuée dans la cotation ne s'applique qu'aux mesures envoyées dans le payload de la cotation —
elle **n'est pas** appliquée aux données renseignées à partir de l'enregistrement. Enregistrez toujours en cm/kg.

### Types de données (le traitement est strict)

La validation asynchrone vérifie le **type JSON** de chaque champ. Les valeurs ayant un type incorrect entraînent l'abandon de l'article :

| Champ | À envoyer sous forme de | Exemple correct | Exemple ignoré |
|---|---|---|---|
| `sku`, `hscode`, `ean`, `nsmp`, `category` | texte (entre guillemets) | `"hscode": "610910"` | `"hscode": 610910` |
| `box_code` | nombre entier | `"box_code": 3` | `"box_code": "3"` ou `3.0` |
| `height`, `width`, `depth`, `weight`, `unit_value`, `selling_unit_value` | nombre | `"weight": 0.25` | `"weight": "0,25"` |
| `is_groupable`, `is_foldable` | booléen JSON | `"is_groupable": false` | `"is_groupable": "false"` (enregistré, mais traité comme `true` dans la cotation) |

## Endpoint : lister les articles

- **Méthode et route :** `GET /v1/item`
- **Quand l'utiliser :** pour consulter l'enregistrement, vérifier si l'importation asynchrone est terminée et contrôler les données d'un SKU.

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `items[skus][]` | query | array de string | non | - | Liste de SKU (correspondance exacte, sensible à la casse). **Vide ou absent, retourne tous les articles** |
| `page` | query | integer | non | `1` | Page de la liste |

- **Éléments par page :** 20 (fixe).
- Les SKU inexistants sont ignorés (ils ne génèrent pas d'erreur).
- Sans cache : les modifications traitées apparaissent immédiatement.

### Payload minimal
Sans paramètres (retourne tous les articles, page 1) :
```
GET /v1/item
```

### Payload maximal
```
GET /v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044&page=1
```

### Exemple de requête

```bash
# Tous les articles
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'

# Filtrage par SKU
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

> Le `--globoff` empêche curl d'interpréter les crochets de la query string.

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "sku": "TSH-000-S",
        "name": "T-shirt coton P",
        "description": "T-shirt 100% coton manches courtes blanc taille P",
        "category": "610910",
        "country_code": "BR",
        "hscode": "610910",
        "ean": "7894900010015",
        "nsmp": "FAB-TSH-P-001",
        "height": 2,
        "width": 20,
        "depth": 25,
        "weight": 0.25,
        "is_groupable": true,
        "is_foldable": true,
        "box_code": 3,
        "box_identifier": "CX0236",
        "unit_value": 29.9,
        "selling_unit_value": 49.9
      },
      {
        "sku": "SNK-044",
        "name": "Basket sport 44",
        "description": "Chaussure de course homme pointure 44 couleur noire",
        "category": "640411",
        "country_code": "BR",
        "hscode": "640411",
        "height": 15,
        "width": 22,
        "depth": 33,
        "weight": 0.8,
        "is_groupable": false,
        "nsmp": null,
        "unit_value": null,
        "selling_unit_value": null
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/item?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 20,
    "prev_page_url": null,
    "to": 2,
    "total": 2
  }
}
```

### Champs de la réponse

**Article (`data.data[]`) :**

| Champ | Type | Toujours présent | Description |
|---|---|---|---|
| `sku` | string | oui | Code unique du produit |
| `name` | string | oui | Nom du produit |
| `description` | string | oui | Description du produit |
| `category` | string | oui | Catégorie (6 caractères) |
| `country_code` | string | oui | Pays d'origine/de fabrication |
| `hscode` | string | oui | NCM/HS Code |
| `height`, `width`, `depth` | number | oui | Dimensions (cm) |
| `weight` | number | oui | Poids unitaire (kg) |
| `nsmp` | string \| null | **oui** | Identifiant non standardisé du fabricant. Vaut `null` lorsqu'il n'est pas enregistré |
| `unit_value` | number \| null | **oui** | Valeur unitaire par défaut. `null` lorsqu'elle n'est pas enregistrée |
| `selling_unit_value` | number \| null | **oui** | Valeur de vente unitaire. `null` lorsqu'elle n'est pas enregistrée |
| `ean` | string | non | Code EAN/GTIN (n'apparaît que s'il est enregistré) |
| `is_groupable` | boolean | non | Permet de regrouper avec d'autres articles (n'apparaît que s'il est enregistré) |
| `is_foldable` | boolean | non | Article pliable (n'apparaît que s'il est enregistré) |
| `box_code` | integer | non | Boîte par défaut liée (n'apparaît que si elle est enregistrée) |
| `box_identifier` | string | non | Boîte par défaut liée, par l'identifiant (n'apparaît que si elle est enregistrée) |

> Les champs sont renvoyés tels qu'ils ont été enregistrés (même type et même valeur que ceux envoyés). L'ordre des clés peut varier.
> Les champs hors de la liste documentée qui sont envoyés lors de l'enregistrement sont également enregistrés et peuvent apparaître dans la liste :
> **n'envoyez que les champs documentés**.

**Pagination (`data`) :** mêmes champs que la [liste des boîtes](api-caixa-v1.md#champs-de-la-réponse)
(`current_page`, `from`, `to`, `per_page` = 20, `total`, `last_page`, URL de navigation, `path` et `links`).

## Endpoint : créer des articles

- **Méthode et route :** `POST /v1/item`
- **Quand l'utiliser :** pour enregistrer de nouveaux produits, un ou plusieurs par requête.
- **Traitement :** asynchrone, par lot. Chaque article est validé individuellement ; les articles valides sont enregistrés et les invalides
  sont ignorés et listés dans le rapport d'importation.
- **Un SKU déjà existant n'est pas mis à jour :** utilisez `PUT` pour le modifier.

### Paramètres

**Racine du payload :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `items` | array | **oui** | Liste des articles. Sans `items`, rien n'est enregistré (la réponse reste `200`) |

**Paramètres — `items[]` :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `sku` | string | **oui** | 1 | 50 (recommandé) | - | Code unique du produit sur votre compte. **Ne peut pas être répété.** La cotation accepte des SKU de 50 caractères maximum |
| `name` | string | **oui** | 1 | - | - | Nom du produit (apparaît sur l'invoice) |
| `description` | string | **oui** | 1 (6 recommandé) | 155 (recommandé) | - | Description du produit. **Des descriptions distinctes entre les articles évitent les alertes douanières** |
| `category` | string | **oui** | 6 | 6 | - | Catégorie du produit. **Exactement 6 caractères** (sans accents : les lettres accentuées comptent pour 2) |
| `country_code` | string | **oui** | 2 | 2 | - | Pays d'origine/de fabrication en ISO 3166-1 alpha-2, en majuscules (ex. : `BR`) |
| `hscode` | string | **oui** | 1 | - | - | NCM/HS Code, **sous forme de texte**. Définit la table de taxes appliquée à destination |
| `height` | number | **oui** | 0.01 | - | - | Hauteur du produit (cm) |
| `width` | number | **oui** | 0.01 | - | - | Largeur du produit (cm) |
| `depth` | number | **oui** | 0.01 | - | - | Profondeur du produit (cm) |
| `weight` | number | **oui** | 0.01 | - | - | Poids unitaire (kg) |
| `is_groupable` | boolean | non (**recommandé**) | - | - | `true` | `true` permet de regrouper l'article avec d'autres dans la même boîte ; `false` exige une boîte exclusive. Dans le regroupement `default`, un article sans ce champ provoque une erreur dans la cotation |
| `ean` | string | non | - | 50 (recommandé) | - | Code EAN/GTIN, **sous forme de texte** |
| `nsmp` | string | non | - | 50 (recommandé) | `null` | *Non Standard Manufacturer Product Id* : identifiant non standardisé du fabricant (ex. : code de modèle/pièce). Imprimé sur l'invoice et utilisé dans les informations réglementaires de certains transporteurs pour les destinations de l'Union européenne. La cotation accepte 50 caractères maximum |
| `unit_value` | number | non | - | - | - | Valeur unitaire par défaut. Si elle est absente de l'enregistrement **et** de la cotation, la cotation échoue |
| `selling_unit_value` | number | non | - | - | - | Valeur de vente unitaire. Nécessaire lorsque la cotation utilise `ior: true` |
| `box_code` | integer | non | - | - | - | Boîte par défaut de l'article ([Boîte](api-caixa-v1.md)). Doit être un **entier** JSON |
| `box_identifier` | string | non | - | 50 | - | Boîte par défaut de l'article, par l'identifiant textuel |
| `is_foldable` | boolean | non | - | - | `false` | Article pliable. Dans le `smartbox`, les articles pliables sont placés selon leur volume ; les non pliables, selon leurs dimensions |

> **Boîte par défaut (Forme A / Forme B) :** liez la boîte par `box_code` (Forme A) **ou** par `box_identifier` (Forme B).
> N'envoyez pas les deux avec des valeurs correspondant à des boîtes différentes. L'existence de la boîte **n'est pas** vérifiée lors de l'enregistrement de l'article —
> elle n'est vérifiée que dans la cotation. Préférez `box_code` : c'est le lien reconnu par la protection contre la suppression des boîtes.

> Les valeurs `null` dans `unit_value` et `selling_unit_value` sont ignorées (le champ n'est pas enregistré).

### Payload minimal

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt coton P",
      "description": "T-shirt 100% coton manches courtes blanc taille P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25
    }
  ]
}
```

> C'est le plus petit payload accepté. Pour utiliser l'article dans la cotation `type: items` avec le regroupement `default`, incluez également
> `is_groupable` et la boîte par défaut (`box_code` ou `box_identifier`).

### Payload maximal

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt coton P",
      "description": "T-shirt 100% coton manches courtes blanc taille P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-001",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Basket sport 44",
      "description": "Chaussure de course homme pointure 44 couleur noire",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": "FAB-SNK-44-BLK",
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

### Exemple de requête

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "T-shirt coton P", "description": "T-shirt 100% coton manches courtes blanc taille P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-001", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true, "box_code": 3}
    ]
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` (reçu) ou `error` (échec inattendu) |
| `message` | string | Message traduit selon le header `lang` |

> Le `200` confirme uniquement la réception. Les articles ignorés apparaissent dans le rapport d'importation (voir
> [Validation asynchrone](#validation-asynchrone--tableau-de-référence)).

## Endpoint : mettre à jour des articles

- **Méthode et route :** `PUT /v1/item`
- **Quand l'utiliser :** pour modifier les données de produits déjà enregistrés, par lot.
- **Clé de mise à jour :** le `sku`. L'article enregistré avec ce SKU reçoit les valeurs envoyées.
- **Traitement :** asynchrone, par lot, avec **la même validation que le `POST`**.

> ⚠️ **Envoyez l'article complet.** La validation du `PUT` est identique à celle du `POST` : `sku`, `name`, `description`, `category`,
> `country_code`, `hscode`, `height`, `width`, `depth` et `weight` sont **obligatoires à chaque mise à jour**. Un article
> envoyé uniquement avec le SKU et le champ modifié est **ignoré** (motif `name` dans le rapport).

> ⚠️ **Un SKU inexistant n'est pas créé.** La mise à jour ne fait pas d'« upsert » : si le SKU n'existe pas, rien ne se passe
> (et rien n'est signalé). Utilisez `POST` pour créer.

Règles de mise à jour des champs facultatifs :

| Situation dans le payload du `PUT` | Résultat dans l'enregistrement |
|---|---|
| Champ facultatif **omis** | Conserve la valeur actuelle |
| Champ facultatif envoyé avec une valeur | Remplace la valeur actuelle |
| `ean`, `nsmp`, `box_code`, `box_identifier`, `is_foldable` envoyés avec `null` | La valeur est effacée (passe à `null`) |
| `unit_value` ou `selling_unit_value` envoyés avec `null` | **Ignoré** : la valeur actuelle est conservée (il n'est pas possible d'effacer le prix via le `PUT` ; envoyez une nouvelle valeur) |

### Paramètres

Même structure et mêmes règles que le [`POST /v1/item`](#endpoint--créer-des-articles) : racine `items` (array) et les champs de `items[]`.

### Payload minimal

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt coton P",
      "description": "T-shirt premium 100% coton manches courtes blanc P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27
    }
  ]
}
```

### Payload maximal

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt coton P",
      "description": "T-shirt premium 100% coton manches courtes blanc P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-002",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27,
      "unit_value": 34.9,
      "selling_unit_value": 54.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Basket sport 44",
      "description": "Chaussure de course homme pointure 44 couleur noire",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": null,
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 189.9,
      "selling_unit_value": 339.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

> Dans le second article, `"nsmp": null` efface le NSMP enregistré.

### Exemple de requête

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "T-shirt coton P", "description": "T-shirt premium 100% coton manches courtes blanc P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-002", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` (reçu) ou `error` (échec inattendu) |
| `message` | string | Message traduit selon le header `lang` |

## Endpoint : supprimer des articles

- **Méthode et route :** `DELETE /v1/item`
- **Quand l'utiliser :** pour supprimer définitivement des produits de l'enregistrement.
- **Traitement :** asynchrone, par lot. Les SKU inexistants sont ignorés sans avertissement.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Min. | Description |
|---|---|---|---|---|---|
| `items` | body | object | **oui** | - | Enveloppe |
| `items.skus` | body | array de string | **oui** | 1 élément | Liste des SKU à supprimer. Chaque SKU doit être un texte non vide |

> L'absence de `items.skus` ne génère pas d'erreur dans la réponse HTTP (`200`) ; le problème est consigné dans le rapport.

### Payload minimal

```json
{
  "items": {
    "skus": ["TSH-000-S"]
  }
}
```

### Payload maximal

```json
{
  "items": {
    "skus": ["TSH-000-S", "TSH-000-M", "SNK-044"]
  }
}
```

### Exemple de requête

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` (reçu) ou `error` (échec inattendu) |
| `message` | string | Message traduit selon le header `lang` |

## Validation asynchrone — tableau de référence

Comme la validation a lieu pendant le traitement (`POST` et `PUT`), utilisez ce tableau pour garantir que les articles soient intégrés.
Les règles sont vérifiées **dans cet ordre** et la **première** règle enfreinte entraîne l'abandon de l'article.
Dans le rapport d'importation, chaque article ignoré apparaît sous la forme :

```
sku => TSH-000-S, status => invalid, reason => category
```

| Ordre | Champ | Règle | `reason` dans le rapport |
|---|---|---|---|
| 1 | (article) | Doit être un objet | `not_array` |
| 2 | `name` | Obligatoire, texte non vide | `name` |
| 3 | `description` | Obligatoire, texte non vide | `description` |
| 4 | `category` | Obligatoire, texte, **exactement 6 caractères** | `category` |
| 5 | `country_code` | Obligatoire, texte, **exactement 2 caractères** | `country_code` |
| 6 | `sku` | Obligatoire, texte non vide | `sku` |
| 7 | `ean` | S'il est envoyé (non nul), doit être un texte | `ean` |
| 8 | `nsmp` | S'il est envoyé (non nul), doit être un texte | `nsmp` |
| 9 | `height` | Obligatoire, numérique, **≥ 0.01** | `height` |
| 10 | `width` | Obligatoire, numérique, **≥ 0.01** | `width` |
| 11 | `depth` | Obligatoire, numérique, **≥ 0.01** | `depth` |
| 12 | `weight` | Obligatoire, numérique, **≥ 0.01** | `weight` |
| 13 | `hscode` | Obligatoire, texte non vide | `hscode` |
| 14 | `box_code` | S'il est envoyé (non nul), doit être un **entier** (ni string, ni décimal) | `box_code` |
| 15 | `box_identifier` | S'il est envoyé (non nul), doit être un texte | `box_identifier` |
| 16 | `unit_value` | S'il est envoyé (non nul), doit être numérique | `unit_value` |
| 17 | `selling_unit_value` | S'il est envoyé (non nul), doit être numérique | `selling_unit_value` |

Règles appliquées silencieusement (sans abandon et sans avertissement) :

| Situation | Comportement |
|---|---|
| `is_groupable` avec n'importe quelle valeur | Accepté et enregistré tel qu'envoyé. Utilisez un booléen JSON : `"false"` sous forme de texte est traité comme `true` dans la cotation |
| `is_foldable` avec n'importe quelle valeur | Non validé ; enregistré tel qu'envoyé |
| `name`, `description`, `sku`, `ean`, `nsmp` au-delà des limites recommandées | Enregistrés, mais peuvent générer une erreur lorsque l'article est utilisé dans la cotation/l'expédition |
| `country_code` de 2 caractères qui n'est pas un pays valide | Enregistré ; l'erreur n'apparaît que dans la cotation |
| `box_code`/`box_identifier` d'une boîte inexistante | Enregistré ; l'erreur n'apparaît que dans la cotation |
| Champs non documentés | Enregistrés tels qu'envoyés |

Autres situations consignées dans le rapport :

| Opération | Situation | Comportement |
|---|---|---|
| `POST`/`PUT` | Corps vide (`{}`) | Rapport : `Itens isn't an array or is empty` |
| `POST`/`PUT` | `items` absent avec d'autres champs dans le corps, ou `items` qui n'est pas une liste | Le traitement échoue et est consigné comme échec dans le rapport ; rien n'est enregistré |
| `POST`/`PUT` | `items: []` | Rien n'est enregistré et rien n'est signalé |
| `POST` | SKU déjà enregistré, ou répété dans le même lot | L'article dupliqué est rejeté ; les autres articles valides du lot sont enregistrés et le lot est marqué comme échec dans le rapport |
| `PUT` | SKU inexistant | Rien n'est modifié, rien n'est créé et rien n'est signalé |
| `DELETE` | `items.skus` absent ou vide | Rapport : `SKUs isn't an array or is empty` |
| `DELETE` | SKU vide ou qui n'est pas un texte (ex. : un nombre) | Rapport : `sku => ..., status => invalid, reason => sku_invalid` |
| `DELETE` | SKU inexistant | Ignoré sans avertissement |

## Comment utiliser les articles dans la cotation

| Cotation `type` | Payload dans `items[]` | Comportement |
|---|---|---|
| `advanced` (v1 et v2) | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Vous définissez dans quelle boîte (position dans `boxes[]`) va l'article |
| `items` (**uniquement v2**) | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | La plateforme constitue les boîtes selon le regroupement du compte |
| n'importe lequel | `{ "box_id": 1, "name": "T-shirt coton P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Article ponctuel, sans enregistrement |

Lorsque l'article de la cotation a un `sku` et qu'il manque une donnée (nom, pays, HS Code, dimensions, poids, valeurs ou `nsmp`),
les données manquantes sont complétées à partir de l'enregistrement. Les champs envoyés dans la cotation ont priorité sur l'enregistrement.

Utilisation de l'enregistrement selon le regroupement du compte (cotation `type: items`, v2) :

| Regroupement | Prérequis de l'enregistrement | `is_groupable` |
|---|---|---|
| `default` | Chaque article doit avoir une boîte par défaut (`box_code` ou `box_identifier`) et `is_groupable` enregistrés | `true` : regroupé avec d'autres articles de la même boîte par défaut. `false` : une boîte exclusive par unité |
| `smartbox` | Les boîtes candidates proviennent de `boxes[]` dans le payload de la cotation | Absent = `true`. `false` : boîte exclusive. `is_foldable` (absent = `false`) définit le placement selon le volume ou selon les dimensions |
| `all_in_one` | Uniquement les données de l'article | Absent = `true`. `false` : l'article reste hors de la consolidation, dans sa propre boîte |

Voir la section `items[]` dans [Cotation (v1)](api-cotacao-v1.md#paramètres--items).

## Règles métier
- Toutes les opérations d'écriture s'exécutent en **arrière-plan** ; le `200` confirme uniquement la réception.
- Le `sku` est **unique** par compte : le `POST` n'écrase pas un SKU existant.
- Les articles sont **par compte** : chaque utilisateur ne voit et ne modifie que les siens.
- Le `PUT` exige l'article complet (même validation que le `POST`) et ne crée pas de nouveaux SKU.
- Les dimensions et le poids de l'enregistrement sont toujours en **cm** et en **kg**.
- `nsmp` est facultatif ; lorsqu'il est renseigné, il est imprimé sur l'invoice et envoyé dans les informations réglementaires de certains
  transporteurs pour les destinations de l'Union européenne. Sans `nsmp`, cette donnée reste vide.
- L'enregistrement doit être complet pour que la cotation fonctionne. Messages associés dans la cotation :

| Statut | Message | Cause |
|---|---|---|
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | SKU non enregistré et article sans les données minimales dans le payload |
| `404` | `O item com o sku :name não existe!` | SKU non enregistré dans la cotation `type: items` |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Article sans boîte par défaut valide dans le regroupement `default` |
| `400` | `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).` | Pas de `unit_value` dans l'enregistrement ni dans la cotation |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | Valeur à zéro |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | `hscode` incorrect pour la destination |

> Les codes HTTP des messages de cotation ci-dessus suivent la page [Cotation (v1)](api-cotacao-v1.md#erreurs-courantes).

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `200` avec des articles manquants | Article ignoré lors de la validation asynchrone (`reason => champ` dans le rapport) | Vérifier le rapport dans le tableau de bord et le [tableau de validation](#validation-asynchrone--tableau-de-référence) |
| `200` sans effet dans le `PUT` | SKU inexistant ou article incomplet | Créer avec `POST` ou envoyer l'article complet |
| `403` | `Has no permission!` — profil sans permission pour lister, enregistrer/mettre à jour ou supprimer des articles | Demander la permission au support |
| `422` | `O campo items.skus deve conter um array.` (uniquement `GET`) | Envoyer `items[skus][]` sous forme de liste dans la query string |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Réessayer avec un intervalle croissant ; si le problème persiste, contacter le support |
| Rapport | `Itens isn't an array or is empty` | Envoyer `items` avec au moins un article |
| Rapport | `SKUs isn't an array or is empty` | Dans le `DELETE`, envoyer `items.skus` avec au moins un SKU |
| Rapport | `sku => ..., status => invalid, reason => sku_invalid` | Dans le `DELETE`, envoyer les SKU sous forme de texte non vide |

> `POST`, `PUT` et `DELETE` **ne retournent pas `422`** : les problèmes de champs n'apparaissent que dans le rapport d'importation.

## Remarques finales
- Comme la validation n'est pas synchrone, **confirmez toujours avec `GET /v1/item`** après avoir enregistré, mis à jour ou supprimé.
- Envoyez les types JSON corrects : `sku`, `hscode`, `ean` et `nsmp` sous forme de texte ; `box_code` sous forme d'entier ; les booléens sans guillemets.
- Conservez des descriptions distinctes pour des produits différents : des descriptions répétées génèrent l'avertissement
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` dans la cotation.
- Renseignez `nsmp` pour les produits envoyés vers l'Union européenne chaque fois que le fabricant dispose d'un code propre au produit.
- Pour standardiser les emballages utilisés avec ces articles, voir [Boîte](api-caixa-v1.md).
- Pour utiliser les articles dans la cotation, voir [Cotation (v1)](api-cotacao-v1.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
