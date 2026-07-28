# Documents d'expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Consulte les documents générés après le traitement de l'expédition.
- Peut retourner PDF, ZPL, pièces jointes et, lorsque cela est autorisé, documents intégrés.
- Ce module est publié en v1.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Conventions
- `attachments`, `embedded` et `base64` acceptent des booléens via query string.
- `documents` reçoit une liste séparée par des virgules.

## Flux d'utilisation
1. Créez l'expédition.
2. Attendez que le status indique le traitement/la conclusion.
3. Consultez les documents par `shipping` ou par canal.
4. Téléchargez le contenu dans `url` ou `base64`.

## Endpoints
### Consulter les documents par expédition
- Méthode et route : `GET /v1/shipping/{shipping}/docs`
- `shipping` accepte l'UUID du devis, l'AWB ou la NF-e.

### Consulter les documents par canal
- Méthode et route : `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | oui | Identifiant de l'expédition | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accepte aussi AWB et NF-e |
| `channel_identifier_number` | path | string | oui | Identifiant du canal | `334455` | - |
| `channel_code` | path | integer | non | Code du canal | `1` | Optionnel |
| `attachments` | query | boolean | non | Inclut les pièces jointes | `true` | Default `true` |
| `embedded` | query | boolean | non | Inclut les docs intégrés | `true` | Default `true` |
| `base64` | query | boolean | non | Inclut le base64 | `true` | Default `true` |
| `documents` | query | string | non | Filtre les documents | `invoice_pdf,awb_pdf` | Quand vide, retourne tout |

### Exemple de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemple de response
```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
    "tracking_id": "000000",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "string"
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "string" }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "string" }
    ]
  }
}
```

## Règles métier
- S'il n'y a ni documents ni pièces jointes, l'API peut retourner `warning`.
- Les documents intégrés dépendent d'une autorisation spécifique.
- Les types de documents filtrables sont ceux listés dans la réponse de l'expédition.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | Aucun document trouvé | Confirmer si l'expédition a déjà été traitée |
| `500` | Erreur inattendue | Réessayer et contacter l'équipe responsable |

## Observations finales
- Pour les intégrations, la consultation par `shipping` est la forme la plus directe.
- La consultation par canal est utile lorsque l'intégrateur enregistre l'identifiant commercial dans l'envoi.
