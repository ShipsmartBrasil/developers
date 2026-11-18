# Boîte

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Gère l'enregistrement des emballages que vous utilisez fréquemment.
- Une fois enregistrée, la boîte peut être référencée dans la cotation par `box_code` ou `box_identifier`,
  sans répéter les dimensions et le poids à chaque requête.
- L'enregistrement et la suppression sont **asynchrones** : l'API confirme la réception et traite en arrière-plan.

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

## Pourquoi utiliser l'enregistrement des boîtes

**Sans enregistrement** — en répétant les dimensions dans chaque cotation :
```json
"boxes": [{ "name": "Boite S", "height": 20, "width": 20, "depth": 20 }]
```

**Avec enregistrement** — en référençant la boîte :
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Avantages : payload plus petit, dimensions standardisées et utilisation du regroupement `smartbox`
(qui place automatiquement les articles dans les boîtes enregistrées).

## Flux d'utilisation
1. Enregistrez les boîtes avec `POST /v1/box`.
2. Attendez quelques instants (traitement asynchrone).
3. Confirmez avec `GET /v1/box`.
4. Utilisez `box_code` ou `box_identifier` dans les [cotations](api-cotacao-v1.md).
5. Supprimez les boîtes obsolètes avec `DELETE /v1/box`.

## Endpoint : lister les boîtes

- **Méthode et route :** `GET /v1/box`

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array | non | - | Liste de `box_code` et/ou `box_identifier`. **Vide ou absente, toutes sont retournées** |
| `page` | query | integer | non | `1` | Page de la liste |

- **Éléments par page :** 15
- La liste est **mise en cache jusqu'à 5 minutes**. Une boîte tout juste enregistrée peut prendre ce temps avant d'apparaître.

```bash
# Toutes les boîtes
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'

# En filtrant par identifiants
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartouche Individuelle",
        "width": "65",
        "height": "65",
        "depth": "65",
        "weight": "2.8",
        "created_at": "2026-01-27T10:05:05.000000Z",
        "updated_at": "2026-01-27T10:05:05.000000Z",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `box_code` | integer | **Code numérique de la boîte.** À utiliser dans `boxes[].box_code` de la cotation |
| `box_identifier` | string | **Identifiant textuel de la boîte.** À utiliser dans `boxes[].box_identifier` de la cotation |
| `name` | string | Nom de la boîte |
| `height`, `width`, `depth` | number | Dimensions enregistrées |
| `weight` | number | Poids de l'emballage vide |
| `active` | boolean | Boîte active. **Les boîtes inactives ne peuvent pas être utilisées dans la cotation** |
| `created_at`, `updated_at` | string | Dates de création et de dernière modification |

## Endpoint : créer ou mettre à jour des boîtes

- **Méthode et route :** `POST /v1/box`
- **Comportement :** si le `box_identifier` (ou `box_code`) existe déjà, la boîte est **mise à jour**.
  Sinon, elle est **créée**.
- **Traitement :** asynchrone. La réponse `200` ne confirme que la réception.

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `boxes` | array | **oui** | 1 élément | - | Liste des boîtes à créer/mettre à jour |
| `boxes[].box_identifier` | string | **oui** | - | 155 | Identifiant unique de la boîte dans votre base (ex. : `CX0236`) |
| `boxes[].name` | string | **oui** | - | 155 | Nom descriptif |
| `boxes[].height` | number | **oui** | 1 | - | Hauteur |
| `boxes[].width` | number | **oui** | 1 | - | Largeur |
| `boxes[].depth` | number | **oui** | 1 | - | Profondeur |
| `boxes[].weight` | number | **oui** | - | - | Poids de l'emballage vide. Jusqu'à 11 chiffres entiers et **3 décimales** |
| `boxes[].activated` | boolean | **oui** | - | - | `true` pour rendre la boîte disponible dans les cotations |
| `boxes[].box_code` | string | non | - | 50 | Code de la boîte. À indiquer pour mettre à jour une boîte existante par son code |

> Les dimensions suivent le système de mesure de votre compte/cotation : **centimètres** en `metric`,
> **pouces** en `imperial`. Le poids est en **kilogrammes** (`metric`) ou en **livres** (`imperial`).

### Payload minimal

```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartouche Individuelle",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload maximal

