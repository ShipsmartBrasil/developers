# Brouillon

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Permet d'enregistrer un envoi **incomplet** : vous transmettez ce que vous avez déjà, et la plateforme valide ensuite.
- À l'entrée, **tous les champs sont optionnels** (seul le format de chaque champ est vérifié). La validation
  stricte a lieu ensuite, lors du **dispatch**, qui s'exécute automatiquement par cycles de quelques minutes.
- Lorsque le brouillon passe le dispatch, la plateforme génère elle-même la **cotation + l'expédition** et le brouillon
  **cesse d'exister** (il est supprimé). Tant que des données manquent, il reste `incomplete` et peut être corrigé.
- C'est le flux indiqué lorsque l'intégration reçoit les données de la commande par étapes (ex. : la commande arrive avant
  l'adresse finale ou la facture fiscale).
- Les **préréglages du compte** (presets) sont appliqués au brouillon, y compris le responsable des
  taxes, qui **remplace** la valeur envoyée lorsqu'il est configuré.

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

Le header `lang` accepte `pt`, `en` et `es` (par défaut `pt`). Chaque opération exige une permission propre
dans le profil (créer, modifier, lister et supprimer des brouillons). Sans elle, la réponse est `403 Has no permission!`.

## Flux d'utilisation
1. `POST /v1/draft` avec les données que vous avez déjà. La réponse confirme seulement la réception
   (**elle ne renvoie pas le code du brouillon**).
2. Attendez quelques minutes : le brouillon est enregistré, puis passe par le dispatch automatique.
3. `GET /v1/draft` pour localiser le brouillon (par votre `channel_identifier_number`), obtenir le `codigo`
   et voir le `status`. S'il est `incomplete`, lisez le `status_descricao`.
4. `PUT /v1/draft/{code}` en envoyant **le brouillon complet et corrigé** (le `PUT` remplace tout le contenu).
5. Répétez les étapes 2 à 4 jusqu'à ce que le brouillon disparaisse de la liste : cela signifie que la cotation et l'expédition ont été créées.
6. Suivez l'expédition via le [suivi](api-rastreio.md) et téléchargez les [documents](api-documentos.md),
   en utilisant le `channel_identifier_number` (et le `channel_code`, le cas échéant).
7. `DELETE /v1/draft` pour supprimer les brouillons qui ne seront pas utilisés (ou qui sont restés au statut `error`).

## Concepts essentiels

### Brouillon vs. autres flux

| Thème | Cotation + Expédition | Manifeste | Brouillon |
|---|---|---|---|
| Payload | Complet en 2 étapes | Complet en 1 appel | **Partiel, en autant d'étapes que nécessaire** |
| Validation | Synchrone, par étape | Synchrone, totale | **Format à l'entrée ; règles complètes au dispatch (asynchrone)** |
| Réponse | Cotation avec les prix | Documents prêts | Confirmation de réception (sans code) |
| Paiement | `billed`, `gateway`, `afterward` | `billed` | **`billed` ou `afterward`** |
| Choix du contrat | Vous choisissez | Vous choisissez | **Automatique** (`carrier_code` préféré ou préférence du compte) |
| Correction ultérieure | Nouvelle cotation | Non | **Oui, via `PUT` (uniquement en `incomplete`)** |

### Cycle de vie du brouillon

```
POST /v1/draft                      PUT /v1/draft/{code}
     │                                       │
     ▼                                       ▼
  received ──► validating ──► incomplete ──► edited ──► validating ──► ...
                    │
                    ▼
                 complete ──► processing ──► cotation + expédition créées (brouillon supprimé)
                                  │
                                  ├──► incomplete  (refus par une règle métier : motif dans status_descricao)
                                  └──► error       (défaillance inattendue : motif dans status_descricao)
```

| Statut | Signification | Modifiable (`PUT`) ? |
|---|---|---|
| `received` | Brouillon reçu, en attente du prochain cycle de dispatch | Non |
| `edited` | Brouillon mis à jour via `PUT`, en attente du prochain cycle de dispatch | Non |
| `validating` | En cours de validation | Non |
| `incomplete` | **Des données manquent ou une règle a refusé l'envoi.** Le motif se trouve dans `status_descricao` | **Oui** |
| `complete` | A passé la validation, prêt à générer la cotation et l'expédition | Non |
| `processing` | Génération de la cotation et de l'expédition en cours | Non |
| `error` | Défaillance inattendue lors du traitement | Non (supprimez-le et créez-le à nouveau) |

| Détail | Comportement |
|---|---|
| Fréquence du dispatch | Les brouillons `received` et `edited` sont validés automatiquement toutes les ~3 minutes |
| Succès | La cotation et l'expédition sont créées et **le brouillon est supprimé** (il n'apparaît plus dans le `GET`) |
| Brouillon bloqué en `processing` | Après ~15 minutes sans aboutissement, il revient à `incomplete` avec un `status_descricao` vide |
| Taille du `status_descricao` | Limitée à **254 caractères** : la liste des points en suspens peut être tronquée |
| Format du `status_descricao` | Lors de la validation des champs, il contient le premier message de chaque champ, concaténés sans séparateur |

> **Seuls les brouillons au statut `incomplete` peuvent être modifiés.** Dans tout autre statut, la réponse est
> `400 Apenas rascunhos com o status incomplete podem ser editados`.

