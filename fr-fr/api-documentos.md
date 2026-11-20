# Documents de l'Expédition

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre opération.

## Vue d'ensemble
- Récupère les documents générés par la plateforme et par le transporteur après le traitement de l'expédition :
  AWB (étiquette), invoice, packing list, déclaration et autres.
- Renvoie également les **pièces jointes** que vous avez envoyées lors de la création/mise à jour de l'expédition.
- Chaque document est accompagné d'une `url` de téléchargement et, en option, du contenu en `base64`.
- Les comptes disposant d'une permission spécifique reçoivent également les **documents incorporés** (tous les PDF dans un seul fichier et toutes les étiquettes ZPL dans un seul contenu).
- Fonctionne pour les expéditions créées en **v1 et en v2** (les routes de documents n'existent qu'en v1).

## URL de base
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
lang: pt
```

| Header | Obligatoire | Valeurs | Par défaut | Remarque |
|---|---|---|---|---|
| `Authorization` | **oui** | `Bearer <token>` | - | Token de votre compte. Le profil doit disposer de la permission de consultation des documents d'expédition |
| `lang` | non | `pt`, `en`, `es` | `pt` | Langue des messages |

## Flux d'utilisation
1. Créez l'expédition ([Expédition v1](api-remessa-v1.md) ou [Expédition v2](api-remessa-v2.md)).
2. Consultez le [statut](api-rastreio.md) jusqu'à ce que l'expédition apparaisse dans le groupe `shipped` — c'est à ce moment que
   les documents commencent à exister.
3. Consultez les documents par `{shipping}` ou par le canal.
4. Téléchargez le fichier via l'`url` **immédiatement** ou décodez le `base64`, et stockez-le de votre côté.

> **Avant `shipped`, la consultation peut retourner `400 Nenhum documento encontrado com o código especificado!`.**
> C'est normal : attendez le traitement chez le transporteur. Si l'expédition a des pièces jointes, la réponse peut être
> `200` contenant uniquement les pièces jointes (voir [Règles métier](#règles-métier)).

## Concepts essentiels

### Types de document filtrables (`documents`)

| Valeur dans `documents` | Clé dans `data.docs` | Contenu |
|---|---|---|
| `awb_pdf` | `AWB` | Étiquette/AWB en PDF |
| `awb_zpl` | `AWB_ZPL` et `AWB_ZPL_UPS` | Étiquette/AWB en ZPL (imprimante thermique). Pour certains transporteurs, la clé est `AWB_ZPL_UPS` |
| `awb_prc_pdf` | `AWBPRC` | Étiquette AWB PRC en PDF, émise pour certains transporteurs |
| `awb_prc_zpl` | `AWBPRCZPL` | Étiquette AWB PRC en ZPL |
| `awb_prc_fedex_pdf` | `AWBPRCFedex` | Étiquette AWB PRC FedEx en PDF |
| `awb_prc_fedex_zpl` | `AWBPRCFedexZPL` | Étiquette AWB PRC FedEx en ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice en PDF |
| `packing_list_pdf` | `PackingList` | Packing list en PDF |
| `declaration_pdf` | `Declaration` | Déclaration de contenu en PDF |
| `non_dangerous_pdf` | `NonDangerousObject` | Déclaration de marchandise non dangereuse en PDF |

> **Les clés de `data.docs` ne contiennent pas d'espaces** (`PackingList`, `NonDangerousObject`, `AWBPRC`...), bien que la
> documentation interactive les affiche avec des espaces (`Packing List`, `Non-Dangerous Object`, `AWB PRC`...).
> Programmez la lecture à partir des clés de ce tableau.

Exemple de filtre :
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> Tous les documents n'existent pas pour toutes les expéditions : l'ensemble dépend du transporteur, du pays de destination et du
> type d'objet. Seuls les documents effectivement générés apparaissent dans `data.docs`, et chaque clé apparaît
> **une seule fois**.

### Documents incorporés (`embedded_docs`)

| `type` | Contenu | Condition |
|---|---|---|
| `pdf` | Tous les documents PDF retournés (après le filtre `documents`) réunis dans un seul PDF. Le nombre de copies de chaque document et l'inclusion des pièces jointes PDF suivent la configuration d'incorporation de votre compte | L'expédition doit déjà avoir été traitée chez le transporteur |
| `zpl` | Toutes les étiquettes ZPL retournées (après le filtre `documents`) concaténées, séparées par un saut de ligne | Il doit exister au moins un document ZPL |

Les documents incorporés ne sont générés que lorsque **toutes** les conditions ci-dessous sont vraies :
`embedded=true`, `base64=true`, votre compte dispose de la permission des documents incorporés et l'expédition a
au moins un document retourné. Sinon, `embedded_docs` est renvoyé sous forme de liste vide (`[]`).

### Impression des étiquettes

| Format | Quand l'utiliser |
|---|---|
| `awb_pdf` | Imprimante classique (A4/Letter). Ouvrez l'`url` ou décodez le `base64` |
| `awb_zpl` | Imprimante thermique Zebra. Envoyez le contenu **brut** (après décodage du Base64) directement dans la file d'impression |

> Le contenu ZPL est du texte. Ne le convertissez pas et ne le retraitez pas : envoyez-le exactement tel qu'il a été reçu après le décodage.

### Gestion des pièces jointes

| Opération | Route | Page |
|---|---|---|
| Envoyer des pièces jointes lors de la création | `POST /v1/shipping` (champ `attachments`) | [Expédition (v1)](api-remessa-v1.md) |
| Envoyer des pièces jointes ultérieurement | `PUT /v1/shipping/{shipping}` | [Expédition (v1)](api-remessa-v1.md#endpoint--mettre-à-jour-une-expédition) |
| Lister les pièces jointes | `GET /v1/shipping/{shipping}/docs?attachments=true&base64=true` | Cette page |
| Supprimer des pièces jointes | `DELETE /v1/shipping/{shipping}/docs` | [Expédition (v1)](api-remessa-v1.md#endpoint--supprimer-des-pièces-jointes) |

Limites des pièces jointes :

| Limite | Valeur |
|---|---|
| Formats acceptés | `PDF`, `XML`, `JPEG`, `PNG` |
| Taille par fichier | **5 Mo** |
| Taille totale de la requête | **90 Mo** |

> Les pièces jointes **ne sont retournées qu'avec `base64=true`**. Avec `base64=false`, `data.attachments` est toujours vide.

## Endpoint : consulter les documents par expédition

- **Méthode et route :** `GET /v1/shipping/{shipping}/docs`
- **Quand l'utiliser :** c'est la manière la plus directe. Accepte l'UUID de la cotation, l'AWB ou la clé de la NF-e.

### Paramètres

| Nom | Emplacement | Type | Obligatoire | Taille | Par défaut | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | - | Identifiant de l'expédition. Voir les formes acceptées ci-dessous |
| `attachments` | query | boolean | non | - | `true` | Inclure les pièces jointes que vous avez envoyées. N'a d'effet qu'avec `base64=true` |
| `embedded` | query | boolean | non | - | `true` | Inclure les documents incorporés. N'a d'effet qu'avec `base64=true` et avec la permission spécifique |
| `base64` | query | boolean | non | - | `true` | Inclure le contenu des fichiers en Base64, en plus de l'`url`. Avec `false`, les pièces jointes et les documents incorporés ne sont pas retournés |
| `documents` | query | string | non | - | tous | Liste de types séparés par des virgules (valeurs du tableau [Types de document filtrables](#types-de-document-filtrables-documents)). Vide ou absent : tous |

Façons d'indiquer `{shipping}` (n'en utilisez qu'une) :

| Forme | Valeur | Comment elle est reconnue | Remarques |
|---|---|---|---|
| A | UUID de la cotation | Contient un trait d'union (`-`) | Forme recommandée |
| B | Clé de la NF-e | Exactement 44 chiffres | Prend en compte l'expédition la plus récente avec cette clé, en ignorant les expéditions annulées |
| C | AWB (numéro de suivi) | Toute autre valeur | Uniquement les expéditions créées au cours des **90 derniers jours** ; prend en compte la plus récente avec cet AWB |

> Comme toute valeur contenant un trait d'union est traitée comme un UUID, un AWB contenant `-` n'est pas trouvé par cette route.
> Dans ce cas, utilisez l'UUID de la cotation.

Règles des paramètres de query :

| Règle | Comportement |
|---|---|
| Valeurs booléennes | Sont considérées comme vraies : `true`, `1`, `on`, `yes`. **Toute autre valeur, y compris vide** (`attachments=`), est considérée comme `false` |
| `documents` avec une valeur inconnue | La valeur est ignorée. Si **aucune** valeur n'est valide, le filtre n'est pas appliqué et tous les documents sont retournés |
| `documents` en majuscules | Non reconnu (les valeurs sont sensibles à la casse) |
| `documents` répété (`documents=a&documents=b`) | Seul le **dernier** est pris en compte. Envoyez-les tous dans un seul paramètre, séparés par des virgules |
| `documents[]=...` | Non pris en charge : la requête échoue avec `500` |

> **Astuce de performance :** envoyez `base64=false` lorsque vous téléchargez via l'`url`. Le payload est beaucoup plus petit
> et la réponse bien plus rapide. N'oubliez pas que, dans ce cas, les pièces jointes et les documents incorporés ne figurent pas dans la réponse.

### Payload minimal

Endpoint sans corps. Le paramètre de path suffit (tous les paramètres de query prennent leur valeur par défaut) :

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs
```

