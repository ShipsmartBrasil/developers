# Suivi d'expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Consulte le status opérationnel et l'historique d'une expédition.
- Affiche la réception, le paiement, le traitement, le transport et l'annulation.
- Associé au flux opérationnel principal.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Conventions
- Le champ `shipping` accepte UUID, AWB ou NF-e, selon l'endpoint.
- Les status utilisent des codes internes comme `RCV0001`, `PPY0001`, `SPD0001`, `CCG0001` et `CCD0001`.

## Formats de réponse
- Le même endpoint retourne deux formats : `timeline` et `status`.
- `timeline` apporte les événements dans l'ordre où ils arrivent du transporteur.
- `status` apporte les événements regroupés par status, en utilisant le mappage interne de la plateforme pour simplifier la lecture.

## Flux d'utilisation
1. Créez l'expédition.
2. Consultez le status par `shipping` ou par canal.
3. Suivez `received`, `processing_payment`, `processing_shipment`, `shipped` et, le cas échéant, l'annulation.

## Endpoints
### Status par expédition
- Méthode et route : `GET /v1/shipping/{shipping}/status`

### Status par canal
- Méthode et route : `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | oui | Identifiant de l'expédition | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Accepte UUID, AWB et NF-e |
| `channel_identifier_number` | path | string | oui | Numéro du canal | `334455` | - |
| `channel_code` | path | integer | non | Code du canal | `1` | Optionnel |

### Exemple de request
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemple de response
```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        }
      ]
    },
    "received": [],
    "processing_payment": [],
    "shipped": []
  }
}
```

## Règles métier
- Le paiement doit exister pour que l'expédition retourne un status complet.
- Si le paiement n'a pas encore été approuvé, le status peut rester en `processing_payment`.
- Lorsque l'expédition est facturée et pas encore traitée par le transporteur, `processing_shipment` peut apparaître.
- Lorsqu'il y a une annulation, la réponse peut inclure `cancelling` ou `cancelled`.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | Expédition introuvable | Vérifier l'identifiant |
| `404` | Aucun paiement associé | Confirmer le traitement de l'expédition |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- Le suivi est la forme la plus complète pour surveiller la durée de vie de l'envoi.
- La timeline est dérivée des événements internes et du retour du paiement/transporteur.
- Le regroupement de status est un résumé opérationnel pour faciliter la compréhension du suivi de l'expédition.
