# Articles

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Gère l'enregistrement des produits que vous expédiez, avec le poids, les dimensions, le HS Code et les valeurs.
- Avec l'article enregistré, la cotation peut le référencer uniquement par le `sku`, sans répéter toutes les données.
- C'est ce qui active la cotation `type: items` et le regroupement automatique `smartbox`.
- Toutes les opérations d'écriture sont **asynchrones**.

> ⚠️ **Important :** `POST`, `PUT` et `DELETE` **ne valident pas les champs de façon synchrone**.
> L'API répond `200` en confirmant la réception et valide chaque article pendant le traitement.
> Les articles invalides sont **ignorés silencieusement** et le motif est consigné dans le rapport
> d'importation, dans le portail. Confirmez toujours le résultat avec `GET /v1/item`.

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

## Pourquoi utiliser l'enregistrement des articles

**Sans enregistrement** — en répétant tout dans chaque cotation :
```json
"items": [
  { "box_id": 1, "name": "T-shirt coton S", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Avec enregistrement** — en référençant le SKU :
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Avec `type: items`, vous n'avez même pas besoin d'indiquer la boîte : la plateforme choisit parmi les
[boîtes enregistrées](api-caixa-v1.md).

## Flux d'utilisation
1. Enregistrez les articles avec `POST /v1/item`.
2. Attendez le traitement asynchrone.
3. Confirmez avec `GET /v1/item`.
4. Utilisez le `sku` dans les [cotations](api-cotacao-v1.md).
5. Mettez à jour avec `PUT /v1/item` et supprimez avec `DELETE /v1/item`.

## Endpoint : lister les articles

- **Méthode et route :** `GET /v1/item`

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `items[skus][]` | query | array | non | - | Liste de SKU. **Vide ou absente, tous sont retournés** |
| `page` | query | integer | non | `1` | Page de la liste |

- **Éléments par page :** 15

```bash
# Tous les articles
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'

# En filtrant par SKU
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Baskets",
        "description": "Chaussures de course pour homme 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "TSH-000-S",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "unit_value": 220.25,
        "selling_unit_value": 330.00,
        "hscode": "950450",
        "is_groupable": true,
        "is_foldable": false,
        "box_code": 3,
        "box_identifier": "CX0236"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

## Endpoint : créer des articles

