# Suivi de l'Expédition

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Consulte le statut opérationnel et l'historique complet d'une expédition, de sa réception sur la plateforme jusqu'à la
  livraison (ou l'annulation).
- La même réponse contient **deux formats** : une `timeline` chronologique et les événements **regroupés par statut**.
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

> Les descriptions de statut (`status_description`) sont traduites selon le header `lang`.
> Les codes (`status_code`) ne changent **pas** avec la langue — utilisez-les pour la logique d'intégration.

## Endpoints

### Statut par expédition
- **Méthode et route :** `GET /v1/shipping/{shipping}/status`

### Statut par canal
- **Méthode et route :** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

## Paramètres

| Nom | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `shipping` | path | string | **oui** | UUID de la cotation, numéro de l'AWB (90 derniers jours) ou clé de la NF-e (44 chiffres) |
| `channel_identifier_number` | path | string | **oui** (variante par canal) | Identifiant commercial indiqué dans la cotation |
| `channel_code` | path | integer | non | Code du canal de vente |

## Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

Par canal :
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

## Exemple de réponse

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
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:40",
          "status_code": "POT0001",
          "status_description": "Pagamento aprovado junto ao gateway: approved"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:44:38",
          "status_code": "SPD0001",
          "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
        },
        {
          "update_date": "2026-02-22",
          "update_time": "09:10:02",
          "status_code": "CLD0001",
          "status_description": "Remessa coletada pelo transportador"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "ITR0001",
          "status_description": "Remessa em trânsito internacional"
        }
      ]
    },
    "cost_center": {
      "code": 1,
      "name": "E-commerce"
    },
    "received": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "RCV0001",
        "status_description": "Remessa foi recebida em nosso sistema"
      }
    ],
    "paid_out": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:40",
        "status_code": "POT0001",
        "status_description": "Pagamento aprovado junto ao gateway: approved"
      }
    ],
    "shipped": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:44:38",
        "status_code": "SPD0001",
        "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
      }
    ],
    "collected": [
      {
        "update_date": "2026-02-22",
        "update_time": "09:10:02",
        "status_code": "CLD0001",
        "status_description": "Remessa coletada pelo transportador"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    }
  }
}
```

## Structure de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.carrier.name` | string | Nom du contrat/transporteur qui a traité l'expédition |
| `data.carrier.tracking_number` | string | Numéro de suivi (AWB) chez le transporteur |
| `data.carrier.timeline` | array | **Tous les événements par ordre chronologique.** Utilisez ce champ pour construire une chronologie dans votre interface |
| `data.cost_center` | object | **Optionnel.** N'apparaît que lorsque l'expédition a un centre de coûts |
| `data.cost_center.code` | integer | Code du centre de coûts |
| `data.cost_center.name` | string | Nom du centre de coûts |
| `data.<groupe_de_statut>` | array | Événements regroupés par statut. Les clés présentes varient selon l'avancement |
| `data.last_update` | object | **Dernier événement**, au format `{ "<groupe>": { ...événement } }`. À utiliser pour connaître l'état actuel en une seule lecture |

Chaque événement (dans `timeline`, dans les groupes et dans `last_update`) a la même forme :

| Champ | Type | Description |
|---|---|---|
| `update_date` | string | Date de l'événement (`Y-m-d`) |
| `update_time` | string | Heure de l'événement (`H:i:s`) |
| `status_code` | string | Code interne du statut (voir tableau) ou le code brut du gateway/transporteur |
| `status_description` | string | Description lisible, traduite selon le header `lang` |

### Les deux formats, et quand utiliser chacun

| Format | Où il se trouve | À utiliser pour |
|---|---|---|
| Chronologique | `data.carrier.timeline` | Afficher l'historique complet à l'utilisateur final |
| Regroupé par statut | `data.received`, `data.shipped`, `data.in_transit`... | Vérifier si l'expédition **est déjà passée** par une étape donnée (`if (isset(data.shipped))`) |
| Dernier état | `data.last_update` | Tableaux de bord et synchronisations : une lecture répond à « où est l'expédition » |

## Groupes de statut et codes

