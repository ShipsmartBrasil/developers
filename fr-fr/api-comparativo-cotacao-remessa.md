# Cotation & Expédition : v1 vs v2

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
Il existe deux versions publiées pour le couple **cotation + expédition**. Elles utilisent le **même calcul**
et les **mêmes transporteurs**. Les différences résident dans le **moment** où vous indiquez l'adresse complète et dans la
**manière** dont vous décrivez le chargement : seule la v2 accepte la cotation par articles, sans indiquer les boîtes.

| | v1 | v2 |
|---|---|---|
| Adresse complète | Indiquée **dans la cotation** | Indiquée **dans l'expédition** |
| Payload de la cotation | Pays + ville + code postal/état | Uniquement le pays (+ état optionnel) |
| Cotation par articles (`type: items`) | Non disponible : boîtes toujours indiquées | **Disponible** : uniquement le `sku` des articles enregistrés |
| Payload de l'expédition | Voie, nom, contact | Voie, nom, contact **+ ville, état, code postal** |
| Validation de l'adresse | Dans la cotation | Dans l'expédition |
| Cycle opérationnel complet | Oui, en `/v1` | Création en `/v2`, le reste en `/v1` |

## En une phrase

> **Utilisez la v1** si, au moment de coter, vous savez déjà vers quelle ville/quel code postal vous allez envoyer.
> **Utilisez la v2** si, au moment de coter, vous ne connaissez que le pays de destination **ou** si vous souhaitez coter uniquement avec les SKU
> enregistrés, sans indiquer les boîtes.

## Règle d'or : ne mélangez pas les versions

Une cotation créée en v1 ne génère une expédition **que** dans `POST /v1/shipping`.
Une cotation créée en v2 ne génère une expédition **que** dans `POST /v2/shipping`.

Les mélanger renvoie `422`, avec le message ci-dessous dans `data.quotation` :
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Comparatif de la cotation

