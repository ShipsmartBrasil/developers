# Boîte

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre opération.

## Vue d'ensemble
- Gère l'enregistrement des emballages que vous utilisez fréquemment (nom, dimensions et poids de l'emballage vide).
- Une fois enregistrée, la boîte peut être référencée dans la cotation par `box_code` ou `box_identifier`,
  sans répéter les dimensions et le poids à chaque requête.
- La boîte peut également être liée à un [article enregistré](api-itens-v1.md) comme boîte par défaut du produit.
- L'enregistrement/la mise à jour et la suppression sont **asynchrones** : l'API valide le format, confirme la réception et traite en arrière-plan.
- La liste est **paginée (15 par page)** et conservée en cache pendant **5 minutes** maximum.

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
| `Content-Type` | oui en `POST` et `DELETE` | `application/json` | - |
| `lang` | non | `pt`, `en`, `es` | `pt` |

Chaque opération exige une permission spécifique dans votre profil (lister, enregistrer et supprimer des boîtes).
Sans cette permission, la réponse est `403` avec `Has no permission!`.

## Flux d'utilisation
1. Enregistrez les boîtes avec `POST /v1/box`.
2. Patientez quelques instants (traitement asynchrone). Suivez le résultat via le rapport d'importation dans le tableau de bord.
3. Confirmez avec `GET /v1/box` et notez le `box_code` généré (n'oubliez pas le cache de 5 minutes maximum).
4. Utilisez `box_code` ou `box_identifier` dans les [cotations](api-cotacao-v1.md) et dans l'enregistrement des [articles](api-itens-v1.md).
5. Supprimez les boîtes obsolètes avec `DELETE /v1/box` (de préférence par le `box_code`).

## Concepts essentiels

### Pourquoi utiliser l'enregistrement des boîtes

**Sans enregistrement** — en répétant les dimensions dans chaque cotation :
```json
"boxes": [{ "name": "Boite P", "height": 20, "width": 20, "depth": 20, "weight": 0.3 }]
```

**Avec enregistrement** — en référençant la boîte :
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Avantages : payload plus petit, dimensions standardisées et possibilité de lier chaque article enregistré à une
boîte par défaut (utilisée dans le regroupement `default` de la cotation `type: items`, exclusive à la v2).

### `box_code` x `box_identifier`

| Champ | Qui le définit | Type dans la liste | Utilisation |
|---|---|---|---|
| `box_code` | **La plateforme** (généré à l'enregistrement) | integer | Code numérique de la boîte. Manière la plus fiable de référencer la boîte dans la cotation, l'article et la suppression |
| `box_identifier` | **Vous** (dans le `POST`) | string | Votre propre code d'emballage (ex. : `CX0236`). Doit être unique sur votre compte |

> **Évitez un `box_identifier` composé uniquement de chiffres** (ex. : `"123"`). Dans les filtres de la liste et dans la suppression,
> toute valeur numérique est interprétée comme `box_code`. Préférez des identifiants contenant des lettres, comme `CX0123`.

### Unités de mesure

Les dimensions et le poids des boîtes enregistrées sont toujours interprétés en **centimètres** et en **kilogrammes**.
La conversion du système `imperial` (pouces/livres) effectuée dans la cotation ne s'applique qu'aux mesures envoyées
dans le payload même de la cotation — elle **n'est pas** appliquée aux données provenant de l'enregistrement. Enregistrez toujours en cm/kg.

## Endpoint : lister les boîtes

- **Méthode et route :** `GET /v1/box`
- **Quand l'utiliser :** pour consulter les boîtes enregistrées, découvrir le `box_code` généré et vérifier si l'enregistrement asynchrone est terminé.

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array de string/integer | non | - | Liste de `box_code` et/ou de `box_identifier` (vous pouvez les mélanger). **Vide ou absent, retourne toutes les boîtes** |
| `page` | query | integer | non | `1` | Page de la liste |

- **Éléments par page :** 15 (fixe).
- Les valeurs numériques dans `boxes[box_ids][]` sont recherchées comme `box_code` ; les autres, comme `box_identifier`.
- Les identifiants inexistants sont simplement ignorés (ils ne génèrent pas d'erreur).

> ⚠️ **Cache de 5 minutes :** le résultat est conservé en cache pendant 5 minutes maximum pour chaque combinaison de compte et de filtre.
> Pendant cet intervalle :
> - les boîtes récemment enregistrées, modifiées ou supprimées peuvent ne pas apparaître dans la liste ;
> - **des requêtes pour des pages différentes avec le même filtre peuvent retourner la même page déjà consultée**
>   (le numéro de page ne différencie pas le cache). Si vous avez plus de 15 boîtes, préférez consulter par
>   `boxes[box_ids][]` ou attendez l'expiration du cache entre une page et une autre.

### Payload minimal
Sans paramètres (retourne toutes les boîtes, page 1) :
```
GET /v1/box
```

### Payload maximal
```
GET /v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1&page=1
```

### Exemple de requête

```bash
# Toutes les boites
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'

# Filtrage par identifiant et par code
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartouche Individuelle",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "created_at": "2026-01-27 10:05:05",
        "updated_at": "2026-01-27 10:05:05",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/box?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Champs de la réponse

**Boîte (`data.data[]`) :**

| Champ | Type | Description |
|---|---|---|
| `box_code` | integer | **Code numérique de la boîte**, généré par la plateforme. À utiliser dans `boxes[].box_code` de la cotation et dans `box_code` de l'article |
| `box_identifier` | string \| null | **Votre identifiant textuel de la boîte**. À utiliser dans `boxes[].box_identifier` de la cotation. Peut être `null` pour les boîtes créées sans identifiant via le tableau de bord |
| `name` | string | Nom de la boîte |
| `height`, `width`, `depth` | number | Dimensions enregistrées (cm) |
| `weight` | number | Poids de l'emballage vide (kg) |
| `created_at`, `updated_at` | string | Dates de création et de dernière modification, au format `AAAA-MM-JJ HH:MM:SS` |
| `active` | boolean | Indique si la boîte est active. Traitez également `1`/`0` comme vrai/faux |

**Pagination (`data`) :**

| Champ | Type | Description |
|---|---|---|
| `current_page` | integer | Page actuelle |
| `from`, `to` | integer \| null | Position du premier et du dernier enregistrement de la page |
| `per_page` | integer | Enregistrements par page (15) |
| `total` | integer | Nombre total d'enregistrements trouvés |
| `last_page` | integer | Numéro de la dernière page |
| `first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url` | string \| null | URL de navigation (elles conservent les filtres de la query) |
| `path` | string | URL de base de la liste |
| `links` | array | Liens de navigation prêts à l'emploi (`url`, `label`, `active`) |

## Endpoint : créer ou mettre à jour des boîtes

- **Méthode et route :** `POST /v1/box`
- **Quand l'utiliser :** pour enregistrer de nouvelles boîtes ou modifier des boîtes existantes, par lot.
- **Comportement :** pour **mettre à jour** une boîte existante, indiquez son `box_identifier` (ou son `box_code`).
  Si la boîte n'existe pas, elle est **créée**.
- **Traitement :** l'API valide le format de tous les champs (erreur `422` en cas de problème) et, si tout est correct,
  répond `200` en confirmant **uniquement la réception**. L'enregistrement a lieu en arrière-plan et le résultat figure dans le
  rapport d'importation des boîtes, dans le tableau de bord.

### Paramètres

**Racine du payload :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `boxes` | array | **oui** | 1 élément | - | - | Liste des boîtes à créer/mettre à jour. Un array vide est rejeté |

**Paramètres — `boxes[]` :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `box_identifier` | string | **oui** | - | 155 (utilisez jusqu'à **50**) | - | Votre identifiant unique de la boîte (ex. : `CX0236`). La validation accepte jusqu'à 155 caractères, mais les identifiants de plus de 50 caractères **ne sont pas enregistrés correctement** |
| `name` | string | **oui** | - | 155 | - | Nom descriptif de la boîte |
| `height` | number | **oui** | 1 | - | - | Hauteur en cm |
| `width` | number | **oui** | 1 | - | - | Largeur en cm |
| `depth` | number | **oui** | 1 | - | - | Profondeur en cm |
| `weight` | number | **oui** | - | 11 chiffres entiers + 3 décimales | - | Poids de l'emballage vide en kg. Format accepté : jusqu'à 11 chiffres entiers et jusqu'à **3 décimales**, séparateur `.`, sans signe (ex. : `0.85`, `2.8`, `12`). Indiquez une valeur supérieure à zéro |
| `activated` | boolean | **oui** | - | - | - | `true` pour une boîte active, `false` pour une boîte inactive. Accepte également `1`/`0` |
| `box_code` | **string** | non | - | 50 | - | Code d'une boîte déjà existante, pour la mettre à jour par son code. **Envoyez-le sous forme de texte** (ex. : `"1"`) : envoyé sous forme de nombre, il est rejeté avec `422` |

> **Attention au type de `box_code` dans le `POST` :** ici, il est de type **string** (`"1"`). Dans la cotation, dans l'enregistrement des articles et dans la
> suppression, il est de type **integer** (`1`).

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
      "name": "Boite abandonnee",
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

### Exemple de réponse

**Succès (`200`) :**
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

**Erreur de validation (`422`) :**
```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "boxes.0.height": ["O campo boxes.0.height deve conter um número superior ou igual a 1."],
    "boxes.0.box_code": ["O campo boxes.0.box code deve ser uma string."]
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` (validation/règle) ou `error` (inattendu) |
| `message` | string | Message traduit selon le header `lang` |
| `data` | object | Uniquement en cas d'erreurs de validation : table `champ` → liste de messages. Le champ est accompagné de l'indice de la boîte (`boxes.0.*` = première boîte) |

> **Le `200` ne garantit pas que la boîte a été enregistrée.** Confirmez le résultat avec `GET /v1/box` (en tenant compte du cache de
> 5 minutes maximum) ou via le rapport d'importation dans le tableau de bord.

## Endpoint : supprimer des boîtes

- **Méthode et route :** `DELETE /v1/box`
- **Quand l'utiliser :** pour supprimer définitivement les boîtes qui ne sont plus utilisées.
- **Traitement :** l'API valide le format et répond `200` ; la suppression est effectuée en arrière-plan.
  À la fin, vous recevez **un rapport par e-mail** (à l'adresse e-mail du compte) avec le total des boîtes traitées,
  le total supprimé et la liste des erreurs.
- La suppression est **définitive**. Pour simplement désactiver, utilisez `POST /v1/box` avec `activated: false`.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Min. | Description |
|---|---|---|---|---|---|
| `boxes` | body | object | **oui** | - | Enveloppe |
| `boxes.box_ids` | body | array de integer/string | **oui** | 1 élément | Liste des boîtes à supprimer. **Utilisez le `box_code` sous forme de nombre entier** (ex. : `1`) |

> ⚠️ **Supprimez par le `box_code` (entier).** Bien que la documentation interactive accepte `box_identifier` dans cette liste,
> la suppression par identifiant textuel **n'est pas fiable** pour le moment :
> - la boîte peut être signalée comme `box CX0236 not found!` alors qu'elle existe ;
> - la vérification des articles liés ne reconnaît que le lien fait par `box_code` — une boîte liée à des articles uniquement
>   par `box_identifier` peut être supprimée.
>
> Envoyez également le `box_code` sous forme de **nombre** (`1`) et non de texte (`"1"`) : avec du texte, la vérification des articles
> liés peut ne pas identifier le lien. Consultez le `box_code` avec `GET /v1/box` avant de supprimer.

### Payload minimal

```json
{
  "boxes": {
    "box_ids": [1]
  }
}
```

### Payload maximal

```json
{
  "boxes": {
    "box_ids": [1, 2, 3, 15]
  }
}
```

### Exemple de requête

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": [1, 2]}}'
```

### Exemple de réponse

**Reçu (`200`) :**
```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Malgré le texte, le message ci-dessus confirme uniquement la **réception**. Le résultat réel arrive dans le rapport par e-mail.

**Erreur inattendue (`500`) :**
```json
{
  "status": "error",
  "message": "Algo inesperado aconteceu. Tente novamente mais tarde!"
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` ou `error` |
| `message` | string | Message traduit selon le header `lang` |
| `data` | object | Uniquement en cas d'erreurs de validation : table `champ` → liste de messages |

### Règles de la suppression (rapport par e-mail)

Chaque boîte de `box_ids` est évaluée individuellement ; un échec n'empêche pas la suppression des autres.

| Règle | Message dans le rapport en cas d'échec |
|---|---|
| La boîte doit exister sur votre compte | `Box Error: box %s not found!` (`%s` = valeur envoyée) |
| La boîte **ne doit pas avoir d'articles liés** (`box_code` de l'article) | `Box Error: box %s cannot be deleted, has related item` |

> Pour retirer de l'usage une boîte qui a déjà des articles liés, modifiez la boîte par défaut de ces articles
> ([`PUT /v1/item`](api-itens-v1.md#endpoint--mettre-à-jour-des-articles)) avant de la supprimer, ou désactivez simplement la boîte
> avec `POST /v1/box` et `activated: false`.

## Comment utiliser les boîtes dans la cotation

| Forme | Payload dans `boxes[]` | Quand l'utiliser |
|---|---|---|
| Par code | `{ "box_code": 1 }` | Vous avez conservé le code numérique (integer) |
| Par identifiant | `{ "box_identifier": "CX0236" }` | Vous utilisez votre propre code d'emballage |
| Manuelle | `{ "height": 20, "width": 20, "depth": 20, "weight": 0.3 }` | Emballage ponctuel, sans enregistrement |
| Kit | `{ "kit": "KIT-001" }` | La boîte **et** les articles proviennent de l'enregistrement du kit |

Utilisez **une seule** forme par boîte (ne mélangez pas `box_code` avec `box_identifier` ou avec des dimensions manuelles à la même position).
Une boîte référencée qui n'existe pas génère une erreur dans la cotation :

| Situation | Statut | Message |
|---|---|---|
| `box_code`/`box_identifier` inexistant sur la plateforme | `422` | `O valor selecionado para o campo boxes.0.box code é inválido.` (ou `box identifier`) |
| La boîte n'appartient pas à votre compte | `400` | `A caixa :name não foi encontrada para o usuário especificado!` |

Utilisation des boîtes enregistrées selon le regroupement configuré sur le compte (cotation `type: items`, exclusive à la v2) :

| Regroupement | Comment les boîtes enregistrées sont utilisées |
|---|---|
| `default` | Chaque article enregistré doit avoir une boîte par défaut (`box_code` ou `box_identifier` dans l'[enregistrement de l'article](api-itens-v1.md)). Sans boîte liée : `A caixa com o box_code/box_identifier :name não existe ou está desativada` (HTTP `404`) |
| `smartbox` | Utilise les boîtes indiquées avec dimensions dans `boxes[]` du payload de la cotation. Sans boîte valide : `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (HTTP `400`) |
| `all_in_one` | Consolide les articles ; ne dépend pas de la boîte par défaut de l'article |

Voir la section `boxes[]` dans [Cotation (v1)](api-cotacao-v1.md#paramètres--boxes).

## Règles métier
- La création, la mise à jour et la suppression sont traitées en **arrière-plan** ; le `200` confirme uniquement la réception.
- La liste est conservée en **cache pendant 5 minutes maximum** par compte et par filtre (voir l'alerte sur la pagination ci-dessus).
- Les boîtes sont **par compte** : chaque utilisateur liste, utilise et supprime uniquement les siennes.
- `box_identifier` doit être unique sur votre compte — c'est ce qui permet de mettre à jour au lieu de dupliquer. Utilisez au maximum 50 caractères et évitez les valeurs uniquement numériques.
- Les dimensions et le poids enregistrés sont toujours en **cm** et en **kg**, indépendamment du `measurement` de la cotation.
- L'indicateur `active` est informatif dans la liste : **ne référencez pas de boîtes inactives** dans les cotations. Pour le moment, la cotation
  ne bloque pas de manière garantie une boîte inactive référencée par `box_code`/`box_identifier`.
- Une boîte ayant des articles liés (par le `box_code` de l'article) n'est pas supprimée.
- La suppression est définitive et fait l'objet d'un rapport par e-mail.

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `403` | `Has no permission!` — le profil n'a pas la permission de lister, d'enregistrer ou de supprimer des boîtes | Demander la permission au support |
| `422` | `O campo boxes é obrigatório.` — `boxes` absent ou vide (`POST`/`DELETE`) | Envoyer au moins une boîte (`POST`) ou l'enveloppe `boxes` (`DELETE`) |
| `422` | `O campo boxes deve conter um array.` — `boxes` n'est pas une liste/un objet | Corriger la structure du payload |
| `422` | `O campo boxes.0.box identifier é obrigatório.` / `O campo boxes.0.name é obrigatório.` | Indiquer `box_identifier` et `name` dans toutes les boîtes |
| `422` | `O campo boxes.0.box identifier não pode conter mais de 155 caracteres.` / `O campo boxes.0.name não pode conter mais de 155 caracteres.` | Raccourcir la valeur (pour `box_identifier`, utilisez jusqu'à 50) |
| `422` | `O campo boxes.0.height deve conter um número superior ou igual a 1.` (idem `width`/`depth`) | Indiquer des dimensions ≥ 1 cm |
| `422` | `O campo boxes.0.height deve conter um valor numérico.` (idem `width`/`depth`/`weight`) | Envoyer des nombres, sans unité dans la valeur |
| `422` | `O formato do valor informado no campo boxes.0.weight é inválido.` | Utiliser jusqu'à 3 décimales, le séparateur `.` et aucun signe |
| `422` | `O campo boxes.0.activated deve conter o valor verdadeiro ou falso.` | Envoyer `true`/`false` |
| `422` | `O campo boxes.0.box code deve ser uma string.` | Dans le `POST`, envoyer `box_code` sous forme de texte (`"1"`) |
| `422` | `O campo boxes.0.box code não pode conter mais de 50 caracteres.` | Vérifier le `box_code` |
| `422` | `O campo boxes.box ids é obrigatório.` / `O campo boxes.box ids deve conter um array.` (`DELETE` ; dans le `GET`, uniquement le second) | Envoyer `box_ids` sous forme de liste non vide |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Réessayer avec un intervalle croissant ; si le problème persiste, contacter le support |
| Rapport (e-mail) | `Box Error: box %s not found!` | Vérifier le `box_code` avec `GET /v1/box` |
| Rapport (e-mail) | `Box Error: box %s cannot be deleted, has related item` | Dissocier la boîte des articles ou simplement la désactiver |

> Dans les messages de validation, l'indice (`boxes.0`, `boxes.1`...) indique la position de la boîte dans l'array, à partir de `0`,
> et le `_` du nom du champ apparaît sous forme d'espace (`box identifier`).

## Remarques finales
- Enregistrer les boîtes et les lier aux articles est le prérequis pour la [cotation par articles de la v2](api-cotacao-v2.md#cotation-darticles) (`type: items`) avec le regroupement `default`.
- Gardez les `box_identifier` alignés avec votre ERP/WMS : cela simplifie beaucoup l'intégration.
- Conservez le `box_code` retourné dans la liste : c'est la référence la plus fiable pour la cotation, les articles et la suppression.
- Pour enregistrer les produits placés dans les boîtes, voir [Articles](api-itens-v1.md).
- Pour référencer les boîtes dans la cotation, voir [Cotation (v1)](api-cotacao-v1.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
