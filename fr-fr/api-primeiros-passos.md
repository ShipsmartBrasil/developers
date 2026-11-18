# Premiers Pas

- [Retour au README de la langue](README.md)

## Vue d'ensemble
Cette page rassemble tout ce qui est commun à **tous** les endpoints de l'API Publique de Shipsmart Global :
environnements, authentification, langue, format de réponse, versionnement et traitement des erreurs.

Lisez cette page avant toute autre. Les autres pages du guide supposent que vous connaissez déjà
ces conventions et ne les répètent pas en détail.

## Environnements

| Environnement | Base URL | Documentation interactive |
|---|---|---|
| Production | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Préproduction (staging) | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

- Utilisez la **préproduction** pour développer et tester l'intégration. Aucune expédition créée là-bas n'est réellement envoyée.
- Les nouvelles fonctionnalités apparaissent d'abord en préproduction. Si un champ décrit ici n'existe pas encore en production,
  vérifiez sur `https://hmlapi.shipsmart.com.br/doc`.
- L'accès à la racine (`/`) redirige vers `/doc`.

## Authentification

Tout appel (sauf la documentation) exige un token d'accès personnel dans le header `Authorization`,
selon le schéma **Bearer**.

```
Authorization: Bearer VOTRE_TOKEN
```

| Élément | Valeur |
|---|---|
| Type | Bearer token |
| Header | `Authorization` |
| Où l'obtenir | Portail Shipsmart Global, dans la zone de configuration de votre compte |
| Expiration | Le token n'expire pas avec le temps ; il est révoqué si le compte est bloqué ou si le token est régénéré |

### Erreurs d'authentification

| Statut | Réponse | Signification |
|---|---|---|
| `404` | `{"status":"warning","message":"..."}` | Token non trouvé/invalide |
| `403` | `{"status":"warning","message":"..."}` | Compte bloqué |
| `403` | `{"status":"warning","message":"Has no permission!"}` | Le token est valide, mais votre profil n'a pas la permission pour l'action |

> Les permissions sont granulaires par action (coter, expédier, annuler, lister les documents, planifier un enlèvement, etc.).
> Si vous recevez `Has no permission!`, demandez au support l'activation de la permission correspondante.

## Langue des réponses

Les messages de succès, d'avertissement et d'erreur sont traduits selon le header `lang`.

| Header | Valeurs acceptées | Par défaut |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> Le header est optionnel. S'il n'est pas envoyé (ou envoyé avec une valeur inconnue), l'API répond en portugais.

> **À propos des exemples de ce guide :** tous les messages de réponse reproduits dans les pages suivantes
> utilisent la langue par défaut de l'API (`pt`). En envoyant `lang: en` ou `lang: es`, le texte de `message`
> est traduit, mais le `status` et le code HTTP restent toujours les mêmes.

## Conventions de requête

| Élément | Valeur |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Encodage | UTF-8 |
| Dates (réponse) | `Y-m-d H:i:s` |
| Dates (planification) | `Y-m-d` pour la date et `H:i:s` pour l'heure |
| Devises | Code ISO 4217 à 3 lettres (ex. : `BRL`, `USD`, `EUR`) |
| Pays | Code ISO 3166-1 alpha-2 à 2 lettres (ex. : `BR`, `US`, `PT`) |
| Séparateur décimal | Point (`.`) — jamais la virgule |
| Poids | Kilogrammes quand `measurement` vaut `metric` ; livres quand il vaut `imperial` |
| Dimensions | Centimètres quand `measurement` vaut `metric` ; pouces quand il vaut `imperial` |
| Volume | m³ quand `measurement` vaut `metric` ; ft³ quand il vaut `imperial` |

## Versionnement

La version fait partie du path : `/v1/...` et `/v2/...`.

- `POST /v1/quotation` → cotation dans la première version
- `POST /v2/quotation` → cotation dans la deuxième version

### Appels sans préfixe de version
Si vous appelez une ressource **sans** le préfixe de version, l'API répond `308 Permanent Redirect`
en pointant vers la même ressource sur `/v1`, en préservant la query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> Pour les nouvelles intégrations, indiquez toujours la version explicitement. Ne dépendez pas de la redirection.

### Compatibilité entre versions
Une cotation créée en v1 ne peut générer une expédition **que** en v1, et une cotation créée en v2 ne peut générer
une expédition **que** en v2. Mélanger les versions retourne `422` avec le message
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

Voir [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md) pour choisir la version.

## Format standard des réponses

Toutes les réponses suivent la même enveloppe :