| Thème | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Obligatoire | Obligatoire |
| `address_*.city` | **Obligatoire** | Ignoré |
| `address_*.postal_code` | Obligatoire (ou `state`) ; **toujours obligatoire au Brésil** | Ignoré |
| `address_*.state` | Obligatoire (ou `state_code`/`postal_code`) | Ignoré |
| `address_*.state_code` | Obligatoire (ou `state`/`postal_code`) | **Optionnel** (améliore la précision) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Identique |
| `type` | `simple`, `advanced` | `simple`, `advanced` **et `items`** (exclusif à la v2) |
| `boxes[]` (incl. `kit`, `volume`) | **Toujours obligatoire** | Obligatoire, sauf en `type: items` (selon le regroupement du compte) |
| `items[]` (incl. `ean`, `nsmp`) | Pris en charge | Identique, **sans** `selling_unit_value` |
| `ior` (Importer of Record) | Pris en charge | **Ne fait pas partie du contrat** |
| `declared_freight` | Pris en charge | Identique |
| `manufacturer` | Pris en charge | Identique |
| `level` (`simple`/`detailed`) | Pris en charge | Identique |
| Validation de l'adresse (code postal, ville, état) | Dans la cotation | Aucune (effectuée dans l'expédition) |
| Consulter le résultat ensuite | `GET /v1/quotation/{quotation}/{level}` | **N'existe pas** |
| Warnings douaniers | Pris en charge | Identique |
| Préréglages du compte (taxe, assurance automatique, etc.) | Appliqués | Identique |
| Scénarios (document, colis, palette, articles, kit) | [Scénarios de la v1](api-cotacao-v1.md#scénarios-de-cotation) | [Scénarios de la v2](api-cotacao-v2.md#scénarios-de-cotation) |

### Impact pratique sur la cotation
- **Précision :** en v1, la ville et le code postal permettent d'appliquer les frais et restrictions régionales **dès le calcul**.
  En v2, le calcul tient compte du pays et de l'état ; l'adresse complète n'intervient que dans l'expédition.
- **Erreurs d'adresse :** en v1, elles apparaissent dans la cotation ; en v2, elles apparaissent dans l'expédition.
- **Nouvelle consultation :** la v2 n'a pas d'endpoint de nouvelle consultation. **Conservez la réponse du `POST`** si vous devez
  réafficher la cotation à l'utilisateur.

## Comparatif de l'expédition

| Thème | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | Cotation v1 | Cotation v2 |
| `carrier_code` / `slug` | L'un des deux | Identique |
| `address_*.street`, `name`, `email`, `phone`, `type` | Obligatoires | Identiques |
| `address_*.city` | Ignoré (provient de la cotation) | **Obligatoire** (min. 3, max. 100) |
| `address_*.postal_code` | Ignoré | **Obligatoire** si `state` n'est pas envoyé |
| `address_*.state` | Ignoré | **Obligatoire** si `postal_code` n'est pas envoyé |
| `address_*.state_code` | Ignoré | Optionnel ; celui de la cotation prévaut |
| `address_*.address_code` | Doit correspondre à l'adresse de la cotation | Ne fonctionne que si la cotation v2 a utilisé le même `address_code` |
| `payment.type` | `billed`, `gateway`, `afterward` | Identique |
| `nfkey`, `invoice_number`, `shipment_reference` | Pris en charge | Identiques |
| `cost_center_code`, `payment_notify_shipping_billing` | Pris en charge | Identiques |
| `attachments` | Pris en charge | Identique |
| `biller` | Pris en charge | Identique, avec localité validée |
| Éligibilité de la cotation | [Non éligibles v1](api-remessa-v1.md#cotations-non-éligibles-à-lexpédition) | [Non éligibles v2](api-remessa-v2.md#cotations-non-éligibles-à-lexpédition) |

## Opérations après la création

Toutes les opérations après la création de l'expédition utilisent **les mêmes endpoints `/v1`**, quelle que soit la
version dans laquelle l'expédition a été créée.

| Opération | Route | Valable pour une expédition v1 ? | Valable pour une expédition v2 ? |
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

**v1 — sans localité (elle provient de la cotation) :**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Rua Principal",
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
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

**v2 — avec localité (elle ne provient pas de la cotation) :**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
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
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
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
2. **Devez-vous consulter à nouveau la cotation plus tard, dans une autre requête ?**
   - Oui → **v1** (`GET /v1/quotation/{quotation}/{level}`)
   - Non → l'une ou l'autre
3. **Utilisez-vous `ior` (Importer of Record) ou `selling_unit_value` ?**
   - Oui → **v1**
   - Non → l'une ou l'autre
4. **Souhaitez-vous le plus haut niveau de précision des taxes dès la cotation ?**
   - Oui → **v1**
   - Non → l'une ou l'autre
5. **Souhaitez-vous coter uniquement avec les SKU enregistrés, sans indiquer les boîtes ?**
   - Oui → **v2** ([Cotation d'Articles](api-cotacao-v2.md#cotation-darticles))
   - Non → l'une ou l'autre

### Scénarios typiques

| Scénario | Version recommandée | Raison |
|---|---|---|
| Checkout d'e-commerce (adresse déjà renseignée) | **v1** | Cotation précise, affichée au client final |
| Calculateur de fret sur la page produit (pays uniquement) | **v2** | Le client n'a pas encore indiqué son adresse |
| Marketplace qui reçoit la commande complète via webhook | **v1** | Toutes les données existent déjà |
| Intégration B2B avec expédition par lot | **v1** | Flux opérationnel complet dans la même version |
| Simulation sur une landing page internationale | **v2** | Payload minimal, moins de friction |
| ERP avec un catalogue de produits déjà enregistré | **v2** | Cotation par articles : uniquement SKU et quantité, la plateforme constitue les boîtes |
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
  [Contrat](api-contrato-v1.md)) sont partagés et n'existent qu'en `/v1`.
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
