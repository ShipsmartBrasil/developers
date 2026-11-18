# Planification d'Enlèvement

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md) et [Expédition (v1)](api-remessa-v1.md)

## Vue d'ensemble
- Demande au transporteur de **retirer** le colis à votre adresse, au lieu que vous l'apportiez à un point
  de dépôt.
- Permet de consulter le statut de la planification et de l'annuler avec un motif normalisé.
- Fonctionne pour les expéditions créées en **v1 et en v2**.

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

## Prérequis

Avant de planifier, confirmez que :

1. L'expédition est déjà au statut **`shipped`** ([suivi](api-rastreio.md)) — c'est-à-dire qu'elle a déjà été traitée chez le
   transporteur et que les documents existent.
2. Le transporteur du contrat choisi **dispose d'un service d'enlèvement intégré**.
3. L'expédition **n'a pas** une autre planification en cours.

## Flux d'utilisation

```
1. GET  /v1/shipping/{shipping}/status   → confirmer le statut "shipped"
2. POST /v1/schedule                     → créer la planification
3. GET  /v1/schedule/{shipping}          → suivre jusqu'au statut "success"
4. DELETE /v1/schedule/{shipping}/{reason} → annuler (si nécessaire)
```

## Conventions
- Dates au format `Y-m-d` (ex. : `2026-03-10`).
- Heures au format `H:i:s` (ex. : `08:00:00`).
- L'identifiant `{shipping}` accepte l'**UUID de la cotation** ou le **numéro de l'AWB** (expéditions des
  90 derniers jours). Contrairement aux endpoints d'expédition, **la clé de la NF-e n'est pas acceptée ici**.

## Endpoint : créer une planification

- **Méthode et route :** `POST /v1/schedule`
- **Quand l'utiliser :** après que l'expédition a atteint `shipped`.

### Paramètres

| Champ | Type | Obligatoire | Min. | Max. | Format | Description |
|---|---|---|---|---|---|---|
| `shipping` | string | **oui** si `awb_number` n'est pas envoyé | - | - | UUID | Code de la cotation/expédition |
| `awb_number` | string | **oui** si `shipping` n'est pas envoyé | - | 50 | - | Numéro de l'AWB (expéditions des 90 derniers jours) |
| `date` | string | **oui** | - | - | `Y-m-d` | Date de l'enlèvement. Elle doit être **postérieure à aujourd'hui** |
| `initial_time` | string | **oui** | - | - | `H:i:s` | Début de la plage d'enlèvement |
| `final_time` | string | **oui** | - | - | `H:i:s` | Fin de la plage d'enlèvement |
| `instruction` | string | **oui** | 5 | 80 | - | Instruction pour le chauffeur (ex. : où retirer, qui contacter) |
| `address` | object | non | - | - | - | Adresse alternative d'enlèvement (voir tableau ci-dessous) |

> Envoyez **`shipping` ou `awb_number`**, pas les deux.

### Paramètres — `address` (adresse alternative d'enlèvement)

À envoyer uniquement si l'enlèvement a lieu à une adresse **différente** de celle de l'expéditeur de l'expédition.
**L'adresse doit se trouver dans le même pays que l'expéditeur.**

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `street` | string | **oui** si `address` est envoyé | 5 | 70 | Rue |
| `city` | string | **oui** si `address` est envoyé | 5 | 35 | Ville |
| `name` | string | **oui** si `address` est envoyé | 5 | 35 | Nom du contact sur place |
| `email` | string | **oui** si `address` est envoyé | - | 100 | E-mail du contact |
| `phone` | string | **oui** si `address` est envoyé | 10 | 25 | Téléphone du contact |
| `postal_code` | string | conditionnel | - | 12 | Code postal/ZIP. **Obligatoire pour les enlèvements au Brésil** |
| `state_code` | string | non | - | 10 | Sigle de l'état. Il doit appartenir au pays de l'expéditeur |
| `number` | string | conditionnel | - | 6 | Numéro du bâtiment. **Obligatoire pour les enlèvements au Brésil** |
| `district` | string | conditionnel | - | 30 | Quartier. **Obligatoire pour les enlèvements au Brésil** |
| `complement` | string | non | - | 20 | Complément |
| `phone_extension` | string | non | - | 5 | Poste |
| `federal_tax_id` | string | conditionnel | - | 20 | CPF/CNPJ. **Obligatoire et validé pour les enlèvements au Brésil** |

