# Cotation & Expédition : v1 vs v2

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
Il existe deux versions publiées pour le couple **cotation + expédition**. Elles utilisent le **même moteur de calcul**
et les **mêmes transporteurs**. La différence réside dans le **moment** où vous indiquez l'adresse complète.

| | v1 | v2 |
|---|---|---|
| Adresse complète | Indiquée **dans la cotation** | Indiquée **dans l'expédition** |
| Payload de la cotation | Pays + ville + code postal/état | Uniquement le pays (+ état optionnel) |
| Payload de l'expédition | Rue, nom, contact | Rue, nom, contact **+ ville, état, code postal** |
| Cycle opérationnel complet | Oui, sur `/v1` | Création sur `/v2`, le reste sur `/v1` |

## En une phrase

> **Utilisez la v1** si, au moment de coter, vous savez déjà vers quelle ville/quel code postal vous allez expédier.
> **Utilisez la v2** si, au moment de coter, vous ne connaissez que le pays de destination.

## Règle d'or : ne mélangez pas les versions

Une cotation créée en v1 ne génère une expédition **que** avec `POST /v1/shipping`.
Une cotation créée en v2 ne génère une expédition **que** avec `POST /v2/shipping`.

Mélanger retourne `422` :
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Comparatif de la cotation

| Thème | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Obligatoire | Obligatoire |
| `address_*.city` | **Obligatoire** | Non accepté |
| `address_*.postal_code` | Obligatoire (ou `state`) | Non accepté |
| `address_*.state` | Obligatoire (ou `state_code`/`postal_code`) | Non accepté |
| `address_*.state_code` | Obligatoire (ou `state`/`postal_code`) | **Optionnel** (améliore la précision) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Identique |
| `type` | `simple`, `advanced`, `items` | Identique |
| `boxes[]` (y compris `kit`, `volume`) | Pris en charge | Identique |
| `items[]` | Pris en charge | Identique, **sans** `selling_unit_value` |
| `ior` (Importer of Record) | Pris en charge | **Ne fait pas partie du contrat** |
| `declared_freight` | Pris en charge | Identique |
| `manufacturer` | Pris en charge | Identique |
| `level` (`simple`/`detailed`) | Pris en charge | Identique |
| Consulter le résultat plus tard | `GET /v1/quotation/{quotation}/{level}` | **N'existe pas** |
| Avertissements douaniers | Pris en charge | Identique |

### Impact pratique sur la cotation
- **Précision :** en v1, la ville et le code postal permettent d'appliquer les taxes et restrictions régionales **dès le calcul**.
  En v2, le montant peut être affiné quand l'adresse complète arrive dans l'expédition.
- **Nouvelle consultation :** la v2 n'a pas d'endpoint de reconsultation. **Conservez la réponse du `POST`** si vous devez
  réafficher la cotation à l'utilisateur.

## Comparatif de l'expédition