| Groupe (clé dans `data`) | `status_code` | Signification |
|---|---|---|
| `received` | `RCV0001` | L'expédition a été reçue sur la plateforme |
| `processing_payment` | `PPY0001` | En attente de la confirmation du paiement par le gateway |
| `paid_out` | `POT0001` | Paiement approuvé |
| `pending_consolidation` | `PCD0001` | En attente de consolidation (contrats avec consolidation) |
| `consolidating` | `CDT0001` | En cours de consolidation |
| `processing_shipment` | `PSH0001` | Paiement réglé, en attente du traitement chez le transporteur |
| `shipped` | `SPD0001` | **Traitée chez le transporteur. Documents disponibles** |
| `collecting` | `CLG0001` | Enlèvement en cours |
| `collected` | `CLD0001` | Enlevée par le transporteur |
| `in_transit` | `ITR0001` | En transit |
| `delivering` | `DLG0001` | En cours de livraison |
| `delivered` | `DLD0001` | Livrée |
| `error` | `ERR0001` | Erreur lors du traitement |
| `cancelling` | `CCG0001` | Annulation demandée, en cours de traitement |
| `cancelled` | `CCD0001` | Annulée |
| `foc` | `FOC0001` | Finalisée par le transporteur |
| `eol` | `EOL0001` | Fin du cycle de vie du suivi |

> **`status_code` peut contenir le code brut du gateway ou du transporteur** lorsqu'il existe
> (ex. : `approved`, `charge.succeeded`, codes propres au transporteur). Les codes `XXX0001` sont la
> correspondance interne de la plateforme, utilisée quand il n'y a pas de code externe. Pour la logique métier, préférez la
> **clé du groupe** (`shipped`, `delivered`, `cancelled`...) plutôt que le `status_code`.

## Flux typique des états

```
received
   │
   ├── processing_payment ──► paid_out        (paiement via gateway)
   │
   └── (paiement billed : continue directement)
                │
                ├── pending_consolidation ──► consolidating   (quand il y a consolidation)
                │
                ▼
         processing_shipment
                │
                ▼
             shipped          ◄── documents disponibles à partir d'ici
                │
                ▼
    collecting ──► collected ──► in_transit ──► delivering ──► delivered
                │
                └── error | cancelling ──► cancelled | foc | eol
```

## Règles métier

- **Toute expédition doit avoir un enregistrement de paiement** pour que le statut soit construit. S'il n'y en a pas,
  la réponse est `404` avec `Nenhuma entrada de pagamento foi registrada para esta remessa!`.
- Avec `payment.type: billed`, l'expédition ne passe pas par `processing_payment` : elle va directement vers
  `processing_shipment`/`shipped`.
- Avec `payment.type: gateway`, l'expédition reste en `processing_payment` jusqu'à l'approbation du gateway ; ce n'est qu'ensuite
  qu'elle avance vers `processing_shipment`.
- Le groupe passe de `processing_shipment` à `shipped` au moment où les **documents sont générés**.
  C'est le signal pour appeler [Documents de l'Expédition](api-documentos.md).
- Quand l'expédition est en `cancelling`, `cancelled` ou `error`, le `last_update` utilise la description propre
  à la plateforme au lieu du texte brut du transporteur.
- Les expéditions qui restent **29 jours ou plus** sans événement de suivi dans un statut éligible peuvent être annulées
  automatiquement par la plateforme.
- La recherche par AWB fonctionne pour les expéditions créées dans les **90 derniers jours**.

## Bonnes pratiques de polling

| Situation | Intervalle recommandé |
|---|---|
| Juste après la création de l'expédition (en attente de `shipped`) | 10 à 30 secondes, avec backoff |
| Expédition déjà en `shipped`/`in_transit` | 1 à 4 heures |
| Expédition en `delivered`, `cancelled`, `foc` ou `eol` | Arrêtez le polling : ce sont des états finaux |

1. Lisez `data.last_update` pour décider s'il y a eu un changement — c'est la lecture la moins coûteuse.
2. Arrêtez le polling dans les états finaux.
3. Utilisez un backoff exponentiel en cas de `500`.
4. Traitez `400`/`404` comme une condition métier : ne répétez pas la même requête en boucle.

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Vérifier `{shipping}` ou l'identifiant du canal |
| `403` | Sans permission de consulter le statut | Demander la permission au support |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Attendre l'enregistrement du paiement ; si cela persiste, contacter le support |
| `422` | `channel_code` non numérique | Envoyer un entier dans `channel_code` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- `data.carrier.timeline` est la vue la plus complète ; les groupes sont un résumé opérationnel pour faciliter la
  lecture programmatique.
- Conservez le `tracking_number` pour offrir au client final le suivi sur le site du transporteur.
- Pour télécharger l'étiquette et l'invoice dès que l'expédition atteint `shipped`, voir
  [Documents de l'Expédition](api-documentos.md).
