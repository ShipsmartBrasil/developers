# Premiers Pas

- [Retour au README de la langue](README.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
Cette page rassemble tout ce qui est commun à **tous** les endpoints de l'API Publique de Shipsmart Global :
environnements, authentification, langue, format de réponse, versionnement et traitement des erreurs.

Lisez cette page avant toute autre. Les autres pages du guide supposent que vous connaissez déjà
ces conventions et ne les répètent pas en détail.

## Environnements

| Environnement | URL de base | Documentation interactive |
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
| Où l'obtenir | Panneau de Shipsmart Global, dans la zone de paramètres de votre compte |
| Expiration | Le token n'expire pas avec le temps ; il est révoqué si le compte est bloqué ou si le token est régénéré |

### Erreurs d'authentification

| Statut | Réponse | Signification |
|---|---|---|
| `404` | `{"status":"warning","message":"Nenhum usuário encontrado para o token especificado!"}` | Token absent, introuvable ou invalide |
| `403` | `{"status":"warning","message":"Usuário especificado encontra-se bloqueado em nosso sistema!"}` | Compte bloqué |
| `403` | `{"status":"warning","message":"Has no permission!"}` | Le token est valide, mais votre profil n'a pas la permission pour l'action |

> Les permissions sont granulaires par action (coter, expédier, annuler, lister les documents, planifier un enlèvement, etc.).
> Si vous recevez `Has no permission!`, demandez au support l'activation de la permission correspondante.

## Langue des réponses

Les messages de succès, d'avertissement et d'erreur sont traduits selon le header `lang`.

| Header | Valeurs acceptées | Défaut |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> Le header est optionnel. S'il n'est pas envoyé, l'API répond en portugais. S'il est envoyé avec une valeur
> non prise en charge (ex. : `fr`), les messages arrivent en anglais.

> **À propos des exemples de ce guide :** tous les messages de réponse reproduits dans les pages suivantes
> utilisent la langue par défaut de l'API (`pt`). En envoyant `lang: en` ou `lang: es`, le texte de `message` arrive
> traduit, mais le `status` et le code HTTP sont toujours les mêmes.

## Conventions de requête

| Élément | Valeur |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Encodage | UTF-8 |
| Dates (réponse) | `AAAA-MM-JJ HH:MM:SS` |
| Dates (planification) | `AAAA-MM-JJ` pour la date et `HH:MM:SS` pour l'heure |
| Devises | Code ISO 4217 à 3 lettres (ex. : `BRL`, `USD`, `EUR`) |
| Pays | Code ISO 3166-1 alpha-2 à 2 lettres (ex. : `BR`, `US`, `PT`) |
| Séparateur décimal | Point (`.`) — jamais de virgule |
| Poids | Kilogrammes lorsque `measurement` vaut `metric` ; livres lorsqu'il vaut `imperial` |
| Dimensions | Centimètres lorsque `measurement` vaut `metric` ; pouces lorsqu'il vaut `imperial` |
| Volume | m³ lorsque `measurement` vaut `metric` ; ft³ lorsqu'il vaut `imperial` |

## Versionnement

La version fait partie du path : `/v1/...` et `/v2/...`.

- `POST /v1/quotation` → cotation dans la première version
- `POST /v2/quotation` → cotation dans la deuxième version

### Appels sans préfixe de version
Si vous appelez une ressource **sans** le préfixe de version, l'API répond `308 Permanent Redirect`
en pointant vers la même ressource en `/v1`, en préservant la query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> Pour les nouvelles intégrations, indiquez toujours la version explicitement. Ne dépendez pas de la redirection.

### Compatibilité entre versions
Une cotation créée en v1 ne peut générer une expédition **qu'**en v1, et une cotation créée en v2 ne peut générer
une expédition **qu'**en v2. Mélanger les versions renvoie `422` avec le message (dans `data.quotation`)
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

Consultez [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md) pour choisir la version.

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
| `data` | object/array | Contenu de la réponse. Absent dans les réponses qui ne font que confirmer la réception |

### Relation entre `status` et HTTP

| `status` | HTTP typique | Interprétation |
|---|---|---|
| `success` | `200` | L'opération a été acceptée/terminée |
| `warning` | `400`, `401`, `403`, `404`, `406`, `422` | Une règle métier ou une validation a empêché l'opération. **Le payload doit être corrigé** |
| `error` | `500` (et certains refus de poids/transporteur avec `400`/`404`) | Échec inattendu ou refus du transporteur. Lisez `message` ; les erreurs `500` peuvent être réessayées |

## Erreurs de validation (`422`)

Lorsqu'un champ ne passe pas la validation, la réponse contient `data` avec la table `champ → liste de messages`.
Le `message` de premier niveau est **générique** (il ne décrit pas l'erreur) : utilisez toujours le contenu de `data`.

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "object": ["O campo object é obrigatório."],
    "address_sender.country_code": [
      "O campo address sender.country code é obrigatório quando address sender.address code não está presente."
    ],
    "boxes": ["O campo boxes é obrigatório a menos que type esteja presente em items."]
  }
}
```

> Dans les champs imbriqués, la clé utilise le point (`address_sender.country_code`) et, dans les tableaux,
> elle utilise l'index (`items.0.hscode`). Utilisez cela pour signaler l'erreur dans votre formulaire/intégration.

## Tableau général des statuts HTTP

| Statut | Quand cela se produit | Que faire |
|---|---|---|
| `200` | Requête acceptée/traitée | Poursuivre le flux |
| `308` | Appel sans préfixe de version | Utiliser `/v1/...` ou `/v2/...` |
| `400` | Une règle métier a bloqué l'opération (ex. : document fiscal invalide, boîte sans article, code postal invalide) | Lire `message` et corriger le scénario |
| `401` | Type de paiement non activé pour le compte | Activer l'option de paiement ou utiliser un autre type |
| `403` | Sans permission, compte bloqué, cotation simple ou expirée lors de l'expédition | Demander l'activation au support / refaire la cotation |
| `404` | Ressource introuvable (token, cotation d'un autre compte, expédition, contrat non listé dans la cotation, adresse) | Vérifier l'identifiant indiqué |
| `406` | Conflit métier (cotation déjà utilisée, adresse enregistrée incompatible, chargement exigeant un traitement personnalisé) | Lire `message` et ajuster le flux |
| `422` | Échec de validation des champs, ou règle métier sur les données (ex. : HS Code sans table de taxe, compte uniquement DDP) | Corriger les champs listés dans `data` / lire `message` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Préréglages du compte (presets)

Certaines options peuvent être **préconfigurées sur votre compte** dans le panneau. Elles sont appliquées par l'API après la
validation du payload, avec des règles différentes selon le champ :

| Préréglage | Affecte | Comment il est appliqué |
|---|---|---|
| Taxe (`tax`) | Cotation, Manifeste, Brouillon | **Remplace** le `tax` envoyé. Les pays de destination configurés comme exception reçoivent l'inverse du défaut. Si le compte est **verrouillé en DDP**, la taxe est toujours `sender` |
| Assurance automatique | Cotation, Manifeste | Souscrit l'assurance lorsque la valeur déclarée (convertie en USD) atteint le minimum configuré, même avec `insurance: false` |
| Canal de vente (`channel_code`) | Cotation, Manifeste, Brouillon | Utilisé lorsque le champ n'est pas envoyé |
| Système de mesure (`measurement`) | Cotation, Manifeste, Brouillon | Utilisé lorsque le champ n'est pas envoyé ; à défaut, `metric` |
| Livraison résidentielle / non empilable | Cotation, Brouillon | Utilisés lorsque les champs ne sont pas envoyés |
| Fret déclaré (`declared_freight`) | Cotation, Manifeste, Brouillon | Utilisé lorsque le champ n'est pas envoyé. Exige la permission de fret déclaré |
| Regroupement de boîtes | Cotation v2 (`type: items`) | Définit comment les boîtes sont constituées (voir ci-dessous). Ce n'est pas un champ du payload |
| Centre de coûts (`cost_center_code`) | Expédition v1/v2, Manifeste, Brouillon | Utilisé lorsque le champ n'est pas envoyé **ou est envoyé vide/nul** |
| Finalité de l'envoi (`type` de l'expédition) | Expédition v1/v2, Brouillon | Utilisée lorsque le champ n'est pas envoyé |
| Facturant (`biller`) | Expédition v1/v2, Manifeste, Brouillon | Utilisé lorsque le champ n'est pas envoyé (ou arrive vide) et que le compte est configuré pour afficher le facturant |
| Notification de facturation (`payment_notify_shipping_billing`) | Expédition v1/v2 | Utilisée lorsque le champ n'est pas envoyé |
| Enlèvement automatique | Expédition | Crée automatiquement la planification d'enlèvement (voir [Planification d'Enlèvement](api-agendamento.md)) |

**Priorité :** à l'exception de la **taxe** (qui suit toujours le préréglage, lorsqu'il est configuré), ce que vous envoyez
dans le payload l'emporte sur le préréglage. Les champs obligatoires du payload (`object`, `type`, `insurance`, devises,
`address_sender`) doivent toujours être envoyés — le préréglage ne les remplace pas.

> L'adresse d'expéditeur préconfigurée dans le panneau ne remplace **pas** l'`address_sender` de l'API.
> Pour réutiliser une adresse enregistrée, envoyez `address_sender.address_code`.

### Regroupement de boîtes
Le regroupement définit comment les articles sont répartis dans les boîtes lorsque vous utilisez `type: items`
(cotation par articles, **exclusive à la v2**).
Ce n'est **pas** un champ du payload : c'est une configuration de votre compte.

| Mode | Comportement |
|---|---|
| `default` | Chaque article utilise la boîte liée à son enregistrement (`box_code`/`box_identifier`) ; les articles regroupables ayant la même boîte sont réunis et les articles non regroupables reçoivent une boîte par unité |
| `all_in_one` | Consolide les articles regroupables dans une seule boîte virtuelle avec le volume total (à partir des dimensions enregistrées) ; les articles non regroupables reçoivent une boîte par unité |
| `smartbox` | Place les articles dans les boîtes **envoyées dans `boxes[]`** (obligatoire), en tenant compte du volume et des dimensions avec 10 % de marge et en respectant `is_groupable`/`is_foldable` de l'article |

Exemples de chaque mode dans [Cotation d'Articles](api-cotacao-v2.md#cotation-darticles).

## Enregistrements effectués dans le panneau

Certains codes utilisés dans les payloads proviennent d'enregistrements maintenus **dans le panneau de Shipsmart Global** (il n'existe pas d'endpoint
public pour les créer). Ils doivent exister, être actifs et appartenir à votre compte (ou aux comptes autorisés) :

| Enregistrement | Champ dans le payload | Où il est utilisé |
|---|---|---|
| Adresse | `address_sender.address_code`, `address_receiver.address_code`, `manufacturer.address_code`, `biller.address_code` | Cotation, Expédition, Manifeste, Brouillon |
| Canal de vente | `channel_code` | Cotation, Manifeste, Brouillon, consultations et annulation par canal |
| Centre de coûts | `cost_center_code` | Expédition (création et mise à jour), Manifeste, Brouillon |
| Kit | `boxes[].kit` | Cotation, Manifeste, Brouillon |

Les boîtes et les articles peuvent également être enregistrés via l'API : voir [Boîte](api-caixa-v1.md) et [Articles](api-itens-v1.md).

## Devises acceptées

Même liste pour `currency_quote` (devise de la cotation) et `currency_payment` (devise du paiement) :

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> La disponibilité de `currency_payment` avec paiement via passerelle dépend du prestataire :
> `BRL` est traité par un prestataire et les autres devises par un autre. Si la devise choisie n'est pas
> activée pour votre compte, l'expédition renvoie `warning` au moment du paiement.

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

1. **Conservez toujours le `quotation`** renvoyé par la cotation : c'est la clé de tout le reste du flux
   (expédition, statut, documents, planification).
2. **Enregistrez également votre identifiant commercial** dans `channel_identifier_number`. Cela permet de consulter
   le statut, les documents et d'annuler en utilisant le numéro de votre commande, sans dépendre du `quotation`.
3. **Ne faites jamais de polling agressif.** Les consultations de statut/documents doivent avoir un intervalle de quelques
   secondes et un backoff exponentiel.
4. **Traitez `warning` comme une erreur métier** (ne répétez pas la même requête) et `error` comme transitoire
   (vous pouvez réessayer).
5. **Respectez les limites de taille des champs** décrites dans chaque page. Les textes plus longs sont rejetés
   avec `422`, et non tronqués.
6. **Les cotations expirent.** Créez l'expédition dans la durée de validité indiquée dans `valid_until` de chaque transporteur.

## Prochaines étapes

- [Cotation](api-cotacao-v1.md) ou [Cotation V2](api-cotacao-v2.md)
- [Expédition](api-remessa-v1.md) ou [Expédition V2](api-remessa-v2.md)
- [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Codes d'Erreur](api-codigos-de-erro.md)