- **Méthode et route :** `POST /v1/item`
- **Traitement :** asynchrone, par lot.

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `sku` | string | **oui** | - | - | Code unique du produit dans votre base. **Il ne peut pas se répéter** |
| `name` | string | **oui** | - | - | Nom du produit (apparaît sur l'invoice) |
| `description` | string | **oui** | 6 | 155 | Description du produit. **Des descriptions distinctes entre les articles évitent les alertes douanières** |
| `category` | string | **oui** | 6 | 6 | Catégorie du produit. **Exactement 6 caractères** |
| `country_code` | string | **oui** | 2 | 2 | Pays d'origine/de fabrication en ISO 3166-1 alpha-2 |
| `hscode` | string | **oui** | - | - | NCM/HS Code. Définit la table de taxes appliquée à destination |
| `height` | number | **oui** | 0.01 | - | Hauteur du produit |
| `width` | number | **oui** | 0.01 | - | Largeur du produit |
| `depth` | number | **oui** | 0.01 | - | Profondeur du produit |
| `weight` | number | **oui** | 0.01 | - | Poids unitaire |
| `is_groupable` | boolean | **oui** | - | - | `true` permet de regrouper l'article avec d'autres dans la même boîte. `false` force une boîte exclusive dans `smartbox` |
| `ean` | string | non | - | - | Code EAN/GTIN |
| `unit_value` | number | non | - | - | Valeur unitaire par défaut. Si elle est absente de l'enregistrement **et** de la cotation, la cotation échoue |
| `selling_unit_value` | number | non | - | - | Valeur de vente unitaire. Nécessaire quand la cotation utilise `ior: true` |
| `box_code` | integer | non | - | - | Boîte par défaut de l'article ([Boîte](api-caixa-v1.md)). Doit être un **entier** |
| `box_identifier` | string | non | - | - | Boîte par défaut de l'article, par son identifiant textuel |
| `is_foldable` | boolean | non | - | - | Article pliable. Dans `smartbox`, les articles pliables sont agencés par volume ; les non pliables par dimensions |

> **Les dimensions et le poids** suivent le système de mesure du compte/de la cotation : cm/kg en `metric`,
> in/lb en `imperial`.

### Payload minimal

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton manches courtes blanc taille S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "is_groupable": true
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
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton manches courtes blanc taille S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3,
      "box_identifier": "CX0236"
    },
    {
      "sku": "SNK-044",
      "name": "Baskets de sport 44",
      "description": "Chaussures de course pour homme taille 44 couleur noire",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
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
      {"sku": "TSH-000-S", "name": "T-shirt coton S", "description": "T-shirt 100% coton manches courtes blanc taille S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint : mettre à jour des articles

- **Méthode et route :** `PUT /v1/item`
- **Clé de mise à jour :** le `sku`. Envoyez le SKU existant et les champs à modifier.
- **Traitement :** asynchrone, par lot.
- **Structure du payload :** identique au `POST`.

> ⚠️ **Un SKU inexistant n'est pas créé.** La mise à jour ne fait pas d'« upsert » : si le SKU n'existe pas,
> rien ne se passe (et rien n'est signalé dans la réponse HTTP). Utilisez `POST` pour créer.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "T-shirt coton S", "description": "T-shirt premium 100% coton manches courtes blanc S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint : supprimer des articles

- **Méthode et route :** `DELETE /v1/item`
- **Traitement :** asynchrone, par lot.

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `items` | object | **oui** | Enveloppe |
| `items.skus` | array | **oui** | Liste des SKU à supprimer |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Validation asynchrone — tableau de référence

Comme la validation a lieu pendant le traitement, utilisez ce tableau pour garantir que les articles sont bien pris en compte.
La première règle violée fait rejeter l'article.

| Champ | Règle | Motif consigné dans le rapport |
|---|---|---|
| (article) | Doit être un objet | `not_array` |
| `name` | Obligatoire, texte | `name` |
| `description` | Obligatoire, texte | `description` |
| `category` | Obligatoire, texte, **exactement 6 caractères** | `category` |
| `country_code` | Obligatoire, texte, **exactement 2 caractères** | `country_code` |
| `sku` | Obligatoire, texte | `sku` |
| `ean` | S'il est envoyé, doit être du texte | `ean` |
| `height` | Obligatoire, numérique, **≥ 0.01** | `height` |
| `width` | Obligatoire, numérique, **≥ 0.01** | `width` |
| `depth` | Obligatoire, numérique, **≥ 0.01** | `depth` |
| `weight` | Obligatoire, numérique, **≥ 0.01** | `weight` |
| `hscode` | Obligatoire, texte | `hscode` |
| `is_groupable` | S'il est envoyé, doit être booléen | `is_groupable` |
| `box_code` | S'il est envoyé, doit être un **entier** (pas une chaîne) | `box_code` |
| `box_identifier` | S'il est envoyé, doit être du texte | `box_identifier` |
| `unit_value` | S'il est envoyé, doit être numérique | `unit_value` |
| `selling_unit_value` | S'il est envoyé, doit être numérique | `selling_unit_value` |

Autres situations consignées dans le rapport :

| Situation | Comportement |
|---|---|
| `items` vide ou n'est pas un tableau | Rapport : `Itens isn't an array or is empty` |
| SKU dupliqué dans l'enregistrement | L'enregistrement est rejeté (le SKU est un index unique) |
| SKU inexistant dans le `PUT` | Rien n'est modifié et rien n'est créé |
| SKU invalide dans le `DELETE` | Rapport : `sku_invalid` |

## Comment utiliser les articles dans la cotation

| `type` de cotation | Payload dans `items[]` | Comportement |
|---|---|---|
| `advanced` | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Vous définissez dans quelle boîte va l'article |
| `items` | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | La plateforme choisit les boîtes (regroupement du compte) |
| n'importe lequel | `{ "box_id": 1, "name": "...", "country_code": "BR", "hscode": "...", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Article ponctuel, sans enregistrement |

Voir la section `items[]` dans [Cotation (v1)](api-cotacao-v1.md#paramètres--items).

## Règles métier
- Toutes les opérations d'écriture s'exécutent dans une **file asynchrone**.
- Le `sku` est un **index unique** par compte.
- Les articles sont **par compte** : chaque utilisateur ne voit que les siens.
- L'enregistrement doit être complet pour que la cotation fonctionne. Messages associés :
  - `Nenhum item foi encontrado com o SKU :sku especificado !`
  - `Por favor verifique o cadastro do item :sku, pois poder haver informações imcompletas!`
  - `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).`
  - `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...`
- `is_groupable: false` garantit une boîte exclusive dans le regroupement `smartbox`.
- Un `hscode` incorrect génère une erreur de taxe dans la cotation :
  `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item).`

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `200` avec des articles absents | Article rejeté lors de la validation asynchrone | Vérifier le rapport dans le portail et le tableau de validation ci-dessus |
| `403` | Sans permission pour l'action | Demander la permission au support |
| `422` | `items` absent | Envoyer le tableau `items` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Comme la validation n'est pas synchrone, **confirmez toujours avec `GET /v1/item`** après avoir enregistré ou mis à jour.
- Gardez des descriptions distinctes entre produits différents : des descriptions répétées génèrent le warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` dans la cotation.
- Pour standardiser les emballages utilisés avec ces articles, voir [Boîte](api-caixa-v1.md).