```json
{
  "boxes": [
    {
      "box_code": "1",
      "box_identifier": "CX0236",
      "name": "Cartouche Individuelle",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    },
    {
      "box_identifier": "CX0237",
      "name": "Boite Master",
      "height": 40,
      "width": 30,
      "depth": 25,
      "weight": 0.85,
      "activated": true
    },
    {
      "box_identifier": "CX0100",
      "name": "Boite obsolete",
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.2,
      "activated": false
    }
  ]
}
```

### Exemple de requête

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {"box_identifier": "CX0236", "name": "Cartouche Individuelle", "height": 65, "width": 65, "depth": 65, "weight": 2.8, "activated": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

> **Le traitement est asynchrone.** Confirmez le résultat avec `GET /v1/box` (en tenant compte du cache de 5 minutes)
> ou suivez le rapport d'importation dans le portail.

## Endpoint : supprimer des boîtes

- **Méthode et route :** `DELETE /v1/box`
- **Traitement :** asynchrone. À la fin, un rapport par e-mail indique combien de boîtes ont été supprimées
  et lesquelles ont échoué.

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `boxes` | object | **oui** | Enveloppe |
| `boxes.box_ids` | array | **oui** | Liste de `box_code` et/ou `box_identifier` à supprimer |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": ["CX0236", 1]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Règles de la suppression

| Règle | Résultat en cas d'échec |
|---|---|
| La boîte doit exister dans votre base | Erreur dans le rapport : `box %s not found!` |
| La boîte **ne peut pas avoir d'articles liés** | Erreur dans le rapport : `box %s cannot be deleted, has related item` |

> Pour « désactiver » une boîte qui a déjà des articles liés, utilisez `POST /v1/box` avec `activated: false`
> au lieu de la supprimer.

## Comment utiliser les boîtes dans la cotation

| Forme | Payload dans `boxes[]` | Quand l'utiliser |
|---|---|---|
| Par code | `{ "box_code": 1 }` | Vous avez conservé le code numérique |
| Par identifiant | `{ "box_identifier": "CX0236" }` | Vous utilisez votre propre code d'emballage |
| Manuelle | `{ "height": 20, "width": 20, "depth": 20 }` | Emballage ponctuel, sans enregistrement |
| Kit | `{ "kit": "KIT-001" }` | La boîte **et** les articles viennent de l'enregistrement du kit |

Voir la section `boxes[]` dans [Cotation (v1)](api-cotacao-v1.md#paramètres--boxes).

## Règles métier
- L'enregistrement et la suppression sont traités dans une **file asynchrone**.
- La liste est **mise en cache 5 minutes** par utilisateur et par page.
- Les boîtes avec `activated: false` n'apparaissent pas comme disponibles dans les cotations
  (`A caixa :name não existe ou está desativada!`).
- Les boîtes sont **par compte** : chaque utilisateur ne voit que les siennes.
- `box_identifier` doit être unique dans votre base — c'est ce qui permet de mettre à jour au lieu de dupliquer.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | Règle métier lors du traitement | Lire `message` et revoir le payload |
| `403` | Sans permission pour l'action | Demander la permission au support |
| `422` | Validation des champs | Corriger les champs listés dans `data` (dimensions ≥ 1, `box_identifier` obligatoire, `activated` booléen) |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Enregistrer les boîtes est le prérequis pour utiliser le regroupement `smartbox` et le type de cotation `items`.
- Gardez les `box_identifier` alignés avec votre ERP/WMS : cela simplifie beaucoup l'intégration.
- Pour enregistrer les produits qui vont dans les boîtes, voir [Articles](api-itens-v1.md).
