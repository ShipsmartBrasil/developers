# Contrat

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Liste les contrats de transport disponibles pour l'utilisateur authentifié, de manière paginée.
- Un « contrat » est la combinaison **transporteur + service + conditions commerciales** de votre compte.
  C'est ce que la cotation retourne dans `carriers[]`.
- Utilisez cet endpoint pour le diagnostic et pour savoir à l'avance quels transporteurs peuvent apparaître
  dans vos cotations.
- Lecture seule : les contrats sont configurés commercialement, pas via l'API.

> Vous **n'avez pas besoin** d'appeler cet endpoint dans le flux d'intégration normal. La cotation renvoie déjà les
> contrats éligibles pour cet envoi précis, avec le prix et le délai.

## URL de base
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
lang: pt
```

> Votre profil doit disposer de la permission de lister les contrats. Sans elle, la réponse est `403` avec `Has no permission!`.

## Flux d'utilisation

```
1. GET /v1/contract                 → contrats actifs et en vigueur (par défaut)
2. GET /v1/contract?status=all      → inclure aussi les contrats désactivés
3. GET /v1/contract?status=all&page=2 → pages suivantes (répétez le status à chaque page)
```

### Quand l'utiliser

| Situation | En quoi cela aide |
|---|---|
| La cotation retourne `Nenhum transportador encontrado para este usuário!` | Confirme si le compte possède au moins un contrat actif et en vigueur |
| La cotation retourne `Nenhum contrato disponível para este usuário!` | Confirme le rattachement des contrats |
| Vous voulez montrer à l'utilisateur les transporteurs activés | Liste les noms et les périodes de validité |
| Vous avez reçu `Contrato especificado não foi listado para esta cotação!` | Aide à vérifier le `code` attendu |

## Concepts essentiels

| Concept | Signification |
|---|---|
| Contrat rattaché | Contrat associé directement à votre compte. Si votre utilisateur est rattaché à un compte principal, ce sont les contrats du compte principal qui s'appliquent |
| Contrat par défaut | Contrat par défaut de la plateforme, utilisé quand le compte n'a aucun contrat rattaché |
| Validité | Période entre `initial_date` et `final_date`. **Seuls les contrats en vigueur aujourd'hui sont listés**, quel que soit le filtre `status` |
| Actif / inactif | `active` = contrat activé ; `inactive` = contrat désactivé (encore dans sa période de validité). Cela **ne** concerne **pas** la validité |

## Endpoint : lister les contrats

- **Méthode et route :** `GET /v1/contract`
- **Quand l'utiliser :** diagnostic d'une cotation sans transporteurs ou affichage des transporteurs activés.

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Par défaut | Valeurs | Description |
|---|---|---|---|---|---|---|
| `status` | query | string | non | `active` | `active`, `inactive`, `all` | Filtre selon la situation du contrat (activé/désactivé). Les valeurs vides ou inconnues retombent sur la valeur par défaut `active`, sans erreur |
| `page` | query | integer | non | `1` | entier ≥ 1 | Page de la liste. Les valeurs invalides sont traitées comme `1` |

- **Éléments par page :** 15 (fixe ; il n'existe aucun paramètre pour le modifier).
- **Tri :** non garanti. Ne dépendez pas de l'ordre des éléments.

### Payload minimal

Sans corps et sans query string :

```
GET /v1/contract
```

### Payload maximal

Tous les paramètres acceptés :

```
GET /v1/contract?status=all&page=2
```

### Exemple de requête

```bash
# Contrats actifs (par défaut)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'