> Quand `address` est omis, la plateforme résout l'adresse automatiquement : d'abord le préréglage
> d'enlèvement de votre compte et, à défaut, l'adresse de l'expéditeur de l'expédition.
>
> Pour certains types de compte (personne physique et personne morale mono-CD), l'adresse alternative est **ignorée** et l'enlèvement
> utilise toujours l'adresse de l'expéditeur.

### Payload minimal

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirer a la reception, demander Carlos"
}
```

### Payload maximal

```json
{
  "awb_number": "271249639538",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Retirer au hangar 2, portail lateral, demander Carlos",
  "address": {
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Hangar 2",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

### Exemple de requête

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2026-03-10",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirer a la reception, demander Carlos"
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> La réponse confirme la **réception**. La confirmation du transporteur est asynchrone : consultez la
> planification jusqu'à ce que le statut passe à `success` et que le `confirmation_code` apparaisse.

### Règles de date et de plage horaire

| Règle | Détail | Message en cas d'échec |
|---|---|---|
| Date future | `date` doit être **postérieure à la date actuelle** | `A data de agendamento não pode ser menor que a data Atual` |
| Pas de week-end | Le samedi et le dimanche ne sont pas acceptés | `Coleta não pode ser realizada em um final de semana` |
| Pas de jour férié national | Pour les enlèvements au Brésil | `Coleta não pode ser realizada durante um feriado nacional` |
| Limite de 7 jours ouvrés | La date ne peut pas dépasser 7 jours ouvrés à partir d'aujourd'hui | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Jours fériés nationaux brésiliens pris en compte : **01/01, 21/04, 01/05, 07/09, 12/10, 02/11, 15/11, 20/11 et 25/12**.

## Endpoint : consulter une planification

- **Méthode et route :** `GET /v1/schedule/{shipping}`

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `shipping` | path | string | **oui** | UUID de la cotation ou numéro de l'AWB (90 derniers jours) |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirer a la reception, demander Carlos",
    "status": {
      "code": "processing",
      "description": "Agendamento da Remessa Recebida. Em breve será processada!",
      "reason": null
    }
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.confirmation_code` | string | Numéro de confirmation de l'enlèvement chez le transporteur. Renseigné quand le statut atteint `success` |
| `data.date` | string | Date de l'enlèvement (`Y-m-d`) |
| `data.inicial_time` | string | Début de la plage (`H:i:s`). ⚠️ **La clé est `inicial_time`** dans la réponse, alors que dans la requête le champ est `initial_time` |
| `data.final_time` | string | Fin de la plage (`H:i:s`) |
| `data.instruction` | string | Instruction indiquée à la création |
| `data.status.code` | string | Code du statut de la planification (voir tableau) |
| `data.status.description` | string | Description traduite selon le header `lang` |
| `data.status.reason` | string \| null | Code du motif d'annulation (`001`–`008`). `null` tant qu'il n'y a pas d'annulation |

### Statuts de la planification

| `code` | Signification |
|---|---|
| `processing` | Planification reçue, en attente d'envoi au transporteur |
| `success` | **Enlèvement confirmé par le transporteur.** `confirmation_code` disponible |
| `canceling` | Annulation demandée, en attente de traitement |
| `terminating` | Annulation en cours de traitement chez le transporteur |
| `canceled` | Annulation terminée |
| `error` | Échec de la planification ou de l'annulation |

### Type de traitement

La plateforme classe la planification automatiquement, sans champ dans le payload :

| Type | Quand | Comportement |
|---|---|---|
| `automatico` | Enlèvement pour le lendemain (jusqu'à 1 jour) | Envoyé au transporteur immédiatement |
| `agendado` | Enlèvement avec plus d'un jour d'anticipation | Envoyé au transporteur 1 jour avant l'enlèvement |

> C'est pourquoi une planification future peut rester en `processing` pendant des jours. C'est normal.

## Endpoint : annuler une planification

- **Méthode et route :** `DELETE /v1/schedule/{shipping}/{reason}`
- **Quand l'utiliser :** quand l'enlèvement a déjà été **confirmé** (statut `success`) et doit être annulé.

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `shipping` | path | string | **oui** | UUID de la cotation ou numéro de l'AWB (90 derniers jours) |
| `reason` | path | string | **oui** | Code du motif (voir tableau). Transmis au transporteur |

### Motifs d'annulation

| Code | Motif |
|---|---|
| `001` | Package not ready (le colis n'est pas prêt) |
| `002` | Rates too high (tarifs trop élevés) |
| `003` | Transit time too slow (délai de transit trop lent) |
| `004` | Take to service center or drop box (sera apporté à un point de dépôt) |
| `005` | Commitment time not met (délai convenu non respecté) |
| `006` | Reason not given (motif non indiqué) |
| `007` | Other (autre) |
| `008` | Pickup modified (enlèvement modifié) |

> Ces codes sont normalisés par les transporteurs et leur sont transmis sans traduction.
> Utilisez `006` ou `007` quand aucun des motifs spécifiques ne s'applique.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Règles de l'annulation

| Règle | Message en cas d'échec |
|---|---|
| Le `reason` doit être entre `001` et `008` | `Motivo do cancelamento não é compativel` |
| La planification doit exister | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| L'expédition doit être en `shipped` | idem |
| La planification doit être en `success` (déjà confirmée par le transporteur) | idem |

> Après la demande, le statut passe à `canceling` avec la description
> `Solicitação de cancelamento recebida! Em breve será processada!`, puis `terminating` et, enfin,
> `canceled` (ou `error`).

## Règles métier consolidées

| Règle | Détail |
|---|---|
| Statut de l'expédition | L'expédition doit être en `shipped` |
| Planification unique | Il n'est pas possible d'avoir deux planifications actives sur la même expédition |
| Transporteur | Le service doit prendre en charge l'enlèvement intégré |
| Date | Future, jour ouvré, hors jour férié national (BR), jusqu'à 7 jours ouvrés |
| Adresse BR | Exige `postal_code`, `number`, `district` et un `federal_tax_id` valide (CPF ou CNPJ) |
| Pays de l'adresse | Le `state_code` doit appartenir au pays de l'expéditeur de l'expédition |
| Propriété | Il n'est possible de planifier/consulter/annuler que les expéditions de votre propre compte |
| AWB | Ne résout que les expéditions des **90 derniers jours** |

## Erreurs fréquentes

| Statut | Message | Comment corriger |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Consulter la planification existante avant d'en créer une autre |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Attendre le statut `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Apporter le colis à un point de dépôt |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Utiliser une date future |
| `400` | `Coleta não pode ser realizada em um final de semana` | Choisir un jour ouvré |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Choisir un autre jour ouvré |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Réduire l'anticipation |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Compléter `number` et `district` (enlèvements au Brésil) |
| `400` | `O atributo federal_tax_id não é válido` | Indiquer un CPF ou un CNPJ valide |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Utiliser un `state_code` du pays de l'expéditeur |
| `400` | `Motivo do cancelamento não é compativel` | Utiliser un code entre `001` et `008` |
| `403` | Sans permission | Demander la permission de planification au support |
| `404` | `Agendamento não foi encontrado para esta remessa` | Vérifier l'identifiant ou créer la planification |
| `422` | Validation | Corriger la date/l'heure/l'instruction selon les formats |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Consultez toujours la planification avant de créer ou d'annuler : cela évite les requêtes redondantes et l'erreur
  `já tem um agendamento em progresso`.
- Attention à la différence de nomenclature : **`initial_time` dans la requête** et **`inicial_time` dans la réponse**.
- Les planifications futures restent en `processing` jusqu'à 1 jour avant l'enlèvement. Ce n'est pas une erreur.
