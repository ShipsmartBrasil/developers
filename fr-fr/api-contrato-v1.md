# Contrat

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Liste les contrats disponibles pour l'utilisateur authentifié.
- Les contrats sont utilisés dans le devis et dans la création d'expéditions.
## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue optionnel : `lang`.

## Conventions
- `Content-Type: application/json`
- Le listage peut filtrer par `status`.

## Flux d'utilisation
1. Authentifiez l'utilisateur.
2. Listez les contrats disponibles.
3. Utilisez le `code` du contrat dans le devis ou l'expédition.

## Endpoints
### Lister les contrats
- Méthode et route : `GET /v1/contract`
- Quand l'utiliser : pour identifier les contrats actifs/inactifs avant le devis.

## Paramètres
| Nom | Emplacement | Type | Obligatoire | Description | Exemple | Observations |
|---|---|---:|---:|---|---|---|
| `status` | query | string | non | Filtre les contrats | `active` | `active`, `inactive`, `all` |

## Payload minimum et maximum
- Ne s'applique pas à l'endpoint de listage.

## Exemple de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=active' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Exemple de response
```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contrato padrão PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "Carrier Name",
        "initial_date": "2023-03-06 00:00:00",
        "final_date": "2023-06-30 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ]
  }
}
```

## Règles métier
- S'il n'y a pas de contrats dans le premier filtre, l'API tente une deuxième recherche avec un comportement alternatif du backend.
- Le retour dépend de l'association de l'utilisateur aux contrats.

## Erreurs courantes
| Status | Signification | Comment corriger |
|---|---|---|
| `500` | Erreur inattendue | Contacter l'équipe responsable |
