# Planification de collecte

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Crée, consulte et annule des planifications de collecte liées à l'expédition.
- La collecte peut être effectuée par devis/expédition ou par AWB.
- Associé au flux opérationnel principal.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Conventions
- Dates au format `Y-m-d`.
- Horaires au format `H:i:s`.
- La planification doit respecter le pays de l'expéditeur.

## Flux d'utilisation
1. Assurez-vous que l'expédition est prête pour la planification.
2. Créez la collecte avec une fenêtre horaire et une instruction.
3. Consultez le statut de la planification.
4. Si nécessaire, annulez en utilisant l'un des motifs autorisés.

## Endpoints
### Créer une planification
- Méthode et route : `POST /v1/schedule`
- Règles importantes : renseignez `shipping` ou `awb_number`.

### Consulter une planification
- Méthode et route : `GET /v1/schedule/{shipping}`

### Annuler une planification
- Méthode et route : `DELETE /v1/schedule/{shipping}/{reason}`

### Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `shipping` | body/path | string | conditionnel | UUID du devis/de l'expédition | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Obligatoire si `awb_number` n'est pas fourni |
| `awb_number` | body | string | conditionnel | AWB | `271249639538` | Obligatoire si `shipping` n'est pas fourni |
| `date` | body | date | oui | Date de la collecte | `2023-11-21` | Format `Y-m-d` |
| `initial_time` | body | time | oui | Début de la fenêtre | `08:00:00` | Format `H:i:s` |
| `final_time` | body | time | oui | Fin de la fenêtre | `18:00:00` | Format `H:i:s` |
| `instruction` | body | string | oui | Instruction pour la collecte | `Deixar na portaria` | 5 à 80 caractères |
| `address` | body | object | non | Adresse alternative | - | Doit être du même pays que l'expéditeur |
| `reason` | path | string | oui | Motif de l'annulation | `001` | Code fixe |

### Motifs d'annulation
| Code | Signification |
|---|---|
| `001` | Package not ready |
| `002` | Rates too high |
| `003` | Transit time too slow |
| `004` | Take to service center or drop box |
| `005` | Commitment time not met |
| `006` | Reason not given |
| `007` | Other |
| `008` | Pickup modified |

### Exemple de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2023-11-21",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Deixar na portaria"
  }'
```

### Exemple de response
```json
{
  "status": "success",
  "message": "Agendamento de coleta registrado com sucesso"
}
```

## Règles métier
- Il n'est possible de planifier que si l'expédition est prête pour cela.
- Il peut déjà y avoir une planification en cours, et dans ce cas l'API renvoie warning.
- L'adresse alternative doit respecter le même pays que l'expéditeur.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `400` | La planification existe déjà ou l'expédition n'est pas prête | Ajuster le flux | 
| `422` | Validation | Corriger la date, l'heure ou les champs obligatoires |
| `500` | Erreur inattendue | Contacter l'équipe responsable |

## Observations finales
- Consultez le statut avant d'annuler afin d'éviter des demandes redondantes.