### Payload maximal

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Exemple de requête

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf,awb_zpl' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

Uniquement les URL, sans contenu Base64 :

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK` — avec `base64=true`, des pièces jointes et la permission des documents incorporés :

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "Invoice": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "PackingList": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "NonDangerousObject": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Facture fiscale", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML de la facture fiscale", "base64": "PD94bWwg..." }
    ],
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "carrier": "Contrat Exemple Express",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — avec `base64=false` :

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": null
      }
    },
    "attachments": [],
    "embedded_docs": [],
    "carrier": "Contrat Exemple Express",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — expédition encore sans documents, mais avec des pièces jointes (`base64=true`) :

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": [],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Facture fiscale", "base64": "JVBERi0xLjcK..." }
    ],
    "embedded_docs": []
  }
}
```

`400 Bad Request` — aucun document ni aucune pièce jointe :

```json
{
  "status": "warning",
  "message": "Nenhum documento encontrado com o código especificado!"
}
```

### Champs de la réponse

| Champ | Type | Présence | Description |
|---|---|---|---|
| `status` | string | toujours | `success`, `warning` ou `error` |
| `message` | string | toujours | Message traduit selon le header `lang` |
| `data.docs` | object \| array | toujours | Table `clé du document → { url, base64 }`. Les clés sont celles du tableau [Types de document filtrables](#types-de-document-filtrables-documents). **Lorsqu'il n'y a pas de documents, est renvoyé sous forme de liste vide (`[]`)** |
| `data.docs.*.url` | string | toujours | URL de téléchargement du fichier. Peut être temporaire (voir [Règles métier](#règles-métier)) |
| `data.docs.*.base64` | string \| null | toujours | Contenu en Base64. `null` lorsque `base64=false` |
| `data.attachments` | array | toujours | Pièces jointes que vous avez envoyées. Liste vide lorsque `attachments=false`, `base64=false` ou en l'absence de pièces jointes |
| `data.attachments[].id` | integer | - | **ID de la pièce jointe.** Utilisez-le pour la supprimer via `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | - | Extension du fichier : `pdf`, `xml`, `jpg` ou `png` |
| `data.attachments[].description` | string | - | Description indiquée lors de l'envoi de la pièce jointe |
| `data.attachments[].base64` | string | - | Contenu en Base64 |
| `data.embedded_docs` | array | toujours | Documents incorporés. Liste vide lorsqu'ils ne s'appliquent pas (voir [Documents incorporés](#documents-incorporés-embedded_docs)) |
| `data.embedded_docs[].type` | string | - | `pdf` ou `zpl` |
| `data.embedded_docs[].base64` | string | - | Contenu en Base64 |
| `data.carrier` | string | lorsqu'il y a des documents | Nom du contrat qui a traité l'expédition ou, sans contrat, nom du transporteur |
| `data.tracking_id` | string \| null | lorsqu'il y a des documents | Numéro de suivi/AWB |

## Endpoint : consulter les documents par canal

- **Méthode et route :** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- **Quand l'utiliser :** lorsque vous n'avez conservé que le numéro de votre propre commande (indiqué dans la cotation).

### Paramètres

| Nom | Emplacement | Type | Obligatoire | Taille | Par défaut | Description |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **oui** | - | - | Identifiant de la commande dans le canal, exactement tel qu'indiqué dans la cotation |
| `channel_code` | path | integer | non | - | absent | Code du canal de vente indiqué dans la cotation. Doit être un nombre entier |
| `attachments` | query | boolean | non | - | `true` | Identique à [consulter les documents par expédition](#endpoint--consulter-les-documents-par-expédition) |
| `embedded` | query | boolean | non | - | `true` | Identique à [consulter les documents par expédition](#endpoint--consulter-les-documents-par-expédition) |
| `base64` | query | boolean | non | - | `true` | Identique à [consulter les documents par expédition](#endpoint--consulter-les-documents-par-expédition) |
| `documents` | query | string | non | - | tous | Identique à [consulter les documents par expédition](#endpoint--consulter-les-documents-par-expédition) |

> **Comment fonctionne la recherche :**
>
> - **Avec `channel_code` :** recherche la cotation la plus récente avec cet identifiant **et** ce code de canal.
> - **Sans `channel_code` :** recherche la cotation la plus récente avec cet identifiant, **avec ou sans** code de canal.
>   Si vous utilisez le même identifiant dans des canaux différents, envoyez toujours le `channel_code`.
> - Contrairement à la [consultation du statut par canal](api-rastreio.md#endpoint--consulter-le-statut-par-canal), cette recherche
>   **prend également en compte les expéditions annulées**.
> - Pour omettre le `channel_code`, il suffit de ne pas envoyer le segment. La valeur `,` est également traitée comme absente.

### Payload minimal

```
GET /v1/shipping/docs/PED-2026-0001
```

### Payload maximal

```
GET /v1/shipping/docs/PED-2026-0001/1?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Exemple de requête

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK` — même structure que [consulter les documents par expédition](#endpoint--consulter-les-documents-par-expédition).

`422 Unprocessable Entity` — `channel_code` non numérique :

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "channel_code": [
      "O campo channel code deve conter um número inteiro."
    ]
  }
}
```

### Champs de la réponse

Identiques à ceux de [consulter les documents par expédition](#champs-de-la-réponse).

## Règles métier
- Les documents n'existent **qu'après** le traitement chez le transporteur (groupe `shipped` dans le suivi).
- **Ordre de décision de la réponse :**

  | Documents trouvés (après le filtre) | Pièces jointes (`attachments=true` et `base64=true`) | Réponse |
  |---|---|---|
  | Oui | Oui ou non | `200` avec `docs`, `attachments`, `embedded_docs`, `carrier` et `tracking_id` |
  | Non | Oui | `200` avec `docs: []`, `attachments` et `embedded_docs: []` (sans `carrier` ni `tracking_id`) |
  | Non | Non | `400` `Nenhum documento encontrado com o código especificado!` |

- Le filtre `documents` s'applique également à la décision ci-dessus : si les types filtrés n'existent pas encore et qu'il n'y a pas de
  pièces jointes, la réponse est `400`.
- Les documents dont le fichier n'est pas disponible sont omis silencieusement. S'il n'en reste aucun et qu'il n'y a pas de
  pièces jointes, la réponse est `400`.
- `embedded_docs` dépend d'une **permission spécifique** sur votre compte. Sans elle, la liste est vide, même avec
  `embedded=true`.
- **Validité des URL :** lorsque votre compte est configuré pour protéger les liens des documents, chaque `url`
  est **temporaire et valable 5 minutes**. Dans les autres cas, le lien est public, mais il peut également être temporaire.
  **Téléchargez et stockez** les fichiers de votre côté juste après la consultation ; pour les télécharger à nouveau plus tard, consultez
  à nouveau l'endpoint.
- Seuls les documents des expéditions de votre compte sont retournés. Les expéditions d'un autre compte répondent
  `400 Nenhum documento encontrado com o código especificado!` (par UUID et par canal) ou
  `404 Esta remessa não foi encontrada!` (par AWB et par NF-e).
- La recherche par AWB ne prend en compte que les expéditions créées au cours des **90 derniers jours**.
- La recherche par clé de NF-e ignore les expéditions annulées.

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Attendre le traitement (statut `shipped`), revoir le filtre `documents` ou vérifier l'identifiant. Dans la consultation par canal, indique également qu'aucune expédition n'a été trouvée |
| `400` | `Remessa não encontrada!` — aucune expédition avec l'UUID indiqué (consultation par `{shipping}`) | Vérifier l'UUID de la cotation |
| `403` | `Has no permission!` | Demander au support la permission de consultation des documents |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Compte bloqué : contacter le support |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Vérifier le token envoyé dans `Authorization` |
| `404` | `Esta remessa não foi encontrada!` — AWB ou clé de NF-e sans expédition correspondante sur votre compte (AWB de plus de 90 jours, NF-e d'une expédition annulée) | Vérifier l'AWB/la NF-e ou utiliser l'UUID de la cotation |
| `422` | `O campo channel code deve conter um número inteiro.` (dans `data.channel_code`) | Envoyer un entier dans `channel_code` ou omettre le segment |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Réessayer avec backoff ; si le problème persiste, contacter le support |

## Remarques finales
- La consultation par `{shipping}` avec l'UUID de la cotation est la manière la plus directe et la plus stable.
- La consultation par canal est la meilleure option lorsque vous souhaitez travailler uniquement avec le numéro de votre commande.
- Combinez `documents` + `base64=false` pour des réponses rapides dans les intégrations à fort volume.
- Pour savoir quand les documents deviennent disponibles, voir [Suivi de l'Expédition](api-rastreio.md).
- Pour envoyer et supprimer des pièces jointes, voir [Expédition (v1)](api-remessa-v1.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
