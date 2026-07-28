# Articles

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Gère les articles enregistrés pour une utilisation dans les devis, brouillons et expéditions.
- Les articles peuvent être consultés, créés, mis à jour et supprimés en lot.
## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue optionnel : `lang`.

## Conventions
- `Content-Type: application/json`
- La consultation utilise `items[skus][]` dans la query.
- L'enregistrement/la mise à jour/la suppression sont asynchrones.

## Flux d'utilisation
1. Listez les articles déjà enregistrés.
2. Enregistrez les articles de base.
3. Mettez à jour les articles lorsqu'il y a un changement de dimension, poids ou classification.
4. Supprimez les articles obsolètes.

## Endpoints
### Lister les articles
- Méthode et route : `GET /v1/item`

### Créer des articles
- Méthode et route : `POST /v1/item`

### Mettre à jour des articles
- Méthode et route : `PUT /v1/item`

### Supprimer des articles
- Méthode et route : `DELETE /v1/item`

## Paramètres
### Listage
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `items[skus][]` | query | array | non | Liste de SKUs | `TSH-000-S` | Vide retourne tout |

### Enregistrement/mise à jour
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `items[]` | body | array | oui | Liste d'articles | - | Traitement en lot |
| `name` | body | string | oui | Nom de l'article | `Tenis` | Obligatoire |
| `description` | body | string | oui | Description | `Nike 44` | Jusqu'à 155 caractères |
| `category` | body | string | oui | Catégorie | `950450` | Valeur libre dans la source |
| `country_code` | body | string | oui | Pays d'origine | `BR` | ISO2 |
| `sku` | body | string | oui | SKU | `oLdTsW222` | Obligatoire |
| `ean` | body | string | non | EAN | `7894900010015` | Optionnel |
| `height` | body | integer | oui | Hauteur | `5` | - |
| `width` | body | integer | oui | Largeur | `5` | - |
| `depth` | body | integer | oui | Profondeur | `5` | - |
| `weight` | body | number | oui | Poids | `0.3` | - |
| `unit_value` | body | number | non | Valeur unitaire | `220.25` | Optionnel |
| `selling_unit_value` | body | number | non | Valeur de vente | `330.00` | Optionnel |
| `hscode` | body | string | oui | HS code | `950450` | Obligatoire |
| `is_groupable` | body | boolean | oui | Peut être groupé | `true` | Obligatoire |
| `box_code` | body | integer | non | Colis par défaut | `3` | Optionnel |
| `box_identifier` | body | string | non | Colis par défaut | `CX0236` | Optionnel |

### Suppression
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---|---|---|---|---|
| `items.skus` | body | array | oui | SKUs à supprimer | `["TSH-000-S"]` | Obligatoire |

## Payload minimum et maximum
### Payload minimum
```json
{
  "items": [
    {
      "name": "Teste prod 1",
      "description": "Teste prod 1",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "hscode": "673057",
      "is_groupable": false
    }
  ]
}
```

### Payload maximum
```json
{
  "items": [
    {
      "name": "Teste prod 1 update",
      "description": "Teste prod 1 update",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "ean": "7894900010015",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "unit_value": 220.25,
      "selling_unit_value": 330,
      "hscode": "673057",
      "is_groupable": false,
      "box_code": 129,
      "box_identifier": "CX0236"
    }
  ]
}
```

## Exemple de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Exemple de response
```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Tenis",
        "description": "Nike 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "oLdTsW222",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "hscode": "950450",
        "is_groupable": true
      }
    ]
  }
}
```

## Règles métier
- Les articles sont traités dans un job asynchrone.
- Le listage utilise une collection par utilisateur.
- `box_code` et `box_identifier` sont optionnels et servent de lien par défaut avec le colis.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `422` | Validation | Corriger les champs obligatoires |
| `500` | Erreur inattendue | Contacter l'équipe responsable |