| Thème | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | Cotation v1 | Cotation v2 |
| `carrier_code` / `slug` | L'un des deux | Identique |
| `address_*.street`, `name`, `email`, `phone`, `type` | Obligatoires | Identiques |
| `address_*.city` | Non accepté | **Obligatoire** (min. 3, max. 100) |
| `address_*.postal_code` | Non accepté | **Obligatoire** (ou `state`) |
| `address_*.state` / `state_code` | Non acceptés | **Obligatoires** (l'un des deux, ou `postal_code`) |
| `payment.type` | `billed`, `gateway`, `afterward` | Identique |
| `nfkey`, `invoice_number`, `shipment_reference` | Pris en charge | Identiques |
| `cost_center_code`, `payment_notify_shipping_billing` | Pris en charge | Identiques |
| `attachments` | Pris en charge | Identique |
| `biller` | Pris en charge | Identique, avec la localité validée |

## Opérations après la création

Toutes les opérations après la création de l'expédition utilisent **les mêmes endpoints `/v1`**, indépendamment de la
version dans laquelle l'expédition a été créée.

| Opération | Route | Fonctionne pour une expédition v1 ? | Fonctionne pour une expédition v2 ? |
|---|---|---|---|
| Consulter le statut | `GET /v1/shipping/{shipping}/status` | Oui | **Oui** |
| Statut par canal | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` | Oui | **Oui** |
| Consulter les documents | `GET /v1/shipping/{shipping}/docs` | Oui | **Oui** |
| Documents par canal | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` | Oui | **Oui** |
| Mettre à jour (NF-e, pièces jointes, centre de coûts) | `PUT /v1/shipping/{shipping}` | Oui | **Oui** |
| Supprimer des pièces jointes | `DELETE /v1/shipping/{shipping}/docs` | Oui | **Oui** |
| Annuler | `DELETE /v1/shipping/{shipping}` | Oui | **Oui** |
| Annuler par canal | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` | Oui | **Oui** |
| Planifier un enlèvement | `POST /v1/schedule` | Oui | **Oui** |
| Consulter la planification | `GET /v1/schedule/{shipping}` | Oui | **Oui** |
| Annuler la planification | `DELETE /v1/schedule/{shipping}/{reason}` | Oui | **Oui** |

> Il n'existe pas de routes `GET`, `PUT` ou `DELETE` sous `/v2/shipping`.

## Comparaison des payloads côte à côte

### Cotation

**v1 — adresse complète dans la cotation :**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "T-shirt coton", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

**v2 — uniquement pays/état dans la cotation :**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "T-shirt coton", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

### Expédition

**v1 — sans localité (elle vient de la cotation) :**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Rue Principale",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

**v2 — avec localité (elle ne vient pas de la cotation) :**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "state_code": "FL",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

## Comment choisir

Répondez aux questions dans l'ordre :

1. **Au moment de coter, connaissez-vous déjà la ville et le code postal de destination ?**
   - Oui → **v1**
   - Non → **v2**
2. **Avez-vous besoin de reconsulter la cotation plus tard, dans une autre requête ?**
   - Oui → **v1** (`GET /v1/quotation/{quotation}/{level}`)
   - Non → n'importe laquelle
3. **Utilisez-vous `ior` (Importer of Record) ou `selling_unit_value` ?**
   - Oui → **v1**
   - Non → n'importe laquelle
4. **Voulez-vous le plus haut niveau de précision des taxes dès la cotation ?**
   - Oui → **v1**
   - Non → n'importe laquelle

### Scénarios typiques

| Scénario | Version recommandée | Raison |
|---|---|---|
| Checkout e-commerce (adresse déjà renseignée) | **v1** | Cotation précise, affichée au client final |
| Calculateur de fret sur la page produit (pays uniquement) | **v2** | Le client n'a pas encore indiqué l'adresse |
| Marketplace qui reçoit la commande complète via webhook | **v1** | Toutes les données existent déjà |
| Intégration B2B avec expédition en lot | **v1** | Flux opérationnel complet dans la même version |
| Simulation sur une landing page internationale | **v2** | Payload minimal, moins de friction |
| Cotation + expédition + documents en un seul appel | **[Manifeste](api-manifesto.md)** | Flux synchrone, indépendant de v1/v2 |

## Autres façons de créer une expédition

Outre le couple cotation + expédition, il existe deux flux alternatifs :

| Flux | Route | Caractéristique |
|---|---|---|
| [Manifeste](api-manifesto.md) | `POST /v1/manifest` | **Synchrone** : cote, crée l'expédition chez le transporteur et renvoie les documents dans la même réponse. N'accepte que le paiement `billed` |
| [Brouillon](api-rascunho.md) | `POST /v1/draft` | **Asynchrone** : accepte un payload partiel, valide ensuite et évolue vers cotation + expédition. Accepte le paiement `billed` ou `afterward` |

## Remarques finales
- Les deux versions coexistent : vous pouvez utiliser la v2 sur la vitrine et la v1 au checkout, à condition que **chaque cotation
  génère l'expédition dans sa propre version**.
- Les endpoints de support ([Boîte](api-caixa-v1.md), [Articles](api-itens-v1.md),
  [Contrat](api-contrato-v1.md)) sont partagés et n'existent que sur `/v1`.
