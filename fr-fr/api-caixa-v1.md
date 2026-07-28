# Colis

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Gère l'enregistrement et la consultation des colis utilisés dans les devis, brouillons et expéditions.
- Utilisez-le pour standardiser les dimensions, le poids et l'identification des emballages.
## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue optionnel : `lang`.

## Conventions
- `Content-Type: application/json`
- La consultation utilise `boxes[box_ids][]` dans la query.
- L'enregistrement/la mise à jour s'effectue de manière asynchrone.

## Flux d'utilisation
1. Listez les colis disponibles.
2. Enregistrez ou mettez à jour les colis de base.
3. Supprimez les colis lorsque nécessaire.
4. Réutilisez les codes dans les devis et les brouillons.

## Endpoints
### Lister les colis
- Méthode et route : `GET /v1/box`
- Quand l'utiliser : pour rechercher des colis enregistrés ou filtrer par IDs.

### Créer ou mettre à jour des colis
- Méthode et route : `POST /v1/box`
- Quand l'utiliser : lors de la maintenance de la base d'emballages de l'intégrateur.

### Supprimer des colis
- Méthode et route : `DELETE /v1/box`
- Quand l'utiliser : lors de la suppression d'un ou plusieurs colis enregistrés.

## Paramètres
### Listage
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `boxes[box_ids][]` | query | array | non | Liste de codes/identifiants | `1` | Vide retourne tout |

### Enregistrement
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `boxes[]` | body | array | oui | Liste de colis | - | Chaque élément est traité en lot |
| `box_code` | body | string | non | Code du colis | `1` | Optionnel pour la mise à jour |
| `box_identifier` | body | string | oui | Identifiant du colis | `CX0236` | Obligatoire lors de l'enregistrement |
| `name` | body | string | oui | Nom | `Cartucho Individual` | Jusqu'à 155 caractères |
| `height` | body | number | oui | Hauteur | `65` | >= 1 |
| `width` | body | number | oui | Largeur | `65` | >= 1 |
| `depth` | body | number | oui | Profondeur | `65` | >= 1 |
| `weight` | body | number | oui | Poids | `2.8` | Regex avec jusqu'à 3 décimales |
| `activated` | body | boolean | oui | Activé | `true` | - |

### Suppression
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `boxes.box_ids` | body | array | oui | IDs des colis | `[1, 2]` | Liste obligatoire |

## Payload minimum et maximum
### Payload minimum
```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload maximum
```json
{
  "boxes": [
    {
      "box_code": 1,
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

## Exemple de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "activated": true
      }
    ]
  }'
```

## Exemple de response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Règles métier
- L'enregistrement est traité dans un job asynchrone.
- Le listage peut être mis en cache par utilisateur et par page.
- La suppression accepte plusieurs IDs.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `422` | Validation | Corriger les champs obligatoires |
| `400` | Requête invalide | Vérifier le payload | 
| `500` | Erreur inattendue | Contacter l'équipe responsable |
