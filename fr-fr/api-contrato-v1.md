# Contrat

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Liste les contrats de transport disponibles pour l'utilisateur authentifié.
- Un « contrat » est la combinaison **transporteur + service + conditions commerciales** de votre compte.
  C'est ce que la cotation retourne dans `carriers[]`.
- Utilisez cet endpoint pour le diagnostic et pour savoir à l'avance quels transporteurs peuvent apparaître
  dans vos cotations.

> Vous **n'avez pas besoin** d'appeler cet endpoint dans le flux d'intégration normal. La cotation renvoie déjà les
> contrats éligibles pour cet envoi précis, avec le prix et le délai.

## Base URL
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
lang: pt
```

## Quand l'utiliser

| Situation | En quoi cela aide |
|---|---|
| La cotation retourne `Nenhum transportador encontrado para este usuário!` | Confirme si le compte a au moins un contrat actif |
| La cotation retourne `Nenhum contrato disponível para este usuário!` | Confirme le rattachement des contrats |
| Vous voulez montrer à l'utilisateur les transporteurs activés | Liste les noms et les périodes de validité |
| Vous avez reçu `Contrato especificado não foi listado para esta cotação!` | Aide à vérifier le `code`/`slug` attendu |

## Endpoint : lister les contrats

- **Méthode et route :** `GET /v1/contract`

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Valeurs | Description |
|---|---|---|---|---|---|---|
| `status` | query | string | non | `active` | `active`, `inactive`, `all` | Filtre selon la validité du contrat. Les valeurs inconnues retombent sur la valeur par défaut `active` |
| `page` | query | integer | non | `1` | - | Page de la liste |

- **Éléments par page :** 15

### Exemple de requête

```bash
# Contrats actifs (par défaut)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'

# Tous les contrats, actifs et inactifs
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contrat standard PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "DHL",
        "initial_date": "2026-01-06 00:00:00",
        "final_date": "2026-12-31 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `code` | integer | **Code du contrat.** C'est la même valeur que celle qui apparaît dans `carriers[].code` de la cotation et que vous envoyez dans `carrier_code` de l'expédition |
| `contract_description` | string | Description commerciale du contrat |
| `contract_name` | string | Nom du contrat |
| `country` | string | Pays d'origine du contrat (ISO 3166-1 alpha-2) |
| `carrier` | string | Nom du transporteur |
| `initial_date` | string | Début de la validité (`Y-m-d H:i:s`) |
| `final_date` | string | Fin de la validité (`Y-m-d H:i:s`) |
| `active` | string | `active` ou `inactive` |
| `direct_quotation` | boolean | `true` quand le contrat cote directement chez le transporteur (en temps réel), au lieu d'utiliser une table de fret |

## Relation avec la cotation et l'expédition

```
GET /v1/contract            → contrats activés sur le compte (vue d'ensemble)
        │
POST /v1/quotation          → carriers[] : contrats éligibles POUR CET ENVOI
        │                     (avec code, slug, prix, délai et valid_until)
        ▼
POST /v1/shipping           → carrier_code (ou slug) choisi
```

| Point d'attention | Explication |
|---|---|
| La cotation peut retourner **moins** de contrats que cet endpoint | Le contrat existe, mais il ne couvre pas ce poids/cette dimension/cette route |
| L'expédition n'accepte **que** les contrats listés dans la cotation | Envoyer un `carrier_code` actif mais absent de la cotation retourne `404 Contrato especificado não foi listado para esta cotação!` |
| Le `slug` n'apparaît pas dans cette liste | Utilisez le `slug` retourné dans `carriers[]` de la cotation |

## Règles métier
- Le retour dépend du **rattachement de votre utilisateur** aux contrats.
- Si aucun contrat n'est trouvé avec le filtre indiqué, la plateforme effectue une seconde recherche
  en tenant compte des contrats par défaut du compte.
- Les contrats `inactive` (hors validité) n'apparaissent **pas** dans les cotations.
- `direct_quotation: true` indique une cotation en temps réel chez le transporteur. Ces contrats peuvent mettre
  un peu plus de temps à répondre et dépendent d'identifiants valides
  (`Credenciais inválidas para cotação direta`).

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `403` | Sans permission de lister les contrats | Demander la permission au support |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Cet endpoint est en **lecture seule** : les contrats sont configurés commercialement, pas via l'API.
- Utilisez-le comme outil de diagnostic quand la cotation ne retourne aucun transporteur.
- Pour comprendre pourquoi un contrat précis n'est pas apparu dans la cotation, voir
  **Restrictions de charge** dans [Cotation (v1)](api-cotacao-v1.md#restrictions-de-charge).