### Le `PUT` remplace l'intégralité du brouillon

Le `PUT /v1/draft/{code}` **ne** fusionne **pas** avec le contenu précédent : le contenu enregistré est **remplacé** par le
payload envoyé (déjà fusionné avec les presets du compte). Les champs que vous ne renvoyez pas **sont perdus**.

- Envoyez toujours le brouillon complet, avec les corrections.
- Attention lorsque vous construisez le `PUT` à partir du `GET` : la liste **ne renvoie pas** `measurement`,
  `residential_delivery`, `non_stackable`, `cost_center_code`, `declared_freight`, `biller`,
  `manufacturer` ni `address_sender.address_code`. Renvoyez-les à partir de vos propres données.

### Préréglages du compte (presets) dans le brouillon

Les presets configurés dans le tableau de bord sont fusionnés avec le payload lors du `POST` **et** du `PUT`
(voir [Préréglages du compte](api-primeiros-passos.md#préréglages-du-compte-presets)).

| Champ | Preset utilisé lorsque | Remarque |
|---|---|---|
| `channel_code` | Le champ **n'**est **pas** envoyé | Envoyer le champ (même `null`) empêche le preset |
| `cost_center_code` | Le champ n'est pas envoyé **ou** arrive vide/`null` | Seul champ pour lequel une valeur vide reçoit aussi le preset |
| `carrier_code` | Le champ n'est pas envoyé | - |
| `tax` | **Chaque fois qu'il existe un preset de taxes** | Le preset **remplace** la valeur envoyée (voir ci-dessous) |
| `insurance` | Le champ n'est pas envoyé | - |
| `measurement` | Le champ n'est pas envoyé ou arrive `null` | Sans preset, `metric` est adopté |
| `currency_quote`, `currency_payment` | Le champ n'est pas envoyé | - |
| `residential_delivery`, `non_stackable` | Le champ n'est pas envoyé | - |
| `type` (finalité de l'envoi) | Le champ n'est pas envoyé | - |
| `address_sender` | L'objet `address_sender` **n'**est **pas** envoyé | Le preset est appliqué comme `address_code`. Envoyer `address_sender` avec n'importe quel champ écarte le preset |
| `biller` | L'objet n'est pas envoyé **ou** arrive vide, et le compte est configuré pour afficher le facturant | Si vous envoyez un `biller` renseigné, c'est toujours lui qui s'applique |
| `declared_freight` | L'objet n'est pas envoyé | Fret déclaré transmis à la cotation |

> La fusion se fait **au premier niveau** : un objet envoyé (`address_sender`, `biller`,
> `declared_freight`) remplace entièrement l'objet du preset, sans combinaison champ par champ.

### Responsable des taxes (`tax`)

Lorsque le compte possède un preset de taxes, **la valeur envoyée dans `tax` est ignorée** et la plateforme détermine
le responsable ainsi :

| Configuration du preset de taxes | Valeur enregistrée dans le brouillon |
|---|---|
| Sans preset de taxes | Le `tax` envoyé. S'il n'est pas envoyé, le dispatch signale `tax` comme obligatoire |
| Par défaut `sender` | `sender`, même si vous envoyez `receiver` |
| Par défaut `receiver` | `receiver`, même si vous envoyez `sender` |
| Valeur par défaut définie et pays de destination dans la liste des exceptions | L'**opposé** de la valeur par défaut (`sender` ↔ `receiver`) |
| Mode verrouillé | Toujours `sender` |

- Le pays de destination pris en compte est l'`address_receiver.country_code` **du même payload**. S'il n'est
  pas envoyé, l'exception par pays n'est pas évaluée et la valeur par défaut s'applique.
- Comme le `PUT` réapplique les presets, la taxe est recalculée à chaque mise à jour.

## Endpoint : créer un brouillon

- **Méthode et route :** `POST /v1/draft`
- **Quand l'utiliser :** pour enregistrer un envoi, même avec des données manquantes.
- **Traitement :** asynchrone. La réponse `200` confirme seulement la réception ; le brouillon apparaît dans le
  `GET` quelques instants plus tard.

### Paramètres — racine du payload

« Entrée » = règle vérifiée lors de l'appel (erreur `422`). « Dispatch » = règle vérifiée ensuite, qui laisse
le brouillon `incomplete`.

| Champ | Type | Entrée | Dispatch | Min. | Max. | Par défaut | Valeurs / description |
|---|---|---|---|---|---|---|---|
| `channel_identifier_number` | string | optionnel | optionnel | - | 100 | - | Votre identifiant de commande. **Recommandé** : c'est grâce à lui que vous localisez le brouillon et l'expédition |
| `channel_code` | integer | optionnel | optionnel | - | - | préréglage du compte | Canal de vente enregistré. Au dispatch, il doit appartenir à votre compte |
| `cost_center_code` | integer | optionnel | optionnel | - | - | préréglage du compte | Centre de coûts enregistré de votre compte (ou de comptes autorisés) |
| `carrier_code` | integer | optionnel | optionnel | - | - | préréglage du compte | Contrat préféré. Utilisé s'il est disponible dans le résultat de la cotation |
| `object` | string | optionnel | optionnel | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | string | optionnel | optionnel | - | 30 | `Commercial` (ou preset) | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `tax` | string | optionnel | **obligatoire** | - | - | préréglage du compte | `sender`, `receiver`. Voir [Responsable des taxes](#responsable-des-taxes-tax) |
| `insurance` | boolean | optionnel | **obligatoire** | - | - | préréglage du compte | Souscrire une assurance |
| `measurement` | string | optionnel | optionnel | - | - | preset ou `metric` | `metric`, `imperial` |
| `ior` | boolean | optionnel | optionnel | - | - | `false` | Importer of Record |
| `residential_delivery` | boolean | optionnel | optionnel | - | - | `false` (ou preset) | Livraison résidentielle |
| `non_stackable` | boolean | optionnel | optionnel | - | - | `false` (ou preset) | Colis non empilables |
| `currency_quote` | string | optionnel | **nécessaire** (voir note) | 3 | 3 | préréglage du compte | Devise de la cotation. Voir [devises](api-primeiros-passos.md#devises-acceptées) |
| `currency_payment` | string | optionnel | **nécessaire** (voir note) | 3 | 3 | préréglage du compte | Devise du paiement. Voir [devises](api-primeiros-passos.md#devises-acceptées) |
| `nfkey` | string | optionnel | optionnel | 44 | 44 | - | Clé de la NF-e |
| `description` | string | optionnel | optionnel | - | 155 | - | Description libre |
| `address_sender` | object | optionnel | **obligatoire** | - | - | préréglage du compte | Expéditeur (voir tableau) |
| `address_receiver` | object | optionnel | **obligatoire** | - | - | - | Destinataire (voir tableau) |
| `boxes` | array | optionnel | voir règles | - | - | - | Colis (voir tableau) |
| `items` | array | optionnel | voir règles | - | - | - | Articles (voir tableau) |
| `payment` | object | optionnel | **obligatoire** | - | - | - | Paiement (voir tableau) |
| `declared_freight` | object | optionnel | optionnel | - | - | préréglage du compte | Fret déclaré : `value` (number) et `currency` (3 lettres). Il n'est pas vérifié à l'entrée ; il est transmis à la cotation |
| `biller` | object | optionnel | optionnel | - | - | préréglage du compte | Facturant affiché sur l'invoice (même structure que l'expéditeur) |
| `manufacturer` | object | optionnel | optionnel | - | - | - | Fabricant affiché sur l'invoice (voir tableau) |

> **Note sur les devises :** la validation du dispatch ne signale pas l'absence de `currency_quote` et
> `currency_payment`, mais sans elles (dans le payload ou dans le preset) la génération de la cotation échoue et le brouillon
> passe en `error`, qui **ne peut pas être modifié**. Indiquez toujours les deux devises ou configurez le preset.

### Paramètres — `address_sender` et `address_receiver`

Deux façons d'indiquer l'**expéditeur** (ne les mélangez pas) :
- **Forme A — adresse enregistrée :** envoyez uniquement `address_sender.address_code`. Au dispatch, l'adresse
  enregistrée est copiée dans le brouillon et validée selon les mêmes règles ci-dessous.
- **Forme B — champs manuels :** envoyez les champs ci-dessous et **n'**envoyez **pas** `address_code`.

Le destinataire n'accepte **que la Forme B** (`address_receiver.address_code` n'est pas utilisé).

| Champ | Type | Min. | Max. (entrée) | Max. (dispatch) | Obligatoire au dispatch | Description |
|---|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | - | Code d'adresse enregistrée (uniquement `address_sender`) |
| `country_code` | string | 2 | 2 | 2 | **Oui** | Pays en ISO 3166-1 alpha-2 |
| `postal_code` | string | - | 12 | 12 | Oui, si `state` n'est pas indiqué | Code postal/ZIP |
| `state` | string | - | - | - | Oui, si `postal_code` et `state_code` ne sont pas indiqués | Nom de l'état |
| `state_code` | string | - | - | - | Voir note | Sigle de l'état, vérifié par rapport à la base des états. Pour un pays sans états, utilisez le code du pays lui-même |
| `city` | string | - | 100 | 100 | **Oui** | Ville |
| `street` | string | - | 70 | 70 | **Oui** | Rue |
| `number` | string | - | 100 | **6** | - | Numéro du bâtiment |
| `district` | string | - | 30 | 30 | - | Quartier |
| `complement` | string | - | 20 | 20 | - | Complément |
| `description` | string | - | 155 | 155 | - | Libellé interne de l'adresse |
| `name` | string | 5 | 35 | 35 | **Oui** | Nom du contact/raison sociale |
| `email` | string | - | 60 | 60 | **Oui** | E-mail valide |
| `phone` | string | - | 25 | 25 | **Oui** | Téléphone |
| `phone_extension` | string | - | 5 | 5 | - | Poste |
| `type` | string | - | - | - | **Oui** (sans valeur par défaut) | `pf` ou `pj` |
| `federal_tax_id` | string | - | 20 | 20 | Oui lorsque le pays est `BR` | Document fédéral. Pour `BR` sans `foreign: true`, il doit s'agir d'un CPF ou d'un CNPJ valide |
| `state_tax_id` | string | - | 20 | 20 | Oui lorsque le pays est `BR` | Inscription d'État |
| `foreign` | boolean | - | - | - | - | Indique une personne étrangère (dispense de la vérification du CPF/CNPJ) |

> **Localisation de l'adresse :** indiquez `postal_code` **ou** `state`. `state_code` seul ne suffit pas,
> car, sans `state`, le `postal_code` devient exigé.
>
> **Attention au `number` :** l'entrée accepte jusqu'à 100 caractères, mais le dispatch n'en accepte que **6**.
> Des valeurs plus longues laissent le brouillon `incomplete`.

### Paramètres — `boxes[]`

Lorsque vous **envoyez `boxes`**, chaque boîte doit être indiquée sous **l'une** des formes ci-dessous (ne les mélangez pas
dans la même boîte) :

| Forme | Champs | Ce qui se passe |
|---|---|---|
| A — kit | `kit` | La boîte et les articles du kit sont remplis à partir du kit enregistré |
| B — boîte enregistrée par code | `box_code` | Utilise la [boîte enregistrée](api-caixa-v1.md) |
| C — boîte enregistrée par identifiant | `box_identifier` | Utilise la boîte enregistrée |
| D — boîte manuelle | `name`, `height`, `width`, `depth` | **Les quatre champs sont obligatoires au dispatch** |

| Champ | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU du kit enregistré |
| `box_code` | integer | - | - | Code de la boîte enregistrée |
| `box_identifier` | string | - | - | Identifiant de la boîte enregistrée |
| `name` | string | 4 | 155 | Nom de la boîte |
| `height` | number | 1 | - | Hauteur |
| `width` | number | 1 | - | Largeur |
| `depth` | number | 1 | - | Profondeur |
| `volume` | number | 0.001 | - | Volume. Accepté, mais **ne dispense pas** des dimensions au dispatch |
| `weight` | number | 0.1 | - | Poids du colis, jusqu'à 3 décimales. **Obligatoire (dès l'entrée) lorsque `object` vaut `doc`**. Ignoré pour une boîte avec `kit` |
| `price` | number | 0 | - | Valeur déclarée. **Obligatoire (dès l'entrée) lorsque `object` vaut `doc`**. Ignoré pour une boîte avec `kit` |

> Avec `boxes` et un `object` différent de `doc`, **toute boîte qui n'est pas un kit doit avoir au moins un article**
> qui pointe vers elle dans `items[].box_id`. Avec `object: doc`, les `items` sont écartés.

### Paramètres — `items[]`

Deux façons d'indiquer chaque article :
- **Forme A — article enregistré :** `sku` (+ `quantity`, `unit_value` et, avec `boxes`, `box_id`). Les données
  manquantes proviennent de l'[enregistrement des articles](api-itens-v1.md). Le SKU doit exister.
- **Forme B — article manuel :** tous les champs marqués comme obligatoires sans `sku`.

| Champ | Type | Min. | Max. | Obligatoire au dispatch | Description |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU de l'article enregistré |
| `ean` | string | - | 50 | - | Code EAN |
| `nsmp` | string | - | 50 | - | *Non Standard Manufacturer Product Id* : identifiant de produit non standardisé du fabricant. Imprimé sur l'invoice et utilisé dans les informations réglementaires exigées par certains transporteurs pour les destinations de l'Union européenne. Avec `sku`, s'il n'est pas envoyé, il provient de l'enregistrement de l'article |
| `box_id` | integer | - | - | Oui, lorsqu'il y a `boxes` | **Position** de la boîte dans le tableau `boxes` (1 = la première). Sans `boxes`, il est ignoré |
| `name` | string | 6 | 35 | Oui, sans `sku` | Nom de l'article |
| `description` | string | 4 | 155 | - | Description de l'article (le minimum de 4 est vérifié au dispatch) |
| `country_code` | string | 2 | 2 | Oui, sans `sku` | Pays d'origine de l'article |
| `hscode` | string | 6 | 10 | Oui, sans `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Oui, sans `sku` | Poids unitaire (le maximum de 99 est vérifié au dispatch) |
| `quantity` | integer | 1 | 1000 | **Oui** | Quantité |
| `unit_value` | number | - | - | **Oui** | Valeur unitaire |
| `selling_unit_value` | number | - | - | Exigé par la cotation lorsque `ior` vaut `true` | Valeur de vente unitaire |
| `height`, `width`, `depth` | number | 1 | - | Oui, lorsqu'il **n'**y a **pas** de `boxes` (sauf SKU enregistré avec dimensions) | Dimensions de l'article, utilisées pour calculer les boîtes |
| `is_groupable` | boolean | - | - | - | Par défaut `true`. Sans `boxes` : les articles regroupables vont ensemble dans une boîte calculée ; `false` génère une boîte par unité |
| `is_foldable` | boolean | - | - | - | Accepté, mais sans effet dans le brouillon |

> **Quand `items` est obligatoire :** sans `boxes`, `items` est toujours exigé (y compris pour `doc`). Avec
> `boxes`, il est exigé pour `package`, `pallet` et `not_doc`, sauf lorsque les boîtes utilisent `kit`.
>
> **Sans `boxes` :** la plateforme constitue les boîtes à partir des dimensions des articles. Pour `doc`, envoyez
> toujours `boxes` avec `weight` et `price`.

### Paramètres — `payment`

| Champ | Type | Obligatoire au dispatch | Valeurs | Description |
|---|---|---|---|---|
| `payment.type` | string | **Oui** | `billed`, `afterward` | Le brouillon **n'accepte pas** `gateway` (erreur `422` à l'entrée) |

### Paramètres — `biller`

Même structure et mêmes limites que `address_sender`, y compris `address_code` (Forme A) ou les champs manuels
(Forme B). Au dispatch, tous les champs du `biller` sont optionnels ; le `number` accepte jusqu'à 6 caractères.
Si vous n'envoyez pas de `biller` (ou l'envoyez vide), le facturant du preset du compte peut être appliqué.

### Paramètres — `manufacturer`

Envoyez `address_code` **ou** les champs manuels. Le fabricant **n'est pas vérifié à l'entrée** : les limites
ci-dessous sont appliquées au dispatch.

| Champ | Type | Min. | Max. | Obligatoire au dispatch | Description |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Code d'adresse enregistrée. Lorsqu'il est utilisé, les autres champs sont optionnels |
| `name` | string | 5 | 30 | Oui, sans `address_code` | Raison sociale/nom du fabricant |
| `country_code` | string | 2 | 2 | Oui, sans `address_code` | Pays |
| `state_code` | string | 1 | 10 | Oui, sans `address_code` | Sigle de l'état |
| `state` | string | - | 155 | - | Nom de l'état |
| `postal_code` | string | - | 12 | Oui, sans `address_code` | Code postal/ZIP |
| `city` | string | - | 100 | Oui, sans `address_code` | Ville |
| `street` | string | - | 70 | Oui, sans `address_code` | Rue |
| `number` | string | - | 6 | - | Numéro |
| `district` | string | - | 30 | - | Quartier |
| `complement` | string | - | 20 | - | Complément |
| `email` | string | - | 60 | Oui, sans `address_code` | E-mail |
| `phone` | string | - | 25 | Oui, sans `address_code` | Téléphone |
| `phone_extension` | string | - | 6 | - | Poste |
| `type` | string | - | - | Oui, sans `address_code` | `pf` ou `pj` |
| `federal_tax_id` | string | - | 20 | - | Document fédéral |
| `state_tax_id` | string | - | 20 | - | Inscription d'État |

### Payload minimal

Le plus petit payload accepté est un objet vide : il crée un brouillon qui restera `incomplete` jusqu'à ce qu'il soit complété.

```json
{}
```

En pratique, envoyez au moins votre identifiant, afin de pouvoir localiser le brouillon ensuite :

```json
{
  "channel_identifier_number": "PED-2026-0001"
}
```

Payload minimal pour que le brouillon **passe le dispatch** sans dépendre des presets (expéditeur et destinataire
hors du Brésil, boîte manuelle) :

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "US",
    "postal_code": "10001",
    "city": "New York",
    "street": "Fifth Avenue",
    "name": "Boutique Exemple",
    "email": "contato@example.com",
    "phone": "12125550100",
    "type": "pj"
  },
  "address_receiver": {
    "country_code": "CA",
    "postal_code": "M5V2T6",
    "city": "Toronto",
    "street": "King Street",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "14165550100",
    "type": "pf"
  },
  "boxes": [
    { "name": "Boite S", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "T-shirt coton",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ],
  "payment": { "type": "billed" }
}
```

### Payload maximal

Tous les champs acceptés. Les colis montrent les formes manuelle (boîte 1), boîte enregistrée (boîte 2)
et kit (boîte 3) ; dans une intégration réelle, utilisez la forme adaptée à chaque colis. L'expéditeur utilise la Forme B
(champs manuels) ; pour la Forme A, envoyez uniquement `"address_sender": {"address_code": 1}`.

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 1,
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Brouillon de la commande du marketplace",
  "declared_freight": {
    "value": 50,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centre de distribution SP",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Entrepot 2",
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Bureau Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": "987654321",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Boite master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "volume": 0.008,
      "weight": 2.5,
      "price": 150
    },
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "NSMP-TSH-000-S",
      "name": "T-shirt coton S",
      "description": "T-shirt 100% coton manches courtes blanc",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
    },
    {
      "box_id": 2,
      "name": "Bermuda serge M",
      "description": "Bermuda en serge beige",
      "country_code": "BR",
      "hscode": "620342",
      "weight": 0.4,
      "quantity": 1,
      "unit_value": 59.9,
      "selling_unit_value": 89.9
    }
  ],
  "payment": { "type": "billed" },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Siege",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centro",
    "complement": "Salle 1",
    "name": "Holding Exemple LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Usine Exemple LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "number": "200",
    "district": "Centro",
    "complement": "Bloc B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000144",
    "state_tax_id": "111222333444"
  }
}
```

### Exemple de requête

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Boite S", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt coton", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Exemple de réponse

`200 OK`

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

`422 Unprocessable Entity` (ex. : `payment.type` avec `gateway`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "payment.type": ["O campo payment.type não contém um valor válido."]
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` ou `error` |
| `message` | string | Message de confirmation ou d'erreur |
| `data` | object | Uniquement en cas d'erreur `422` : dictionnaire champ → liste de messages |