# Tous les contrats, actifs et inactifs, page 2
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all&page=2' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'lang: pt'
```

### Exemple de réponse

`200 OK`

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
        "direct_quotation": false,
        "active": "active"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/contract?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Champs de la réponse

Éléments de `data.data[]` :

| Champ | Type | Description |
|---|---|---|
| `code` | integer | **Code du contrat.** C'est la même valeur que celle qui apparaît dans `carriers[].code` de la cotation et que vous envoyez dans `carrier_code` de l'expédition |
| `contract_description` | string \| null | Description commerciale du contrat |
| `contract_name` | string | Nom du contrat |
| `country` | string | Pays d'origine du contrat (ISO 3166-1 alpha-2) |
| `carrier` | string \| null | Nom du service du transporteur |
| `initial_date` | string | Début de la validité (`AAAA-MM-JJ HH:MM:SS`) |
| `final_date` | string | Fin de la validité (`AAAA-MM-JJ HH:MM:SS`) |
| `direct_quotation` | boolean | `true` quand le contrat cote directement chez le transporteur (en temps réel), au lieu d'utiliser une table de fret |
| `active` | string | `active` (activé) ou `inactive` (désactivé) |

Pagination (dans `data`) :

| Champ | Type | Description |
|---|---|---|
| `current_page` | integer | Page actuelle |
| `from` / `to` | integer \| null | Position du premier et du dernier élément de la page (`null` quand la page est vide) |
| `total` | integer | Nombre total de contrats trouvés |
| `per_page` | integer | Éléments par page (`15`) |
| `last_page` | integer | Numéro de la dernière page |
| `first_page_url` / `last_page_url` | string | URLs de la première et de la dernière page |
| `prev_page_url` / `next_page_url` | string \| null | URLs de la page précédente et de la suivante (`null` quand elles n'existent pas) |
| `path` | string | URL de base de la liste |
| `links` | array | Liens de navigation (`url`, `label`, `active`). Le `label` est traduit selon le header `lang` |

> ⚠️ Les URLs de pagination **ne conservent pas le filtre `status`**. Lorsque vous naviguez avec `status=inactive` ou
> `status=all`, construisez vous-même l'URL de la page suivante (`?status=all&page=N`) au lieu de suivre
> `next_page_url` directement.

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
| La cotation peut retourner **moins** de contrats que cet endpoint | La cotation ne prend en compte que les contrats actifs du pays d'origine de l'envoi, disposant d'une table de fret pour le type d'objet (document ou non-document) et qui couvrent ce poids/cette dimension/cette route |
| L'expédition n'accepte **que** les contrats listés dans la cotation | Envoyer un `carrier_code` actif mais absent de la cotation retourne `404 Contrato especificado não foi listado para esta cotação!` |
| Le `slug` n'apparaît pas dans cette liste | Utilisez le `slug` retourné dans `carriers[]` de la cotation |

## Règles métier

| Règle | Détail |
|---|---|
| Rattachement | Le retour dépend des contrats rattachés à votre compte (ou au compte principal, pour les utilisateurs rattachés) |
| Contrats par défaut | Si la recherche ne trouve aucun contrat rattaché **sur la page demandée**, la plateforme renvoie les contrats par défaut de la plateforme avec le même filtre. C'est pourquoi demander une page au-delà de la dernière peut renvoyer des contrats par défaut |
| Validité | Les contrats hors de leur période de validité (avant `initial_date` ou après `final_date`) ne sont jamais listés |
| Filtre `status` | `active` (par défaut), `inactive` ou `all`. Toute autre valeur est traitée comme `active` |
| Cotation | Les contrats `inactive` (désactivés) n'apparaissent **pas** dans les cotations |
| Cotation directe | `direct_quotation: true` indique une cotation en temps réel chez le transporteur. Ces contrats peuvent mettre un peu plus de temps à répondre et dépendent d'identifiants valides (`Credenciais inválidas para cotação direta`) |

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `403` | `Has no permission!` | Demander au support la permission de lister les contrats |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contacter le support |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Vérifier le header `Authorization: Bearer ...` |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Réessayer avec backoff ; si cela persiste, contacter le support |

> Les filtres invalides **ne** génèrent **pas** d'erreur : un `status` inconnu devient `active` et une `page` invalide devient `1`.
> Une liste vide (`data.data: []`, `total: 0`) signifie qu'il n'existe ni contrat en vigueur ni contrat par défaut
> pour ce filtre.

## Remarques finales
- Cet endpoint est en **lecture seule** : les contrats sont configurés commercialement, pas via l'API.
- Utilisez-le comme outil de diagnostic quand la cotation ne retourne aucun transporteur.
- Pour comprendre pourquoi un contrat précis n'est pas apparu dans la cotation, voir
  **Restrictions de charge** dans [Cotation (v1)](api-cotacao-v1.md#restrictions-de-chargement).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
