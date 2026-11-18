# Documents de l'Expédition

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Récupère les documents générés par la plateforme et par le transporteur après le traitement de l'expédition :
  AWB (étiquette), invoice, packing list, déclaration et autres.
- Renvoie également les **pièces jointes** que vous avez envoyées lors de la création/mise à jour de l'expédition.
- Chaque document est accompagné d'une `url` de téléchargement et, en option, du contenu en `base64`.
- Fonctionne pour les expéditions créées en **v1 et en v2**.

## Base URL
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
lang: pt
```

## Flux d'utilisation
1. Créez l'[expédition](api-remessa-v1.md).
2. Consultez le [statut](api-rastreio.md) jusqu'à ce que l'expédition apparaisse dans le groupe `shipped` — c'est à ce moment que
   les documents commencent à exister.
3. Consultez les documents par `{shipping}` ou par le canal.
4. Téléchargez le fichier via l'`url` (recommandé) ou décodez le `base64`.

> **Avant `shipped`, la consultation peut retourner `400 Nenhum documento encontrado com o código especificado!`.**
> C'est normal : attendez le traitement chez le transporteur.

## Endpoints

### Consulter les documents par expédition
- **Méthode et route :** `GET /v1/shipping/{shipping}/docs`

### Consulter les documents par canal
- **Méthode et route :** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- Utile quand vous n'avez conservé que le numéro de votre propre commande.

## Paramètres

| Nom | Emplacement | Type | Obligatoire | Par défaut | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | UUID de la cotation, numéro de l'AWB (90 derniers jours) ou clé de la NF-e (44 chiffres) |
| `channel_identifier_number` | path | string | **oui** (variante par canal) | - | Identifiant commercial indiqué dans la cotation |
| `channel_code` | path | integer | non | - | Code du canal de vente |
| `attachments` | query | boolean | non | `true` | Inclure les pièces jointes que vous avez envoyées |
| `embedded` | query | boolean | non | `true` | Inclure les documents incorporés (AWB + invoice consolidés dans un seul fichier). Nécessite une permission spécifique |
| `base64` | query | boolean | non | `true` | Inclure le contenu du fichier en Base64, en plus de l'`url` |
| `documents` | query | string | non | tous | Liste de types séparés par des virgules. Lorsqu'elle est vide ou absente, tous les documents sont retournés |

> **Astuce de performance :** envoyez `base64=false` lorsque vous téléchargez via l'`url`. Le payload est beaucoup plus petit
> et la réponse bien plus rapide.

## Types de document filtrables (`documents`)

| Valeur dans `documents` | Clé dans `data.docs` | Contenu |
|---|---|---|
| `awb_pdf` | `AWB` | Étiquette/AWB en PDF |
| `awb_zpl` | `AWB_ZPL` | Étiquette/AWB en ZPL (imprimante thermique) |
| `awb_prc_pdf` | `AWB PRC` | AWB du processus de retour/partenaire en PDF |
| `awb_prc_zpl` | `AWB PRC ZPL` | AWB du processus de retour/partenaire en ZPL |
| `awb_prc_fedex_pdf` | `AWB PRC Fedex` | AWB FedEx du processus de retour/partenaire en PDF |
| `awb_prc_fedex_zpl` | `AWB PRC Fedex ZPL` | AWB FedEx du processus de retour/partenaire en ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice en PDF |
| `packing_list_pdf` | `Packing List` | Packing list en PDF |
| `declaration_pdf` | `Declaration` | Déclaration de contenu en PDF |
| `non_dangerous_pdf` | `Non-Dangerous Object` | Déclaration de marchandise non dangereuse en PDF |

Exemple de filtre :
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> Tous les documents n'existent pas pour toutes les expéditions : l'ensemble dépend du transporteur, du pays de destination et du
> type d'objet. Seuls les documents effectivement générés apparaissent dans `data.docs`.

## Exemple de requête

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=false&base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

Par canal :
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

## Exemple de réponse

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Test",
    "tracking_id": "1234567890123",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "Packing List": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Non-Dangerous Object": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML de la Nota Fiscal", "base64": "PD94bWwg..." }
    ]
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.carrier` | string | Nom du transporteur qui a traité l'expédition |
| `data.tracking_id` | string | Numéro de suivi/AWB |
| `data.docs` | object | Table `nom du document → { url, base64 }`. Les clés sont celles du tableau ci-dessus |
| `data.docs.*.url` | string | URL de téléchargement direct du fichier |
| `data.docs.*.base64` | string | Contenu en Base64. Présent uniquement quand `base64=true` |
| `data.embedded_docs` | array | Documents incorporés. `null`/absent quand `embedded=false` ou sans permission |
| `data.embedded_docs[].type` | string | `pdf` ou `zpl` |
| `data.embedded_docs[].base64` | string | Contenu en Base64 |
| `data.attachments` | array | Pièces jointes que vous avez envoyées. `null`/absent quand `attachments=false` |
| `data.attachments[].id` | integer | **ID de la pièce jointe.** À utiliser pour la supprimer via `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | `pdf`, `xml`, `jpeg` ou `png` |
| `data.attachments[].description` | string | Description indiquée à la création |
| `data.attachments[].base64` | string | Contenu en Base64 |

## Impression des étiquettes

| Format | Quand l'utiliser |
|---|---|
| `awb_pdf` | Imprimante classique (A4/Letter). Ouvrez l'`url` ou décodez le `base64` |
| `awb_zpl` | Imprimante thermique Zebra. Envoyez le contenu **brut** (après décodage du Base64) directement à la file d'impression |

> Le contenu ZPL est du texte. Ne le convertissez pas et ne le retraitez pas : envoyez-le exactement tel qu'il est après le décodage.

## Gestion des pièces jointes

| Opération | Route | Page |
|---|---|---|
| Envoyer des pièces jointes à la création | `POST /v1/shipping` (champ `attachments`) | [Expédition (v1)](api-remessa-v1.md) |
| Envoyer des pièces jointes plus tard | `PUT /v1/shipping/{shipping}` | [Expédition (v1)](api-remessa-v1.md#endpoint--mettre-à-jour-lexpédition) |
| Lister les pièces jointes | `GET /v1/shipping/{shipping}/docs?attachments=true` | Cette page |
| Supprimer des pièces jointes | `DELETE /v1/shipping/{shipping}/docs` | [Expédition (v1)](api-remessa-v1.md#endpoint--supprimer-des-pièces-jointes) |

Limites des pièces jointes :

| Limite | Valeur |
|---|---|
| Formats acceptés | `PDF`, `XML`, `JPEG`, `PNG` |
| Taille par fichier | **5 MB** |
| Taille totale de la requête | **90 MB** |

## Règles métier
- Les documents n'existent qu'**après** le traitement chez le transporteur (groupe `shipped` dans le suivi).
- `embedded_docs` dépend d'une **permission spécifique** sur votre compte. Sans elle, le champ n'est pas retourné,
  même avec `embedded=true`. Message associé :
  `Para obter os documentos incoporados a remessa precisa ter sido ao menos enviada.`
- S'il n'y a aucun document ni aucune pièce jointe, la réponse est `400` avec
  `Nenhum documento encontrado com o código especificado!`.
- Les `url` pointent vers un stockage externe et peuvent avoir une durée de validité. **Téléchargez et stockez** les fichiers
  de votre côté si vous devez les conserver longtemps.
- La consultation par canal prend en compte le `channel_identifier_number` indiqué dans la **cotation**.
  Sans `channel_code`, la recherche n'utilise que l'identifiant.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Attendre le traitement (statut `shipped`) ou vérifier l'identifiant |
| `403` | Sans permission de consulter les documents | Demander la permission au support |
| `404` | Expédition non trouvée | Vérifier `{shipping}` ou l'identifiant du canal |
| `422` | `channel_code` non numérique | Envoyer un entier dans `channel_code` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- La consultation par `{shipping}` est la façon la plus directe et la plus stable.
- La consultation par canal est la meilleure option quand vous voulez travailler uniquement avec le numéro de votre commande.
- Combinez `documents` + `base64=false` pour des réponses rapides dans les intégrations à fort volume.