> La réponse de succès **ne contient pas le code du brouillon**. Utilisez `GET /v1/draft` pour l'obtenir.

## Endpoint : lister les brouillons

- **Méthode et route :** `GET /v1/draft`
- **Quand l'utiliser :** pour découvrir le `codigo` d'un brouillon et suivre le `status`/`status_descricao`.
- **Portée :** uniquement les brouillons de votre compte. **Éléments par page :** 20.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `page` | query | integer | non | Page de la liste (par défaut 1) |
| `drafts` | body | object | non | Enveloppe du filtre |
| `drafts.drafts_ids` | body | array | non | Codes de brouillon (entiers ou chaînes). **Vide ou absent, il retourne tous les brouillons** |

> Ce `GET` accepte un corps JSON pour le filtre. Si votre client HTTP ne prend pas en charge un body dans un `GET`,
> listez tout (en paginant) et filtrez de votre côté.

### Payload minimal

Sans corps et sans query : liste la première page de tous les brouillons.

```
GET /v1/draft
```

### Payload maximal

```
GET /v1/draft?page=1
```

```json
{
  "drafts": {
    "drafts_ids": [30, 31]
  }
}
```

### Exemple de requête

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
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
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "carrier_code": "",
          "nfkey": "",
          "object": "package",
          "type": "Commercial",
          "channel_code": "",
          "channel": "",
          "tax": "sender",
          "insurance": true,
          "ior": false,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "description": "",
          "address_sender": {
            "country_code": "",
            "state": "",
            "state_code": "",
            "postal_code": "",
            "city": "",
            "description": "",
            "street": "",
            "number": "",
            "district": "",
            "complement": "",
            "name": "",
            "email": "",
            "phone": "",
            "type": "",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "address_receiver": {
            "country_code": "US",
            "state": "",
            "state_code": "",
            "postal_code": "33101",
            "city": "Miami",
            "description": "",
            "street": "Main Avenue",
            "number": "",
            "district": "",
            "complement": "",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "items": [
            {
              "box_id": 1,
              "name": "T-shirt coton",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9
            }
          ],
          "boxes": [
            { "name": "Boite S", "height": 20, "width": 20, "depth": 20 }
          ],
          "payment": { "type": "billed" }
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "links": [
      { "url": null, "label": "« Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/draft?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo »", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.data[].codigo` | integer | **Code du brouillon.** À utiliser dans `PUT /v1/draft/{code}` et dans `drafts.drafts_ids` |
| `data.data[].codigo_usuario` | integer | Code du compte propriétaire du brouillon |
| `data.data[].data` | object | Contenu du brouillon (déjà fusionné avec les presets), au format fixe : les champs absents arrivent sous la forme `""` |
| `data.data[].data.channel` | string | Nom du canal de vente (lorsqu'il y a un `channel_code`) |
| `data.data[].status` | string | Statut actuel (voir [cycle de vie](#cycle-de-vie-du-brouillon)) |
| `data.data[].status_descricao` | string/null | **Lorsqu'il est `incomplete` ou `error`, indique ce qui a empêché la progression** (jusqu'à 254 caractères) |
| `data.data[].created_at`, `updated_at` | string | Dates au format `AAAA-MM-JJ HH:MM:SS` |
| `data.current_page`, `last_page`, `per_page`, `total`, `from`, `to` | integer | Données de pagination |
| `data.first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url`, `path`, `links` | string/array | Liens de navigation entre les pages |

> L'objet `data` de la liste **n'inclut pas** `measurement`, `residential_delivery`, `non_stackable`,
> `cost_center_code`, `declared_freight`, `biller`, `manufacturer` ni `address_sender.address_code`.
> Lorsque l'expéditeur a été indiqué par `address_code`, après le premier dispatch il apparaît développé
> avec les données de l'adresse enregistrée.
>
> S'il n'existe aucun brouillon pour les filtres (ou si la page demandée est au-delà de la dernière), la réponse est
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`, et non une liste vide.

## Endpoint : mettre à jour un brouillon

- **Méthode et route :** `PUT /v1/draft/{code}`
- **Quand l'utiliser :** pour corriger un brouillon `incomplete`.
- **Restriction :** uniquement les brouillons de votre compte au statut `incomplete`.
- **Corps :** mêmes champs et mêmes règles que le `POST`. **Le contenu précédent est entièrement remplacé**
  (voir [Le PUT remplace l'intégralité du brouillon](#le-put-remplace-lintégralité-du-brouillon)).
- **Effet :** le statut passe à `edited` et le brouillon retourne au prochain cycle de dispatch.
  Le `status_descricao` précédent est conservé jusqu'à la nouvelle validation.

### Paramètres

| Paramètre | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `code` | path | integer | **oui** | Code du brouillon (`codigo` dans la liste) |
| corps | body | object | non | Même structure que le [`POST`](#endpoint--créer-un-brouillon) |

### Payload minimal

`PUT /v1/draft/30` avec le corps `{}`. Il est accepté, mais **efface** le contenu du brouillon (seuls les presets restent).
Ne l'utilisez que si vous voulez recommencer le brouillon.

```json
{}
```

### Payload maximal

Le même [payload maximal du `POST`](#payload-maximal). En pratique, renvoyez le brouillon complet avec les corrections.

### Exemple de requête

Brouillon qui était `incomplete` faute d'expéditeur : le corps contient le contenu original **plus** l'expéditeur.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rue Principale", "number": "100", "name": "Boutique Exemple LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Boite S", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "T-shirt coton", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso.",
  "data": {
    "codigo": 30,
    "codigo_usuario": 1,
    "data": {
      "channel_identifier_number": "PED-2026-0001",
      "object": "package",
      "type": "Commercial",
      "tax": "sender",
      "insurance": true,
      "measurement": "metric",
      "currency_quote": "USD",
      "currency_payment": "BRL",
      "address_sender": {
        "country_code": "BR",
        "postal_code": "09941070",
        "city": "Diadema",
        "street": "Rue Principale",
        "number": "100",
        "name": "Boutique Exemple LTDA",
        "email": "contato@example.com",
        "phone": "11989677465",
        "type": "pj",
        "federal_tax_id": "12345678000199",
        "state_tax_id": "123456789012"
      },
      "address_receiver": {
        "country_code": "US",
        "postal_code": "33101",
        "city": "Miami",
        "street": "Main Avenue",
        "name": "John Doe",
        "email": "johndoe@example.com",
        "phone": "11992494597",
        "type": "pf"
      },
      "boxes": [
        { "name": "Boite S", "height": 20, "width": 20, "depth": 20 }
      ],
      "items": [
        {
          "box_id": 1,
          "name": "T-shirt coton",
          "country_code": "BR",
          "hscode": "610910",
          "weight": 0.25,
          "quantity": 3,
          "unit_value": 29.9
        }
      ],
      "payment": { "type": "billed" }
    },
    "status": "edited",
    "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
    "created_at": "2026-02-20 19:04:00",
    "updated_at": "2026-02-20 19:20:31"
  }
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `data.codigo` | integer | Code du brouillon |
| `data.codigo_usuario` | integer | Code du compte propriétaire du brouillon |
| `data.data` | object | Contenu enregistré : le payload envoyé déjà fusionné avec les presets. Il peut contenir des champs supplémentaires provenant des préréglages du compte ; ignorez ceux que vous ne connaissez pas |
| `data.status` | string | Toujours `edited` après la mise à jour |
| `data.status_descricao` | string/null | Motif de la validation précédente (sera mis à jour au prochain dispatch) |
| `data.created_at`, `data.updated_at` | string | Dates au format `AAAA-MM-JJ HH:MM:SS` |

## Endpoint : supprimer des brouillons

- **Méthode et route :** `DELETE /v1/draft`
- **Quand l'utiliser :** pour supprimer les brouillons qui ne seront pas utilisés, y compris ceux restés en `error`.
- **Portée :** seuls les codes appartenant à votre compte sont supprimés ; les autres sont ignorés.
  La suppression est définitive.

### Paramètres

| Champ | Emplacement | Type | Obligatoire | Description |
|---|---|---|---|---|
| `drafts` | body | object | **oui** | Enveloppe |
| `drafts.drafts_ids` | body | array | **oui** (non vide) | Codes des brouillons à supprimer (entiers ou chaînes) |

### Payload minimal

```json
{ "drafts": { "drafts_ids": [30] } }
```

### Payload maximal

```json
{ "drafts": { "drafts_ids": [30, 31, 32] } }
```

### Exemple de requête

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

### Exemple de réponse

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | `success` lorsqu'au moins un brouillon de la liste a été supprimé |
| `message` | string | Message de confirmation |

> Il suffit qu'**un** code de la liste appartienne à votre compte pour que la réponse soit `success` ; les autres sont
> ignorés sans avertissement. Si aucun code ne correspond, la réponse est
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Règles métier

### Validation au dispatch
Au dispatch, la plateforme applique un ensemble **strict** de règles. Ce qui manque apparaît dans
`status_descricao` et le brouillon reste `incomplete`.

| Groupe | Exigences |
|---|---|
| Général | `tax` et `insurance` présents (dans le payload ou via preset) ; `channel_identifier_number` texte de 100 caractères maximum |
| Expéditeur et destinataire | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`postal_code` ou `state`) ; `number` de 6 caractères maximum |
| Adresses au Brésil (`BR`) | `federal_tax_id` et `state_tax_id` ; `federal_tax_id` doit être un CPF ou un CNPJ valide, sauf `foreign: true` |
| Sans `boxes` | `items` obligatoire (quel que soit l'`object`), avec des dimensions pour calculer les boîtes |
| Avec `boxes` | Chaque boîte : `kit`, `box_code`, `box_identifier` ou (`name` + `height` + `width` + `depth`). Pour `object` ≠ `doc`, chaque boîte (sauf kit) doit avoir au moins un article avec son `box_id` |
| `object` = `doc` | `boxes[].weight` et `boxes[].price` (vérifiés dès l'entrée) |
| Articles | `quantity` (1 à 1000) et `unit_value` ; sans `sku` : `name`, `country_code`, `hscode`, `weight` (jusqu'à 99). Avec `sku`, le SKU doit être enregistré |
| Paiement | `payment.type` (`billed` ou `afterward`) |
| Fabricant | Avec `manufacturer` sans `address_code` : `name`, `country_code`, `state_code`, `postal_code`, `city`, `street`, `email`, `phone` et `type` |
| Canal | `channel_code`, lorsqu'il est indiqué (ou provient du preset), doit appartenir à votre compte |

### Génération de la cotation et de l'expédition
Une fois validé, le brouillon devient automatiquement une cotation et une expédition :

| Étape | Comportement |
|---|---|
| Kits | Chaque `boxes[].kit` est remplacé par la boîte et les articles du kit enregistré |
| Boîtes | Avec `boxes`, les boîtes envoyées sont utilisées. Sans `boxes`, les boîtes sont calculées à partir des dimensions des articles (regroupables ensemble ; non regroupables, une boîte par unité) |
| Cotation | Type simple pour `doc` et avancé pour les autres. Utilise `declared_freight` du payload ou du preset |
| Choix du contrat | Si `carrier_code` figure parmi les résultats, il est utilisé. Sinon, la plateforme choisit selon la préférence du compte : le **moins cher** ou le **plus rapide** |
| Expédition | Un `type` vide devient `Commercial` ; `nfkey`, `cost_center_code` et `biller` sont transmis lorsqu'ils sont indiqués |
| Refus par une règle métier | Le brouillon revient à `incomplete`, avec le message de la cotation/de l'expédition dans `status_descricao` |
| Défaillance inattendue | Le brouillon passe en `error`. Supprimez-le et créez-en un nouveau |

### Messages possibles dans `status_descricao`
En plus des messages de validation des champs (ex. : `O campo address sender.country code é obrigatório.`),
les messages suivants peuvent apparaître :

| Message | Cause | Comment corriger |
|---|---|---|
| `Nenhum item foi enviado para este rascunho` | Sans `items` (et sans kits) | Envoyer `items` |
| `O item com o sku :sku não foi encontrado!` | SKU non enregistré et article sans données complètes | Enregistrer l'article ou envoyer les champs manuels |
| `Nenhuma caixa foi enviado para este rascunho` | `boxes` vide, ou `box_code`/`box_identifier` introuvable | Vérifier les boîtes |
| `Parâmetros da caixa não foram prenchidos corretamente por favor certifique-se dos seguintes campos (name,height,width,depth)` | Boîte manuelle sans `name`, `height`, `width` ou `depth` | Compléter les quatre champs |
| `Nenhum item foi associado para a caixa :name !` | Boîte sans article avec son `box_id` | Ajuster `items[].box_id` |
| `Não há itens válidos, verifique os dados e/ou o codigo SKU` | Sans `boxes`, aucun article exploitable | Vérifier les articles et les SKU |
| `O canal especificado não foi encontrado!` | `channel_code` n'appartient pas à votre compte | Corriger le `channel_code` |
| `O kit :sku é inválido!` | Kit inexistant ou inactif | Vérifier le SKU du kit |
| `A caixa do kit :sku é inválida!` | La boîte du kit est introuvable | Vérifier l'enregistrement du kit |
| `Um ou mais itens do kit :sku são inválidos!` | Article du kit introuvable | Vérifier l'enregistrement du kit |
| Messages de la cotation ou de l'expédition | Ex. : aucun contrat disponible, restrictions de charge | Voir [Cotation](api-cotacao-v1.md) et [Expédition](api-remessa-v1.md) |

## Erreurs courantes

| Statut | Signification / Message | Comment corriger |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | Ne modifiez que des brouillons `incomplete` ; consultez le statut dans le `GET` |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Vérifier le `code`/`drafts_ids` (il peut appartenir à un autre compte ou avoir déjà été converti en expédition et supprimé) |
| `403` | `Has no permission!` | Demander au support la permission de l'action (créer, modifier, lister ou supprimer des brouillons) |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + `data` avec les champs | Corriger les champs listés dans `data` |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` (dans `data.cost_center_code`) | Utiliser un centre de coûts de votre compte ou de comptes autorisés |
| `422` | `O campo drafts é obrigatório.` / `O campo drafts.drafts ids é obrigatório.` (dans le `DELETE`) | Envoyer `drafts.drafts_ids` avec au moins un code |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Réessayer avec backoff ; si cela persiste, contacter le support |

Les erreurs d'authentification (token invalide, compte bloqué) se trouvent dans
[Premiers Pas](api-primeiros-passos.md#erreurs-dauthentification).

## Remarques finales
- Le champ **`status_descricao` est votre guide** : il indique ce qui a empêché le brouillon de progresser.
- Indiquez toujours `channel_identifier_number` : c'est ce qui relie le brouillon à votre commande, permet de trouver le
  `codigo` dans la liste et de consulter l'expédition une fois le brouillon supprimé.
- Dans le `PUT`, **renvoyez le brouillon complet** : le contenu précédent est remplacé.
- Si le compte possède un preset de taxes, le `tax` envoyé est ignoré ; vérifiez la valeur enregistrée dans le `GET`.
- Le brouillon **n'accepte pas** le paiement via `gateway`. Pour ce scénario, utilisez
  [cotation](api-cotacao-v1.md) + [expédition](api-remessa-v1.md).
- Pour le flux inverse (tout validé en une seule fois, documents immédiats), voir [Manifeste](api-manifesto.md).
- Enregistrements utilisés par le brouillon : [Boîte](api-caixa-v1.md), [Articles](api-itens-v1.md), [Contrats](api-contrato-v1.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