```json
{
  "status": "success",
  "message": "Message lisible par un humain",
  "data": {}
}
```

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` ou `error` |
| `message` | string | Message déjà traduit selon le header `lang` |
| `data` | object/array | Contenu de la réponse. Absent dans les réponses qui ne confirment que la réception |

### Relation entre `status` et HTTP

| `status` | HTTP typique | Lecture |
|---|---|---|
| `success` | `200` | L'opération a été acceptée/terminée |
| `warning` | `400`, `403`, `404`, `406`, `422` | Une règle métier ou une validation a empêché l'opération. **Le payload doit être corrigé** |
| `error` | `500` | Échec inattendu côté serveur. Réessayez ; si le problème persiste, contactez le support |

## Erreurs de validation (`422`)

Quand un champ échoue à la validation, la réponse contient `data` avec la table `champ → liste de messages`.

```json
{
  "status": "warning",
  "message": "Validation Error",
  "data": {
    "object": ["O campo object é obrigatório."],
    "address_sender.country_code": [
      "O campo address sender.country code é obrigatório quando address sender.address code não está presente."
    ],
    "boxes": ["O campo boxes é obrigatório a menos que type esteja presente em items."]
  }
}
```

> Pour les champs imbriqués, la clé utilise le point (`address_sender.country_code`) et, dans les tableaux,
> elle utilise l'index (`items.0.hscode`). Utilisez cela pour pointer l'erreur dans votre formulaire/intégration.

## Tableau général des statuts HTTP

| Statut | Quand cela arrive | Que faire |
|---|---|---|
| `200` | Requête acceptée/traitée | Poursuivre le flux |
| `400` | Une règle métier a bloqué l'opération (ex. : type de paiement inexistant, cotation expirée, aucun transporteur éligible) | Lire `message` et corriger le scénario |
| `403` | Sans permission ou compte bloqué | Demander l'activation au support |
| `404` | Ressource non trouvée (token, expédition, contrat non listé dans la cotation) | Vérifier l'identifiant fourni |
| `406` | Structure du payload incompatible avec ce qui est attendu | Revoir les types et l'imbrication des champs |
| `422` | Échec de validation des champs | Corriger les champs listés dans `data` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Préréglages du compte (presets)

Certaines options peuvent être **préconfigurées sur votre compte** dans le portail. Lorsqu'elles sont configurées, elles sont
appliquées automatiquement à la cotation et à l'expédition, et vous pouvez omettre le champ dans le payload.

| Préréglage | Affecte | Remarque |
|---|---|---|
| Canal de vente (`channel_code`) | Cotation | Utilisé quand vous n'indiquez pas le canal |
| Taxe (`tax`) | Cotation | `sender` ou `receiver` |
| Assurance (`insurance`) | Cotation | - |
| Objet (`object`) et type (`type`) | Cotation | - |
| Devises (`currency_quote`, `currency_payment`) | Cotation | - |
| Système de mesure (`measurement`) | Cotation | `metric` ou `imperial` |
| Livraison résidentielle / non empilable | Cotation | - |
| Adresse de l'expéditeur | Cotation et expédition | Appliquée comme `address_code` |
| Fret déclaré (`declared_freight`) | Cotation | Valeur + devise |
| Facturant (`biller`) | Expédition | Adresse affichée sur l'invoice |
| Regroupement de boîtes | Cotation | `default`, `all_in_one` ou `smartbox` |

**Précédence :** ce que vous envoyez dans le payload gagne toujours sur le préréglage. Exceptions :
- Si vous indiquez `address_sender.country_code` dans la cotation, le préréglage d'adresse de l'expéditeur est ignoré.
- Si vous indiquez `address_sender.street` dans l'expédition, le préréglage d'adresse de l'expéditeur est ignoré.
- Si vous indiquez `declared_freight`, le préréglage de fret déclaré est ignoré.

### Regroupement de boîtes
Le regroupement définit comment les articles sont répartis dans les boîtes quand vous utilisez `type: items`.
Ce **n'est pas** un champ du payload : c'est une configuration de votre compte.

| Mode | Comportement |
|---|---|
| `default` | Regroupement standard de la plateforme |
| `all_in_one` | Consolide tout dans une seule boîte |
| `smartbox` | Place les articles dans les boîtes enregistrées en tenant compte du volume et des dimensions (avec 10 % de marge) et en respectant `is_groupable` de l'article |

## Devises acceptées

Même liste pour `currency_quote` (devise de la cotation) et `currency_payment` (devise du paiement) :

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> `currency_payment` avec paiement via gateway a une disponibilité qui dépend du prestataire :
> `BRL` est traité par un prestataire et les autres devises par un autre. Si la devise choisie n'est pas
> activée pour votre compte, l'expédition retourne `warning` au moment du paiement.

## Premier appel (test de connectivité)

L'endpoint le plus simple pour valider le token et la connectivité :

```bash
curl --request GET \
  --url https://hmlapi.shipsmart.com.br/v1/platform/version \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Accept: application/json'
```

Réponse :

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "staging",
    "version": "1.0.0"
  }
}
```

## Bonnes pratiques d'intégration

1. **Conservez toujours le `quotation`** retourné par la cotation : c'est la clé de tout le reste du flux
   (expédition, statut, documents, planification).
2. **Enregistrez aussi votre identifiant commercial** dans `channel_identifier_number`. Cela permet de consulter
   le statut, les documents et d'annuler en utilisant le numéro de votre commande, sans dépendre du `quotation`.
3. **Ne faites jamais de polling agressif.** Les consultations de statut/documents doivent avoir un intervalle de quelques
   secondes et un backoff exponentiel.
4. **Traitez `warning` comme une erreur métier** (ne répétez pas la même requête) et `error` comme transitoire
   (vous pouvez répéter).
5. **Respectez les limites de taille des champs** décrites sur chaque page. Les textes plus longs sont rejetés
   avec `422`, pas tronqués.
6. **Les cotations expirent.** Créez l'expédition dans le délai de validité indiqué dans `valid_until` de chaque transporteur.

## Prochaines étapes

- [Cotation](api-cotacao-v1.md) ou [Cotation V2](api-cotacao-v2.md)
- [Expédition](api-remessa-v1.md) ou [Expédition V2](api-remessa-v2.md)
- [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md)
