# Suivi de l'Expédition

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre opération.

## Vue d'ensemble
- Consulte le statut opérationnel et l'historique d'une expédition, depuis sa réception sur la plateforme jusqu'à la
  livraison (ou l'annulation).
- La même réponse offre **trois lectures** : la `timeline` chronologique, les événements **regroupés par statut** et le
  **dernier état** (`last_update`).
- Il existe deux façons de consulter : par l'identifiant de l'expédition (`{shipping}`) ou par l'identifiant de votre
  canal de vente (`channel_identifier_number` + `channel_code`).
- Fonctionne pour les expéditions créées en **v1 et en v2** (les routes de suivi n'existent qu'en v1).

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
| `Authorization` | **oui** | `Bearer <token>` | - | Token de votre compte. Le profil doit disposer de la permission de consultation du statut des expéditions |
| `lang` | non | `pt`, `en`, `es` | `pt` | Langue des messages et des descriptions générées par la plateforme |

> Le header `lang` traduit le `message` de la réponse et les descriptions des événements générés par la **plateforme**
> (réception, paiement, traitement, annulation et erreur). Les descriptions des événements provenant du
> **transporteur** sont renvoyées telles que le transporteur les a envoyées et **ne** changent **pas** selon la langue.
> Les codes (`status_code`) et les clés des groupes ne changent jamais selon la langue — utilisez-les dans la logique de l'intégration.

## Flux d'utilisation
1. Créez l'expédition ([Expédition v1](api-remessa-v1.md) ou [Expédition v2](api-remessa-v2.md)) et conservez le `quotation`
   (UUID) utilisé lors de la création.
2. Consultez le statut par `{shipping}` (recommandé) ou par l'identifiant du canal.
3. Lorsque le groupe `shipped` apparaît, les documents existent déjà : téléchargez-les dans [Documents de l'Expédition](api-documentos.md).
4. Continuez à consulter à des intervalles plus longs pour suivre l'enlèvement, le transit et la livraison.
5. Arrêtez de consulter lorsque `last_update` indique un état final (voir [Bonnes pratiques de polling](#bonnes-pratiques-de-polling)).

```
créer expédition ──► GET .../status ──► (shipped?) ──► GET .../docs
                          │                 │
                          │ non             │ oui
                          ▼                 ▼
                attendre et répéter   suivre enlèvement → transit → livraison
```

## Concepts essentiels

### Les trois lectures de la réponse

| Lecture | Où elle se trouve | Ce qu'elle contient | À utiliser pour |
|---|---|---|---|
| Chronologique | `data.carrier.timeline` | Tous les événements construits par la plateforme et tous les événements de suivi du transporteur, dans l'ordre où ils ont été enregistrés | Afficher l'historique complet à l'utilisateur final |
| Regroupée par statut | `data.received`, `data.shipped`, `data.in_transit`... | Une clé par statut déjà atteint. Chaque clé est une liste contenant **un seul événement** : le plus récent de ce statut | Vérifier si l'expédition **est déjà passée** par une étape donnée |
| Dernier état | `data.last_update` | L'état actuel, au format `{ "<groupe>": { ...événement } }` | Tableaux de bord et synchronisations : une seule lecture répond à « où se trouve l'expédition » |

> **Attention :** les groupes ne conservent pas l'historique. Si le transporteur envoie trois événements `in_transit`, le groupe
> `data.in_transit` ne contiendra que le dernier ; les trois apparaissent dans `data.carrier.timeline`.

### Groupes de statut et codes

| Groupe (clé) | `status_code` de la plateforme | Signification | Où il peut apparaître |
|---|---|---|---|
| `received` | `RCV0001` | L'expédition a été reçue sur la plateforme | Groupe et `timeline` (toujours présent) |
| `processing_payment` | `PPY0001` | Paiement pas encore approuvé (c'est aussi l'événement affiché pour le paiement facturé, `billed`) | Groupe et `timeline` |
| `paid_out` | `POT0001` | Paiement approuvé par le moyen de paiement | Groupe et `timeline` |
| `pending_consolidation` | `PCD0001` | En attente de consolidation (contrats avec consolidation) | Réservé (voir [Règles métier](#règles-métier)) |
| `consolidating` | `CDT0001` | En cours de consolidation | Réservé (voir [Règles métier](#règles-métier)) |
| `processing_shipment` | `PSH0001` | Paiement résolu, en attente du traitement chez le transporteur | Groupe et `timeline` |
| `shipped` | `SPD0001` | **Traitée chez le transporteur. Documents disponibles** | Groupe et `timeline` |
| `collecting` | `CLG0001` | Transporteur en route pour l'enlèvement | Groupe, `timeline` et `last_update` |
| `collected` | `CLD0001` | Enlevée par le transporteur | Groupe, `timeline` et `last_update` |
| `in_transit` | `ITR0001` | En transit | Groupe, `timeline` et `last_update` |
| `delivering` | `DLG0001` | En cours de livraison | Groupe, `timeline` et `last_update` |
| `delivered` | `DLD0001` | Livrée | Groupe, `timeline` et `last_update` |
| `error` | `ERR0001` | Impossible de finaliser l'embarquement | Uniquement `last_update` |
| `cancelling` | `CCG0001` | Annulation demandée, en cours de traitement | Uniquement `last_update` |
| `cancelled` | `CCD0001` | Annulée | Uniquement `last_update` |
| `foc` | `FOC0001` | Finalisée par le transporteur | Uniquement `last_update` |
| `eol` | `EOL0001` | Fin du cycle de vie du suivi | Uniquement `last_update` |

> **`status_code` n'est pas toujours un code `XXX0001`.** Il contient le code brut de l'origine de l'événement lorsque
> ce code existe :
>
> - dans l'événement de paiement : le statut indiqué par le moyen de paiement (ex. : `approved`) ;
> - dans les événements de suivi : le code propre au transporteur.
>
> Les codes `XXX0001` sont utilisés lorsqu'il n'existe pas de code externe. Pour la logique métier, utilisez la **clé du
> groupe** (`shipped`, `in_transit`, `delivered`, `cancelled`...) plutôt que le `status_code`.

### Descriptions générées par la plateforme

Messages littéraux en `pt` (ils changent selon `lang`) :

| Groupe | `status_description` |
|---|---|
| `received` | `Remessa foi recebida em nosso sistema` |
| `processing_payment` | `Aguardando a finalização do processamento do pagamento junto ao gateway: :description ` |
| `paid_out` | `Pagamento aprovado junto ao gateway: :description ` |
| `processing_shipment` | `Remessa esta sendo processada no transportador especificado!` |
| `shipped` | `Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!` |
| `cancelling` (dans `last_update`) | `Cancelando Remessa` |
| `cancelled` (dans `last_update`) | `Remessa Cancelada` |
| `error` (dans `last_update`) | `Não foi possível concluir o embarque desta remessa.` |

> Dans les messages de paiement, `:description` est la description envoyée par le moyen de paiement ou, à défaut,
> le code de la plateforme lui-même (`PPY0001` / `POT0001`). Les deux messages se terminent par un **espace** —
> tenez-en compte si vous comparez des chaînes (il est recommandé de ne pas comparer les descriptions).

### Flux typique des états

```
received
   │
   ├── processing_payment ──► paid_out          (paiement via gateway)
   │
   ├── processing_payment                       (paiement billed : continue directement)
   │
   ▼
processing_shipment
   │
   ▼
shipped                 ◄── documents disponibles à partir d'ici
   │
   ▼
collecting ──► collected ──► in_transit ──► delivering ──► delivered

États affichés uniquement dans last_update :
   error | cancelling ──► cancelled | foc | eol
```

## Endpoint : consulter le statut par expédition

- **Méthode et route :** `GET /v1/shipping/{shipping}/status`
- **Quand l'utiliser :** c'est la manière recommandée. Utilisez l'UUID de la cotation que vous avez envoyé comme `quotation` lors de la création de l'expédition.

### Paramètres

| Nom | Emplacement | Type | Obligatoire | Taille | Par défaut | Description |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | - | **Forme A :** UUID de la cotation utilisée dans l'expédition. **Forme B :** numéro de suivi (AWB) de l'expédition |

> **Identifiants acceptés sur cette route :** UUID de la cotation ou AWB. Si l'AWB figure dans plus d'une expédition,
> la plus récente est prise en compte. **La clé de la NF-e n'est pas acceptée sur cette route** (la consultation répond
> `400`), bien qu'elle soit acceptée sur la route des documents.

### Payload minimal

Endpoint sans corps. Le paramètre de path suffit :

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
```

### Payload maximal

Il n'y a pas de paramètres de query. La forme maximale est identique à la forme minimale, seul l'identifiant varie :

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
GET /v1/shipping/1234567890123/status
```

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK` — expédition avec paiement facturé (`billed`), déjà traitée et en transit :

```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Contrat Exemple Express",
      "tracking_number": "1234567890123",
      "timeline": [
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "PPY0001",
          "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
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
          "status_code": "PU",
          "status_description": "Shipment picked up"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "IT",
          "status_description": "In transit"
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
    "processing_payment": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "PPY0001",
        "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
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
        "status_code": "PU",
        "status_description": "Shipment picked up"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    }
  }
}
```

`200 OK` — extrait de `last_update` d'une expédition annulée (remplace le dernier événement du transporteur) :

```json
{
  "last_update": {
    "cancelled": {
      "update_date": "2026-02-24",
      "update_time": "10:00:00",
      "status_code": "CCD0001",
      "status_description": "Remessa Cancelada"
    }
  }
}
```

`400 Bad Request` — expédition d'un autre compte :

```json
{
  "status": "warning",
  "message": "Nenhuma remessa encontrada com o código especificado!"
}
```

### Champs de la réponse

| Champ | Type | Présence | Description |
|---|---|---|---|
| `status` | string | toujours | `success`, `warning` ou `error` |
| `message` | string | toujours | Message traduit selon le header `lang` |
| `data.carrier.name` | string | toujours | Nom du contrat qui a traité l'expédition ou, sans contrat, nom du transporteur |
| `data.carrier.tracking_number` | string \| null | toujours | Numéro de suivi (AWB) chez le transporteur. Peut être `null`/vide avant le traitement |
| `data.carrier.timeline` | array | toujours | Tous les événements, dans l'ordre d'enregistrement : réception, paiement, traitement puis les événements de suivi du transporteur |
| `data.cost_center` | object | optionnel | N'apparaît que lorsque l'expédition a un centre de coûts |
| `data.cost_center.code` | integer | optionnel | Code du centre de coûts |
| `data.cost_center.name` | string \| null | optionnel | Nom du centre de coûts |
| `data.received` | array | toujours | Événement de réception (un élément) |
| `data.processing_payment` ou `data.paid_out` | array | toujours l'un des deux | Événement de paiement (un élément) |
| `data.processing_shipment` ou `data.shipped` | array | après un paiement approuvé ou facturé | Événement de traitement chez le transporteur (un élément) |
| `data.collecting`, `data.collected`, `data.in_transit`, `data.delivering`, `data.delivered` | array | selon le suivi | Dernier événement du transporteur dans ce statut (un élément) |
| `data.last_update` | object | optionnel | État actuel. **N'apparaît** qu'après le premier événement de suivi du transporteur ou lorsque l'expédition est en `cancelling`, `cancelled` ou `error` |

Chaque événement (dans `timeline`, dans les groupes et dans `last_update`) a la même forme :

| Champ | Type | Format | Description |
|---|---|---|---|
| `update_date` | string | `AAAA-MM-JJ` | Date de l'événement |
| `update_time` | string | `HH:MM:SS` | Heure de l'événement. **Exception :** dans l'événement `processing_shipment`, ce champ contient actuellement la date (`AAAA-MM-JJ`) au lieu de l'heure |
| `status_code` | string | - | Code de la plateforme (`XXX0001`) ou code brut du moyen de paiement/transporteur |
| `status_description` | string | - | Description lisible. Traduite selon `lang` uniquement dans les événements générés par la plateforme |

## Endpoint : consulter le statut par canal

- **Méthode et route :** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`
- **Quand l'utiliser :** lorsque vous n'avez conservé que l'identifiant de la commande dans votre canal de vente (indiqué dans la cotation),
  et non l'UUID de la cotation.

### Paramètres

| Nom | Emplacement | Type | Obligatoire | Taille | Par défaut | Description |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **oui** | - | - | Identifiant de la commande dans le canal, exactement tel qu'indiqué dans la cotation |
| `channel_code` | path | integer | non | - | absent | Code du canal de vente indiqué dans la cotation. Doit être un nombre entier |

> **Comment fonctionne la recherche :**
>
> - **Avec `channel_code` :** recherche l'expédition dont la cotation possède cet identifiant **et** ce code de canal.
> - **Sans `channel_code` :** recherche **uniquement** les expéditions dont la cotation a été créée **sans** code de canal.
>   Si votre cotation avait un `channel_code`, vous devez l'envoyer.
> - Les expéditions **annulées ne sont pas prises en compte**. S'il existe plus d'une expédition valide, la plus récente prévaut.
> - Pour omettre le `channel_code`, il suffit de ne pas envoyer le dernier segment. La valeur `,` est également traitée comme absente.

### Payload minimal

```
GET /v1/shipping/status/PED-2026-0001
```

### Payload maximal

```
GET /v1/shipping/status/PED-2026-0001/1
```

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK` — même structure que [consulter le statut par expédition](#endpoint--consulter-le-statut-par-expédition).

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

Identiques à ceux de [consulter le statut par expédition](#champs-de-la-réponse).

## Règles métier

- **Uniquement les expéditions de votre compte.** L'expédition doit avoir été créée par le compte du token. Les expéditions d'un autre
  compte répondent `400` avec `Nenhuma remessa encontrada com o código especificado!`.
- **Toute expédition doit avoir un enregistrement de paiement** pour que le statut soit construit. S'il n'y en a pas,
  la réponse est `404` avec `Nenhuma entrada de pagamento foi registrada para esta remessa!` (message fixe,
  qui ne change pas selon `lang`).
- **Paiement et groupe affiché :**

  | `payment.type` de l'expédition | Événement de paiement | Événement de traitement |
  |---|---|---|
  | `billed` | `processing_payment` avec `PPY0001` | `processing_shipment` → `shipped` |
  | `gateway` | `processing_payment` (code du moyen de paiement) jusqu'à l'approbation ; ensuite `paid_out` | N'apparaît qu'après `paid_out` |
  | `afterward` | `processing_payment` jusqu'à l'approbation du paiement | N'apparaît qu'après l'approbation du paiement |

  Avec `afterward`, l'expédition peut déjà avoir des documents sans que le groupe `shipped` n'apparaisse. Dans ce cas, consultez
  directement [Documents de l'Expédition](api-documentos.md).
- Le groupe passe de `processing_shipment` à `shipped` au moment où les **documents sont générés** ; la date
  et l'heure de `shipped` sont celles de la génération du premier document.
- Les groupes de suivi (`collecting` à `delivered`) n'apparaissent qu'après que le transporteur a commencé à envoyer des événements.
- Lorsque l'expédition est en `cancelling`, `cancelled` ou `error`, le `last_update` utilise la description propre
  à la plateforme (tableau [Descriptions générées par la plateforme](#descriptions-générées-par-la-plateforme)) au lieu du texte
  brut du transporteur.
- **Consolidation :** tant que l'expédition attend la consolidation ou est en cours de consolidation, la consultation du statut peut
  répondre `500`. Attendez la fin de la consolidation et consultez à nouveau.
- **Annulation automatique :** les expéditions qui, **29 jours après leur création**, n'ont toujours pas de numéro de suivi chez le
  transporteur (et n'ont pas dépassé `shipped`) peuvent être annulées automatiquement par la plateforme.
- **Fin du cycle de vie :** les expéditions de plus de **2 mois** qui n'ont pas atteint un état final
  (`delivered`, `cancelled`, `foc` ou `eol`) sont clôturées par la plateforme et ne reçoivent plus de mises à jour.
- La recherche par AWB sur cette route prend en compte l'expédition la plus récente avec ce numéro.
- La consultation par canal **ignore les expéditions annulées** : pour suivre une annulation, consultez par `{shipping}`.

### Bonnes pratiques de polling

| Situation | Intervalle recommandé |
|---|---|
| Juste après la création de l'expédition (en attente de `shipped`) | 10 à 30 secondes, avec backoff |
| Expédition déjà en `shipped`/`in_transit` | 1 à 4 heures |
| `last_update` en `delivered`, `cancelled`, `foc` ou `eol` | Arrêtez le polling : ce sont des états finaux |
| Expédition de plus de 2 mois | Arrêtez le polling : l'expédition ne reçoit plus de mises à jour |

1. Lisez `data.last_update` pour décider s'il y a eu un changement — c'est la lecture la plus simple. N'oubliez pas qu'elle
   n'existe qu'après le premier événement du transporteur (ou en cas d'annulation/erreur) ; avant cela, utilisez les groupes.
2. Arrêtez le polling dans les états finaux.
3. Utilisez un backoff exponentiel en cas de `500`.
4. Traitez `400`/`404` comme une condition métier : ne répétez pas la même requête en boucle.

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `400` | `remessa.status.status_not_found: ` — aucune expédition trouvée ni par l'UUID ni par l'AWB (consultation par `{shipping}` ; le message arrive sous ce format, sans traduction). Se produit également lors de l'envoi d'une clé de NF-e | Vérifier le `{shipping}`. Utiliser l'UUID de la cotation ou l'AWB |
| `400` | `Nenhuma remessa encontrada com o código especificado!` — expédition d'un autre compte (par `{shipping}`) ou aucune expédition trouvée (par canal) | Vérifier l'identifiant, le `channel_code` et le token utilisé |
| `403` | `Has no permission!` | Demander au support la permission de consultation du statut |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Compte bloqué : contacter le support |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Vérifier le token envoyé dans `Authorization` |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Attendre l'enregistrement du paiement ; si le problème persiste, contacter le support |
| `422` | `O campo channel code deve conter um número inteiro.` (dans `data.channel_code`) | Envoyer un entier dans `channel_code` ou omettre le segment |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Réessayer avec backoff ; si le problème persiste, contacter le support |

## Remarques finales
- `data.carrier.timeline` est la vue la plus complète ; les groupes sont un résumé opérationnel destiné à faciliter la
  lecture programmatique.
- Conservez le `tracking_number` pour proposer au client final le suivi sur le site du transporteur.
- Pour télécharger l'étiquette et l'invoice dès que l'expédition atteint `shipped`, voir
  [Documents de l'Expédition](api-documentos.md).
- Pour créer et annuler des expéditions, voir [Expédition (v1)](api-remessa-v1.md) et [Expédition (v2)](api-remessa-v2.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
