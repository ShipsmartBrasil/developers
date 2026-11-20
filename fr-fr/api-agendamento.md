# Planification d'Enlèvement

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md) et [Expédition (v1)](api-remessa-v1.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre opération.

## Vue d'ensemble
- Demande au transporteur de **venir chercher** le colis à votre adresse, au lieu de le déposer vous-même dans un point
  de dépôt.
- Permet de consulter l'avancement de la planification et de l'annuler avec un motif standardisé.
- Fonctionne pour les expéditions créées en **v1 et en v2** (les endpoints de planification n'existent qu'en `/v1`).
- La demande est **asynchrone** : l'API enregistre la demande et l'envoi au transporteur a lieu en arrière-
  plan. La confirmation arrive ensuite, lors de la consultation de la planification.
- Chaque expédition accepte **une seule planification** via l'API.

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

> `Content-Type` n'est nécessaire que dans le `POST`. Chaque opération exige une permission propre dans votre profil
> (créer, consulter et annuler une planification). Sans cette permission, la réponse est `403` avec `Has no permission!`.

## Flux d'utilisation

```
1. GET    /v1/shipping/{shipping}/status    → confirmer que le statut ACTUEL est "shipped"
2. POST   /v1/schedule                      → enregistrer la planification (statut "processing")
3. GET    /v1/schedule/{shipping}           → suivre jusqu'au statut "success"
4. DELETE /v1/schedule/{shipping}/{reason}  → annuler, si nécessaire (uniquement avec le statut "success")
```

### Avant de planifier, vérifiez

| # | Condition | Comment vérifier |
|---|---|---|
| 1 | Le statut **actuel** de l'expédition est **`shipped`** (traitée chez le transporteur, documents disponibles). Une fois que l'expédition a avancé (ex. : en transit), la planification n'est plus acceptée | [Suivi](api-rastreio.md) |
| 2 | Le transporteur du contrat de l'expédition **dispose de l'enlèvement intégré** (voir [Transporteurs avec enlèvement intégré](#transporteurs-avec-enlèvement-intégré)) | Contrat choisi dans l'expédition |
| 3 | L'expédition **n'a encore** aucune planification — pas même une planification annulée ou en erreur | `GET /v1/schedule/{shipping}` doit répondre `404` |
| 4 | L'expédition a été créée par le **même utilisateur** que celui du token | - |
| 5 | Votre compte **n'a pas** l'enlèvement automatique activé dans les préréglages (voir [Enlèvement automatique via les préréglages du compte](#enlèvement-automatique-via-les-préréglages-du-compte)) | Tableau de bord du compte |

## Concepts essentiels

### Conventions

| Élément | Format | Exemple |
|---|---|---|
| Date de l'enlèvement | `AAAA-MM-JJ` (exactement ce format) | `2026-03-10` |
| Horaires du créneau | `HH:MM:SS` (avec les secondes) | `08:00:00` |
| Date « actuelle » | Prend en compte le fuseau horaire de la plateforme (heure de Brasília) | - |
| Identifiant `{shipping}` | **UUID de la cotation** ou **numéro de l'AWB** (expéditions créées au cours des 90 derniers jours) | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` |

> Contrairement aux endpoints d'expédition, **la clé de la NF-e n'est pas acceptée** comme identifiant dans la planification.
> Un identifiant contenant un trait d'union (`-`) est toujours traité comme un UUID ; sans trait d'union, il est traité comme un AWB.

### Transporteurs avec enlèvement intégré

| Transporteur | Planification via l'API | Annulation via l'API |
|---|---|---|
| DHL | oui | oui |
| FedEx | oui | oui |
| UPS | oui | oui |
| SkyPostal | oui | oui |
| CTBI Log | oui | **non** (la demande d'annulation se termine en `error` ; contactez le support) |
| GLS | oui | **non** (la demande d'annulation se termine en `error` ; contactez le support) |

> Liste en vigueur à la date de cette version du guide ; elle peut être étendue. Pour tout autre transporteur,
> le `POST` répond `400` avec `Esta transportadora não possui serviço de coleta integrado`.

### Adresse d'enlèvement

L'adresse de retrait est définie de l'une des façons suivantes. **Ne les mélangez pas** : soit vous envoyez `address`, soit vous laissez
la plateforme la déterminer.

| Forme | Comment l'envoyer | Adresse utilisée |
|---|---|---|
| **Forme A — automatique** | Omettez `address` (ou envoyez `null` / `{}`) | 1° le préréglage d'enlèvement de votre compte pour le pays de l'expéditeur (s'il est activé et dispose d'une adresse enregistrée) ; 2° l'adresse de l'expéditeur de l'expédition |
| **Forme B — adresse alternative** | Envoyez l'objet `address` complet | L'adresse indiquée, qui doit se trouver **dans le même pays que l'expéditeur** |

> Pour les comptes de type **personne physique** et **PJ mono-CD**, l'objet `address` est **ignoré** et l'enlèvement utilise
> toujours l'adresse de l'expéditeur de l'expédition. Seul le format des champs (`422`) est vérifié ; les règles
> d'adresse (pays, code postal, document) ne sont pas appliquées.

### Statut de la planification

| `code` | Signification | Étape suivante |
|---|---|---|
| `processing` | Planification enregistrée, en attente d'envoi/de confirmation par le transporteur | Patienter et consulter à nouveau |
| `success` | **Enlèvement confirmé par le transporteur.** `confirmation_code` disponible | Tenir le colis prêt dans le créneau indiqué |
| `canceling` | Annulation demandée, en attente d'envoi au transporteur | Patienter |
| `terminating` | Annulation envoyée au transporteur, en attente de confirmation | Patienter |
| `canceled` | Annulation terminée | - |
| `error` | Échec de la planification ou de l'annulation. Le motif figure dans `status.description` | Lire la description et contacter le support |

```
POST                         DELETE
 │                             │
 ▼                             ▼
processing ──► success ──► canceling ──► terminating ──► canceled
 │                             │              │
 └──► error                    └──► error     └──► error
```

### Type de traitement

La plateforme classe automatiquement la planification selon la date indiquée. Il n'existe aucun champ dans le payload.

| Type | Quand | Comportement |
|---|---|---|
| Immédiat | `date` = demain | Envoyée au transporteur lors du prochain cycle de traitement |
| Programmé | `date` après-demain ou plus tard | Envoyée au transporteur **1 jour avant** la date de l'enlèvement |

> C'est pourquoi une planification programmée reste en `processing` pendant quelques jours. C'est normal.
> Exemple : une demande faite le vendredi pour le lundi suivant est **programmée** (envoyée le dimanche).

### Enlèvement automatique via les préréglages du compte

Si votre compte a l'**enlèvement automatique** activé dans les préréglages (configuré dans le tableau de bord), la
plateforme crée elle-même la planification dès que les documents de l'expédition sont générés, sans appel à l'API :

| Élément | Valeur appliquée |
|---|---|
| Date | Prochain jour ouvré (pour un expéditeur au Brésil, saute également un jour férié national) |
| Créneau | `08:00:00` à `18:00:00` |
| Instruction | L'instruction enregistrée dans le préréglage |
| Adresse | Même règle que la Forme A (préréglage d'enlèvement et, à défaut, l'expéditeur) |

> Dans ce cas, un `POST /v1/schedule` pour la même expédition répond
> `400 Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso`.
> Utilisez uniquement le `GET /v1/schedule/{shipping}` pour suivre.

### Motifs d'annulation

| Code | Motif |
|---|---|
| `001` | Package not ready (colis pas prêt) |
| `002` | Rates too high (tarifs trop élevés) |
| `003` | Transit time too slow (délai de transit trop long) |
| `004` | Take to service center or drop box (sera déposé dans un point de dépôt) |
| `005` | Commitment time not met (délai convenu non respecté) |
| `006` | Reason not given (motif non indiqué) |
| `007` | Other (autre) |
| `008` | Pickup modified (enlèvement modifié) |

> Ces codes sont standardisés par les transporteurs et leur sont transmis sans traduction.
> Utilisez `006` ou `007` lorsqu'aucun des motifs spécifiques ne s'applique. Envoyez toujours les **3 chiffres** (`001`, et non `1`).

## Endpoint : créer une planification

- **Méthode et route :** `POST /v1/schedule`
- **Quand l'utiliser :** lorsque l'expédition a le statut actuel `shipped`, pour demander le retrait du colis.

### Paramètres

| Champ | Type | Obligatoire | Min. | Max. | Format / valeurs | Description |
|---|---|---|---|---|---|---|
| `shipping` | string | **oui** si `awb_number` n'est pas envoyé | - | - | UUID d'une cotation existante | Code de la cotation/expédition |
| `awb_number` | string | **oui** si `shipping` n'est pas envoyé | - | 50 | - | Numéro de l'AWB (expéditions créées au cours des 90 derniers jours) |
| `date` | string | **oui** | - | - | `AAAA-MM-JJ` | Date de l'enlèvement. Voir [Règles de date](#règles-de-date) |
| `initial_time` | string | **oui** | - | - | `HH:MM:SS` | Début du créneau d'enlèvement |
| `final_time` | string | **oui** | - | - | `HH:MM:SS` | Fin du créneau d'enlèvement |
| `instruction` | string | **oui** | 5 | 80 | - | Instruction pour le chauffeur (où retirer, à qui s'adresser) |
| `address` | object \| null | non | - | - | - | Adresse alternative d'enlèvement (Forme B). Voir le tableau ci-dessous |

> Envoyez **`shipping` ou `awb_number`**, pas les deux. Si les deux sont envoyés, `awb_number` prévaut.
>
> L'API **ne** compare **pas** `initial_time` avec `final_time` et ne valide pas les horaires d'ouverture. Assurez-vous que la fin soit
> postérieure au début et que le créneau soit accepté par le transporteur ; sinon, la planification peut
> se terminer en `error`.

### Paramètres — `address` (adresse alternative d'enlèvement)

N'envoyez cet objet que si l'enlèvement a lieu à une adresse **différente** de l'adresse automatique.

**Attention : lorsque `address` est envoyé, les 12 clés ci-dessous doivent toutes être présentes dans l'objet.**
Une clé absente fait échouer la requête avec `500`. Lorsque vous n'avez pas la donnée, utilisez `null` dans les champs qui
l'acceptent et `""` (texte vide) dans `complement` et `phone_extension`.

| Champ | Type | Accepte `null` | Min. | Max. | Règle pour un expéditeur au Brésil | Description |
|---|---|---|---|---|---|---|
| `street` | string | non | 5 | 70 | - | Voie |
| `number` | string | oui | - | 6 | **Obligatoire** (ne peut pas être vide) | Numéro du bâtiment |
| `district` | string | oui | - | 30 | **Obligatoire** (ne peut pas être vide) | Quartier |
| `complement` | string | non (utilisez `""`) | - | 20 | - | Complément |
| `city` | string | non | 5 | 35 | - | Ville |
| `state_code` | string | oui | - | 10 | - | Sigle de l'État. Doit appartenir au pays de l'expéditeur |
| `postal_code` | string | oui | - | 12 | - | CEP/ZIP. Si `state_code` est vide, l'État est déterminé à partir de celui-ci |
| `name` | string | non | 5 | 35 | - | Nom du contact sur place |
| `email` | string | non | - | 100 | - | E-mail du contact (format d'e-mail valide) |
| `phone` | string | non | 10 | 25 | - | Téléphone du contact |
| `phone_extension` | string | non (utilisez `""`) | - | 5 | - | Poste |
| `federal_tax_id` | string | oui | - | 20 | **Obligatoire : CPF ou CNPJ valide** | Document du responsable du lieu |

Comment le pays et l'État sont vérifiés :

| Situation | Comportement |
|---|---|
| `state_code` indiqué | Doit être un État du pays de l'expéditeur (ou le code du pays lui-même). Sinon : `Endereço enviado não pertence ao pais da remessa` |
| `state_code` vide et `postal_code` indiqué (expéditeur BR) | L'État est obtenu à partir du CEP. CEP introuvable : `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| `state_code` vide et `postal_code` indiqué (autres pays) | L'État est obtenu à partir du code postal. Introuvable : `Informações de endereço de coleta insuficientes ou inválidas` |
| `state_code` et `postal_code` vides | Aucune vérification. Nous recommandons de toujours envoyer les deux pour éviter un refus du transporteur |

### Payload minimal

Forme A avec `shipping` (UUID) :

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retrait a la reception"
}
```

Forme A avec `awb_number` :

```json
{
  "awb_number": "1234567890",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retrait a la reception"
}
```

### Payload maximal

Forme B (adresse alternative, expéditeur au Brésil), avec tous les champs acceptés :

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Retrait au hangar 2, portail lateral, demander Carlos",
  "address": {
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Hangar 2",
    "city": "Ville Exemple",
    "state_code": "SP",
    "postal_code": "01000000",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

> Pour utiliser l'AWB, remplacez `shipping` par `awb_number`. Le `federal_tax_id` de l'exemple est fictif : pour
> les expéditions du Brésil, l'API exige un CPF ou un CNPJ valide.

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
    "instruction": "Retrait a la reception"
  }'
```

### Exemple de réponse

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> La réponse confirme uniquement l'**enregistrement** de la demande. La confirmation du transporteur est asynchrone :
> consultez la planification jusqu'à ce que le statut passe à `success` et que le `confirmation_code` apparaisse.

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` lorsque la planification a été enregistrée |
| `message` | string | Message traduit selon le header `lang` |

> Il n'y a pas de `data` dans cette réponse. Pour obtenir des détails, utilisez [consulter une planification](#endpoint--consulter-une-planification).

### Règles de date

Les règles sont vérifiées dans cet ordre ; la première qui échoue définit le message (`400`).

| Règle | Détail | Message en cas d'échec |
|---|---|---|
| Date future | `date` doit être **postérieure à aujourd'hui** (aujourd'hui n'est pas accepté) | `A data de agendamento não pode ser menor que a data Atual` |
| Pas de week-end | Le samedi et le dimanche ne sont pas acceptés, quel que soit le pays | `Coleta não pode ser realizada em um final de semana` |
| Pas de jour férié national | Uniquement lorsque l'expéditeur se trouve au Brésil | `Coleta não pode ser realizada durante um feriado nacional` |
| Limite de 7 jours ouvrés | La date ne peut pas dépasser le 7e jour ouvré après aujourd'hui (pour un expéditeur au Brésil, les jours fériés nationaux ne comptent pas comme jours ouvrés) | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Jours fériés nationaux brésiliens pris en compte (dates fixes) : **01/01, 21/04, 01/05, 07/09, 12/10, 02/11,
15/11, 20/11 et 25/12**. Les jours fériés mobiles (ex. : Carnaval, Vendredi saint, Fête-Dieu) et les jours fériés
des États/municipaux **ne** sont **pas** bloqués par l'API — évitez ces dates de votre propre initiative.

### Ordre des validations

| # | Vérification | Statut | Message |
|---|---|---|---|
| 1 | Format des champs | `422` | Messages par champ dans `data` |
| 2 | AWB trouvé (si `awb_number` a été envoyé) | `400` | `Agendamento não foi encontrado para esta remessa` |
| 3 | Date postérieure à aujourd'hui | `400` | `A data de agendamento não pode ser menor que a data Atual` |
| 4 | Expédition sans planification antérieure | `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` |
| 5 | Expédition existante et avec le statut actuel `shipped` | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 6 | Transporteur avec enlèvement intégré | `400` | `Esta transportadora não possui serviço de coleta integrado` |
| 7 | Règles de date (week-end, jour férié, 7 jours ouvrés) | `400` | Voir [Règles de date](#règles-de-date) |
| 8 | Expédition créée par l'utilisateur du token | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 9 | Adresse alternative (si envoyée) | `400` | Voir [Erreurs courantes](#erreurs-courantes) |

## Endpoint : consulter une planification

- **Méthode et route :** `GET /v1/schedule/{shipping}`
- **Quand l'utiliser :** pour suivre la planification après le `POST` et avant d'annuler.

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `shipping` | path | string | **oui** | UUID de la cotation ou numéro de l'AWB (expéditions des 90 derniers jours) |

### Payload minimal

Sans corps. Uniquement l'identifiant dans le path :

```
GET /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944
```

### Payload maximal

Il n'y a pas d'autres paramètres. L'alternative consiste à utiliser l'AWB à la place de l'UUID :

```
GET /v1/schedule/1234567890
```

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK`

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retrait a la reception",
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
| `data.confirmation_code` | string \| null | Numéro de confirmation de l'enlèvement chez le transporteur. N'est définitif que lorsque `status.code` vaut `success` ; avant cela, il peut être `null` ou une valeur provisoire (ex. : `000000`) |
| `data.date` | string | Date de l'enlèvement (`AAAA-MM-JJ`) |
| `data.inicial_time` | string | Début du créneau (`HH:MM:SS`). ⚠️ **La clé est `inicial_time`** dans la réponse, alors que dans la requête le champ est `initial_time` |
| `data.final_time` | string | Fin du créneau (`HH:MM:SS`) |
| `data.instruction` | string | Instruction indiquée lors de la création (ou celle du préréglage, dans l'enlèvement automatique) |
| `data.status.code` | string | Code du statut de la planification (voir [Statut de la planification](#statut-de-la-planification)). Utilisez ce champ dans votre logique |
| `data.status.description` | string | Texte informatif du dernier événement. **N'est pas traduit dans la consultation** : il peut être dans la langue dans laquelle la demande a été faite, en anglais (ex. : `Pickup Scheduled Successfully!`, `Pickup Cancelled Successfully!`) ou contenir le message renvoyé par le transporteur, en cas d'`error` |
| `data.status.reason` | string \| null | Code du motif d'annulation (`001`–`008`). `null` tant qu'il n'y a pas de demande d'annulation |

## Endpoint : annuler une planification

- **Méthode et route :** `DELETE /v1/schedule/{shipping}/{reason}`
- **Quand l'utiliser :** lorsque l'enlèvement a déjà été **confirmé** (statut `success`) et doit être décommandé.

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|---|
| `shipping` | path | string | **oui** | - | UUID de la cotation ou numéro de l'AWB (expéditions des 90 derniers jours) |
| `reason` | path | string | **oui** | `001` à `008` | Code du motif (voir [Motifs d'annulation](#motifs-dannulation)). Transmis au transporteur |

### Payload minimal

Sans corps. Identifiant et motif dans le path :

```
DELETE /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/006
```

### Payload maximal

Il n'y a pas d'autres paramètres. L'alternative consiste à utiliser l'AWB à la place de l'UUID :

```
DELETE /v1/schedule/1234567890/001
```

### Exemple de requête

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` lorsque la demande d'annulation a été enregistrée |
| `message` | string | Message traduit selon le header `lang` |

> Après la demande, le statut passe à `canceling` avec la description
> `Solicitação de cancelamento recebida! Em breve será processada!`, puis à `terminating` et, enfin, à
> `canceled` (ou `error`). Consultez la planification pour suivre.

### Règles de l'annulation

| Règle | Statut | Message en cas d'échec |
|---|---|---|
| Le `reason` doit être l'un des codes `001` à `008` | `422` | `O campo reason não contém um valor válido.` (dans `data.reason`) |
| L'AWB doit être trouvé (s'il est utilisé dans le path) | `400` | `Agendamento não foi encontrado para esta remessa` |
| La planification doit exister | `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| L'expédition doit toujours avoir le statut actuel `shipped` | `400` | idem |
| La planification doit être en `success` (déjà confirmée). En `processing`, `canceling`, `terminating`, `canceled` ou `error`, l'annulation est refusée | `400` | idem |

## Règles métier

| Règle | Détail |
|---|---|
| Statut de l'expédition | Le statut **actuel** doit être `shipped`, aussi bien pour planifier que pour annuler |
| Planification unique | Chaque expédition accepte une seule planification via l'API. Une planification annulée ou en erreur empêche **également** un nouveau `POST` ; pour replanifier, contactez le support |
| Transporteur | Le service de l'expédition doit disposer de l'enlèvement intégré. Annulation intégrée uniquement pour DHL, FedEx, UPS et SkyPostal |
| Date | Postérieure à aujourd'hui, jour ouvré, hors jour férié national fixe (expéditeur BR), jusqu'à 7 jours ouvrés |
| Créneau | `initial_time` et `final_time` au format `HH:MM:SS` ; il n'y a pas de validation d'ordre ni d'horaires d'ouverture |
| Adresse alternative | Les 12 clés présentes ; au Brésil, exige `number`, `district` et un `federal_tax_id` valide (CPF ou CNPJ) |
| Pays de l'adresse | Toujours le pays de l'expéditeur de l'expédition. Le `state_code` doit appartenir à ce pays |
| Type de compte | Personne physique et PJ mono-CD : `address` ignoré ; enlèvement à l'adresse de l'expéditeur |
| Enlèvement automatique | Lorsque le préréglage d'enlèvement automatique est actif, la planification est créée par la plateforme et le `POST` manuel est refusé |
| Propriété | Planifiez et consultez uniquement les expéditions créées par l'utilisateur du token |
| AWB | Résout uniquement les expéditions créées au cours des **90 derniers jours** (la plus récente avec ce numéro) |
| Traitement | Immédiat pour un enlèvement le lendemain ; programmé (envoyé 1 jour avant) pour les dates ultérieures |

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Une planification existe déjà (manuelle, automatique, annulée ou en erreur). Consultez-la avec `GET` ; pour replanifier, contactez le support |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Vérifier que le statut actuel est `shipped` et que l'expédition appartient à l'utilisateur du token |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Déposer le colis dans un point de dépôt |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Utiliser une date à partir de demain |
| `400` | `Coleta não pode ser realizada em um final de semana` | Choisir un jour ouvré |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Choisir un autre jour ouvré |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Réduire l'anticipation |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Expéditeur BR : renseigner `number` et `district`. Autres pays : vérifier `postal_code` ou envoyer `state_code` |
| `400` | `O atributo federal_tax_id não é válido` | Expéditeur BR : indiquer un CPF ou un CNPJ valide |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Utiliser un `state_code` du pays de l'expéditeur |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Vérifier le `postal_code` ou indiquer le `state_code` |
| `400` | `Agendamento não foi encontrado para esta remessa` | Dans le `POST`/`DELETE` avec AWB : vérifier le numéro, l'ancienneté de l'expédition (90 jours) et qu'elle appartient bien à votre compte |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | N'annulez que des planifications en `success` d'expéditions en `shipped` |
| `403` | `Has no permission!` | Demander au support la permission de l'opération (créer, consulter ou annuler une planification) |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contacter le support |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Vérifier le header `Authorization: Bearer ...` |
| `404` | `Agendamento não foi encontrado para esta remessa` | Dans le `GET` avec UUID : l'expédition n'a pas de planification ou n'appartient pas à votre compte |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + table des champs dans `data` | Corriger les champs listés (ex. : `O campo data é obrigatório.`, `A data informada para o campo :attribute não respeita o formato :format.`, `O valor selecionado para o campo shipping é inválido.`, `O campo reason não contém um valor válido.`) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | `POST` : vérifier que toutes les clés de `address` ont été envoyées. `GET` avec AWB : l'AWB n'a pas été trouvé (inexistant, de plus de 90 jours ou d'un autre compte). Dans les autres cas, réessayer avec backoff et contacter le support |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Échec inattendu dans le `DELETE`. Réessayer avec backoff ; si le problème persiste, contacter le support |

> Attention : dans la **consultation par AWB** (`GET /v1/schedule/{awb}`), un AWB introuvable répond `500`, et non
> `404`. Préférez consulter par l'UUID de la cotation, qui répond `404` lorsqu'il n'y a pas de planification.

## Remarques finales
- Consultez toujours la planification avant de la créer ou de l'annuler : cela évite les requêtes redondantes et l'erreur
  `já tem um agendamento em progresso`.
- Attention à la différence de nomenclature : **`initial_time` dans la requête** et **`inicial_time` dans la réponse**.
- Les planifications programmées restent en `processing` jusqu'à 1 jour avant l'enlèvement. Ce n'est pas une erreur. Si le statut
  ne change pas après la date de l'enlèvement, contactez le support.
- Utilisez `status.code` (et non `status.description`) pour la logique de votre intégration.
- Statut de l'expédition : [Suivi](api-rastreio.md). Documents pour l'enlèvement : [Documents](api-documentos.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
