# Manifeste

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre opération.

## Vue d'ensemble
- Exécute **cotation + expédition + envoi au transporteur + génération de l'étiquette en un seul appel synchrone**.
- La réponse contient déjà l'identifiant de l'expédition, le numéro de suivi et l'étiquette (AWB) en PDF et/ou ZPL.
- C'est le flux recommandé pour les opérations qui ont besoin de l'étiquette **immédiatement**, sans attendre
  de traitement asynchrone.
- Ne prend en compte que les contrats de transporteurs prenant en charge l'expédition synchrone (actuellement **FedEx, UPS et DHL**).
- N'accepte que le **paiement facturé** (`payment.type: billed`), et ce mode de paiement doit être activé sur le compte.

> ⚠️ **Le manifeste est synchrone et irréversible.** Comme il cote et expédie dans la même requête, il n'y a pas
> d'étape intermédiaire de vérification. Validez soigneusement le payload avant de l'envoyer.

## URL de base
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
Content-Type: application/json
Accept: application/json
lang: pt
```

> Le header `lang` accepte `pt`, `en` et `es` (par défaut `pt`). Le profil du token doit disposer de la permission de
> **créer un manifeste** ; sans elle, la réponse est `403` avec `Has no permission!`.

## Flux d'utilisation
1. Construisez le payload complet (adresses, boîtes, articles, paiement). Consultez la section
   [Adresse manuelle : envoyez toutes les clés](#adresse-manuelle--envoyez-toutes-les-clés).
2. `POST /v1/manifest?format=pdf` (ou `zpl`, ou sans `format` pour recevoir les deux).
3. Imprimez l'étiquette reçue dans `data.docs` (utilisez le `base64` ou téléchargez-la via l'`url`).
4. Conservez le `data.shipping` et utilisez-le pour le [suivi](api-rastreio.md), les [documents](api-documentos.md) et
   la [planification d'enlèvement](api-agendamento.md).
5. Quelques instants plus tard, consultez [Documents de l'Expédition](api-documentos.md) pour obtenir les documents
   complémentaires (invoice, packing list, etc.), générés en arrière-plan.

## Concepts essentiels

### Manifeste vs. cotation + expédition

| Thème | Cotation + Expédition | Manifeste |
|---|---|---|
| Appels | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Choix du transporteur | Vous choisissez dans la liste retournée | La plateforme choisit (ou vous l'indiquez avec `carrier_code`/`slug`) |
| Transporteurs | Tous les contrats du compte | **Uniquement FedEx, UPS et DHL** |
| Étiquette (AWB) | Consultation ultérieure dans `/docs` | **Incluse dans la réponse** |
| Paiement | `billed`, `gateway`, `afterward` | **Uniquement `billed`** |
| Type de réponse | Asynchrone | **Synchrone** |
| Vérification avant l'expédition | Oui | Non |
| Champs obligatoires | Moins nombreux (répartis sur 2 étapes) | **Plus nombreux** (tout en une seule fois) |

Utilisez le manifeste lorsque : l'opération est à fort volume, le transporteur est déjà défini par une règle
métier et vous devez imprimer l'étiquette immédiatement.

### Comment le transporteur est choisi

| Ce que vous envoyez | Comportement |
|---|---|
| Ni `slug` ni `carrier_code` | La plateforme choisit selon la préférence configurée sur le compte : **le plus rapide** (délai le plus court, par défaut) ou **le moins cher** (valeur la plus basse dans la devise de paiement) |
| `carrier_code` | C'est une **préférence**. Si le contrat apparaît dans le résultat de la cotation, il est utilisé. S'il n'apparaît pas : contrat inexistant ou d'un transporteur compatible avec le manifeste → la plateforme revient à la préférence du compte ; contrat existant d'un transporteur **non** compatible avec le manifeste → `404` |
| `slug` | **Fixe** le contrat : la cotation n'évalue que celui-ci et le `slug` **remplace** tout `carrier_code` envoyé. Si le contrat ne figure pas parmi les contrats éligibles de votre compte pour cet envoi, la cotation évalue à nouveau tous les contrats et le choix suit la règle du `carrier_code` |

> Ne mélangez pas `slug` et `carrier_code` : utilisez l'un ou l'autre. Si vous envoyez les deux, le `slug` prévaut.
> Lorsque le compte possède un contrat préféré prédéfini et que vous n'envoyez pas `carrier_code`, le préréglage est utilisé comme `carrier_code`.

### Préréglages du compte dans le manifeste

Le manifeste applique les [préréglages du compte](api-primeiros-passos.md#préréglages-du-compte-presets)
**après** la validation des entrées. C'est pourquoi tous les champs obligatoires doivent figurer dans le payload,
même si le compte dispose d'un préréglage pour eux.

| Préréglage | Quand il est appliqué dans le manifeste |
|---|---|
| Responsable de la taxe (`tax`) | **Chaque fois qu'il est configuré : il remplace le `tax` envoyé** (voir le tableau ci-dessous) |
| Centre de coûts (`cost_center_code`) | Lorsque le payload n'envoie pas le champ ou l'envoie vide (`null`, `""` ou `0`) |
| Canal de vente (`channel_code`) | Lorsque la clé `channel_code` n'est pas envoyée. Envoyer `"channel_code": null` désactive le préréglage |
| Contrat préféré (`carrier_code`) | Lorsque la clé `carrier_code` n'est pas envoyée |
| Fret déclaré (`declared_freight`) | Lorsque la clé `declared_freight` n'est pas envoyée. Envoyer `declared_freight` (même `null` ou `{}`) écarte le préréglage |
| Facturant (`biller`) | Lorsque l'affichage du facturant est activé sur le compte et que le payload n'envoie pas `biller`, ou l'envoie vide (`null`, `{}` ou uniquement avec des valeurs vides). Un `biller` renseigné dans le payload prévaut toujours |
| Autres préréglages (`insurance`, `measurement`, `type`, devises, livraison résidentielle, non empilable, adresse de l'expéditeur) | **N'ont aucun effet** : ces champs sont obligatoires dans le manifeste et la valeur du payload prévaut toujours |

**Détermination du responsable de la taxe (`tax`) :**

| Configuration du compte | `tax` effectivement utilisé |
|---|---|
| Sans préréglage de taxe | La valeur envoyée dans le payload |
| Valeur par défaut `sender` ou `receiver` | La valeur par défaut du compte, en **ignorant** la valeur envoyée |
| Valeur par défaut + pays de destination dans la liste d'exceptions du compte | L'**inverse** de la valeur par défaut (`sender` ↔ `receiver`) |
| Mode verrouillé | Toujours `sender`, indépendamment du payload et des exceptions |

> Le pays de destination utilisé dans les exceptions est l'`address_receiver.country_code` du payload. Si le destinataire
> est indiqué uniquement par `address_code`, les exceptions par pays ne sont pas évaluées et la valeur par défaut du compte s'applique.
> Même lorsque le préréglage remplace la valeur, `tax` reste obligatoire dans le payload et doit être `sender` ou `receiver`.

### Étapes du traitement et ce qui se passe en cas d'échec

| Étape | En cas d'échec |
|---|---|
| 1. Validation des entrées | `422`. Rien n'est créé |
| 2. Règles du compte et complément (paiement, canal, `slug`, adresses, boîtes, articles et kits enregistrés, État) | `400`/`404`. Rien n'est créé |
| 3. Cotation, enregistrement de l'expédition et envoi au transporteur | Tout est annulé : aucune cotation ni expédition ne reste enregistrée sur le compte |
| 4. Génération de l'étiquette et des documents (après acceptation de l'envoi par le transporteur) | L'expédition **reste créée**, avec un numéro de suivi, et reçoit un événement d'erreur de traitement dans le suivi. Récupérez les documents dans [Documents de l'Expédition](api-documentos.md) et ne répétez pas le manifeste |

## Endpoint : créer un manifeste

- **Méthode et route :** `POST /v1/manifest`
- **Quand l'utiliser :** pour créer, en un seul appel, une expédition déjà remise au transporteur et
  recevoir l'étiquette dans la réponse.

### Paramètres — query

| Nom | Emplacement | Type | Obligatoire | Par défaut | Valeurs | Description |
|---|---|---|---|---|---|---|
| `format` | query | string | non | vide (tous) | `pdf`, `zpl` | Format de l'étiquette dans la réponse. Vide (ou valeur non reconnue), retourne le PDF **et** le ZPL |

> `format` peut également être envoyé **dans le corps** du payload. Lorsqu'il est envoyé aux deux endroits,
> **la valeur du corps prévaut**. Dans le corps, les valeurs différentes de `pdf`/`zpl` sont rejetées avec `422`.

### Paramètres — racine du payload

| Champ | Type | Obligatoire | Min. | Max. | Valeurs / Par défaut | Description |
|---|---|---|---|---|---|---|
| `object` | string | **oui** | - | - | `doc`, `package`, `pallet`, `not_doc` | Ce qui est envoyé. `not_doc` est accepté par compatibilité et traité comme `package` |
| `type` | string | **oui** | - | - | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalité de l'envoi (valeur exacte, avec majuscules et espaces) |
| `tax` | string | **oui** | - | - | `sender`, `receiver` | Qui paie les taxes. Peut être remplacé par le préréglage du compte (voir [Préréglages](#préréglages-du-compte-dans-le-manifeste)) |
| `insurance` | boolean | **oui** | - | - | - | Souscrire une assurance. Peut être activée automatiquement si le compte dispose de la souscription automatique d'assurance et que la valeur déclarée atteint le minimum configuré |
| `residential_delivery` | boolean | **oui** | - | - | - | Livraison à une adresse résidentielle |
| `non_stackable` | boolean | **oui** | - | - | - | Colis non empilables |
| `measurement` | string | **oui** | - | - | `metric`, `imperial` | Système de mesure des poids et des dimensions |
| `ior` | boolean | **oui** | - | - | - | Importer of Record |
| `currency_quote` | string | **oui** | 3 | 3 | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise de la cotation (et des valeurs des articles) |
| `currency_payment` | string | **oui** | 3 | 3 | voir [devises](api-primeiros-passos.md#devises-acceptées) | Devise du paiement |
| `address_sender` | object | **oui** | - | - | - | Expéditeur (voir le tableau) |
| `address_receiver` | object | **oui** | - | - | - | Destinataire (voir le tableau) |
| `boxes` | array | **oui** | 1 élément | - | - | Colis (voir le tableau) |
| `payment` | object | **oui** | - | - | - | Paiement (voir le tableau) |
| `items` | array | conditionnel | - | - | - | Obligatoire lorsque `object` **n'est pas** `doc` et qu'aucune boîte n'utilise `kit`. **Interdit (la clé ne doit pas exister) lorsque `object` est `doc`** |
| `carrier_code` | integer | non | - | - | - | Contrat préféré (voir [Comment le transporteur est choisi](#comment-le-transporteur-est-choisi)) |
| `slug` | string | non | - | 20 | - | Identifiant textuel qui fixe le contrat. Doit exister ; un contrat supprimé ou hors de sa période de validité retourne `400` |
| `channel_code` | integer | non | - | - | - | Code du canal de vente enregistré **sur votre compte** |
| `channel_identifier_number` | string | non | - | - | - | Votre identifiant commercial (ex. : numéro de commande). **Recommandé** |
| `cost_center_code` | integer | non | - | - | Préréglage du compte | Centre de coûts enregistré pour votre compte ou pour des comptes associés autorisés. Vide/absent → utilise le préréglage du compte |
| `nfkey` | string | non | 44 | 44 | - | Clé de la NF-e (exactement 44 caractères) |
| `description` | string | non | 5 | 155 | - | Description libre du manifeste |
| `declared_freight` | object | non | - | - | Préréglage du compte | Fret déclaré (voir le tableau) |
| `biller` | object | non | - | - | Préréglage du compte | Facturant affiché sur l'invoice (voir la section) |
| `manufacturer` | object | non | - | - | - | Fabricant affiché sur l'invoice (voir la section) |
| `extra` | object | non | - | - | - | Données auxiliaires sous forme de paires clé/valeur (voir le tableau) |
| `format` | string | non | - | - | `pdf`, `zpl` | Alternative à la query string (prévaut sur celle-ci) |
| `size` | string | non | - | - | `label`, `letter` | Accepté lors de la validation, mais **ne modifie actuellement pas** l'étiquette générée |

### Paramètres — `address_sender` et `address_receiver`

Contrairement à l'expédition v1/v2, l'adresse est ici indiquée **complète**, car la cotation et l'expédition
ont lieu dans le même appel. Il existe deux façons d'indiquer chaque adresse — **ne mélangez pas les deux dans le même objet**.

**Forme A — adresse déjà enregistrée :**

| Champ | Type | Obligatoire | Description |
|---|---|---|---|
| `address_code` | integer | **oui** dans cette forme | Code d'une adresse enregistrée et visible pour votre compte |

> Avec `address_code`, **toutes** les données de l'adresse proviennent de l'enregistrement ; tout autre champ envoyé dans le même
> objet est écarté. Si vous envoyez `address_code` avec `country_code`, `street`, `name`, `email`,
> `phone`, `type` et `foreign`, la requête est rejetée avec `422`. L'expéditeur et le destinataire ne peuvent pas utiliser le
> même `address_code`.

**Forme B — adresse manuelle :**

| Champ | Type | Obligatoire | Min. | Max. | Par défaut | Description |
|---|---|---|---|---|---|---|
| `country_code` | string | **oui** | 2 | 2 | - | Pays en ISO 3166-1 alpha-2 |
| `postal_code` | string | conditionnel | - | 12 | - | CEP/ZIP. Obligatoire si `state` et `state_code` ne sont pas indiqués. Au Brésil, il est utilisé pour déterminer l'État lorsque `state_code` n'est pas envoyé |
| `state` | string | conditionnel | - | 100 | - | Nom de l'État. Obligatoire si `state_code` et `postal_code` ne sont pas indiqués |
| `state_code` | string | conditionnel | 1 | 10 | - | Sigle de l'État. Obligatoire si `state` et `postal_code` ne sont pas indiqués. **Toujours recommandé** (voir les règles métier) |
| `city` | string | conditionnel | 1 | 100 | - | Ville. Obligatoire si `postal_code` n'est pas indiqué. **Toujours recommandée** : les transporteurs utilisent la ville sur l'étiquette |
| `street` | string | **oui** | 5 | 70 | - | Voie |
| `number` | string | non | 1 | 6 | - | Numéro du bâtiment |
| `district` | string | non | 1 | 30 | - | Quartier |
| `complement` | string | non | 1 | 20 | - | Complément |
| `name` | string | **oui** | 1 | 70 | - | Nom du contact/raison sociale |
| `email` | string | **oui** | - | 60 | - | E-mail valide |
| `phone` | string | non | 1 | 25 | - | Téléphone. **Toujours recommandé** : les transporteurs exigent un téléphone de contact |
| `phone_extension` | string | non | - | 5 | - | Poste |
| `type` | string | **oui** | - | - | - | `pf` (personne physique) ou `pj` (personne morale) |
| `foreign` | boolean | **oui** | - | - | - | Indique si la personne est étrangère (utilisé pour valider le `federal_tax_id`) |
| `federal_tax_id` | string | conditionnel | - | 20 | - | Document fédéral (CPF/CNPJ ou équivalent). **Obligatoire lorsque `country_code` est `BR` ou `AR`** |
| `state_tax_id` | string | non | - | 20 | - | Inscription d'État |
| `description` | string | non | 1 | 155 | - | Libellé interne de l'adresse |
| `company_name` | string | non | 5 | 50 | - | **Uniquement dans `address_sender`.** Raison sociale utilisée dans les envois FedEx (le transporteur ne prend en compte que les 35 premiers caractères) |

#### Adresse manuelle : envoyez toutes les clés

Dans le manifeste, les clés ci-dessous doivent **exister** dans l'objet d'adresse manuelle, même lorsque vous n'avez pas
de valeur pour elles — dans ce cas, envoyez `null`. L'absence de l'une d'entre elles interrompt le traitement avec une erreur
(`500`, ou `400` à l'étape du transporteur).

| Clés qui doivent être présentes (la valeur peut être `null`) |
|---|
| `postal_code`, `city`, `number`, `district`, `complement`, `phone`, `phone_extension`, `federal_tax_id`, `state_tax_id` |

> N'envoyez pas non plus `"address_code": null` dans la forme B : la clé `address_code` doit être omise.

### Paramètres — `boxes[]`

Chaque boîte peut être indiquée de trois façons — **ne mélangez pas les formes dans la même boîte** :

| Forme | Champs | Comportement |
|---|---|---|
| A — boîte enregistrée | `box_code` **ou** `box_identifier` | Le nom, les dimensions et le poids proviennent de l'[enregistrement des boîtes](api-caixa-v1.md) |
| B — boîte manuelle | `name`, `height`, `width`, `depth` (+ `weight`/`price` en `doc`) | Données indiquées dans le payload |
| C — kit | `kit` | La boîte **et** les articles de cette position proviennent du kit enregistré |

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `kit` | string | non | - | 50 | SKU du kit enregistré. Remplit la boîte et les articles de cette position ; le `weight` et le `price` de la boîte sont ignorés |
| `box_code` | integer | conditionnel | - | - | Code de la boîte enregistrée. Obligatoire lorsque la boîte n'indique aucun autre champ |
| `box_identifier` | string | conditionnel | - | 50 | Identifiant de la boîte enregistrée. Alternative au `box_code` |
| `name` | string | **oui** sans `kit`/`box_code`/`box_identifier` | 5 | 155 | Nom de la boîte |
| `height` | number | **oui** sans `kit`/`box_code`/`box_identifier` | 1 | - | Hauteur (cm en `metric`, in en `imperial`) |
| `width` | number | **oui** sans `kit`/`box_code`/`box_identifier` | 1 | - | Largeur |
| `depth` | number | **oui** sans `kit`/`box_code`/`box_identifier` | 1 | - | Profondeur |
| `weight` | number | **oui** lorsque `object` est `doc` (y compris avec une boîte enregistrée) | 0.001 | - | Poids du colis (kg en `metric`, lb en `imperial`) |
| `price` | number | **oui** lorsque `object` est `doc` (y compris avec une boîte enregistrée) | 0 | - | Valeur déclarée du colis. En `doc`, la valeur déclarée de l'expédition est la somme des `price` |

> Avec une boîte enregistrée, les champs envoyés dans le payload complètent (et remplacent) ceux de l'enregistrement.
> La boîte doit appartenir à votre compte.

### Paramètres — `items[]`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `box_id` | integer | **oui** | - | - | **Position** de la boîte dans l'array `boxes` (1 = première) |
| `quantity` | integer | **oui** | 1 | 99 | Quantité de l'article |
| `sku` | string | conditionnel | - | 50 | SKU de l'article enregistré ([Articles](api-itens-v1.md)). Obligatoire lorsqu'aucun des champs `name`, `country_code`, `hscode`, `weight` et `unit_value` n'est indiqué |
| `name` | string | **oui** sans `sku` | 6 | 50 | Nom de l'article |
| `country_code` | string | **oui** sans `sku` | 2 | 2 | Pays d'origine de l'article (ISO 3166-1 alpha-2) |
| `hscode` | string | **oui** sans `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **oui** sans `sku` | 0.001 | 99 | Poids unitaire (kg en `metric`, lb en `imperial`) |
| `unit_value` | number | **oui** sans `sku` | supérieur à 0 | - | Valeur unitaire, dans la devise `currency_quote`. La validation accepte `0`, mais la cotation refuse un article de valeur nulle |
| `selling_unit_value` | number | non | 0 | - | Valeur de vente unitaire, utilisée lorsque `ior` vaut `true`. Si elle est omise, elle provient de l'enregistrement de l'article (avec `sku`) ou, à défaut, prend la valeur de `unit_value` |
| `description` | string | non | 5 | 450 | Description détaillée de l'article |
| `ean` | string | non | - | 50 | Code EAN (code-barres) de l'article. Utilisé comme identifiant standardisé du fabricant dans les informations réglementaires pour les destinations de l'Union européenne (et comme identifiant du produit en l'absence de `sku`) |
| `nsmp` | string | non | - | 50 | *Non Standard Manufacturer Product Id* : identifiant de produit non standardisé du fabricant. Il est imprimé sur l'invoice et utilisé dans les informations réglementaires exigées par certains transporteurs pour les destinations de l'Union européenne |

> **Articles avec `sku` :** lorsque l'article indique un `sku` et qu'il manque l'un des champs `name`, `country_code`, `hscode`,
> `weight`, `unit_value`, `nsmp` (ou `selling_unit_value` avec `ior: true`), les données manquantes sont complétées
> à partir de l'[enregistrement de l'article](api-itens-v1.md) (y compris `ean` et `nsmp`). Les valeurs envoyées dans le payload
> prévalent toujours sur l'enregistrement. Si le SKU n'est pas enregistré, l'article n'est accepté que s'il contient
> `name`, `country_code`, `hscode` et `weight` dans le payload lui-même.
>
> Les informations réglementaires pour les destinations de l'Union européenne ne sont constituées que lorsque l'article a un `sku` ou un `ean`.

> **Attention aux différences par rapport à la cotation :** ici, `quantity` a un maximum de **99** et
> `description` accepte jusqu'à **450** caractères.

### Paramètres — `payment`

| Champ | Type | Obligatoire | Valeurs | Description |
|---|---|---|---|---|
| `payment` | object | **oui** | - | Enveloppe |
| `payment.type` | string | **oui** | **`billed`** | Le manifeste accepte **uniquement** le paiement facturé. Le paiement facturé doit également être activé sur le compte ; sinon, la réponse est `400` |

### Paramètres — `declared_freight`

| Champ | Type | Obligatoire | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | non | 0 | - | Valeur du fret déclaré. `0` est accepté |
| `currency` | string | **oui** lorsque `value` est indiqué | 3 | 3 | Devise du fret déclaré |
| `declared` | boolean | non | - | - | Accepté, mais **sans effet** dans le manifeste : le fret déclaré n'est pris en compte que lorsque `value` **et** `currency` sont indiqués |

> Pour déclarer un fret nul, envoyez `"value": 0` avec la `currency`. Sans `value` ni `currency`, le fret déclaré
> n'est pas appliqué (ni celui du préréglage, si vous avez envoyé la clé `declared_freight`).

### Paramètres — `extra`

Paires clé/valeur auxiliaires, enregistrées dans l'expédition.

| Règle | Détail |
|---|---|
| Structure | Objet **plat** (un seul niveau, sans objets ni listes imbriqués) |
| Clés | Uniquement des lettres minuscules, des chiffres et `_` |
| Valeurs | Texte, entier ou décimal (les booléens et `null` sont rejetés) |

| Clé connue | Type | Règle | Description |
|---|---|---|---|
| `federal_tax_provision` | number | - | Provision de taxe fédérale |
| `state_tax_provision` | number | - | Provision de taxe d'État |
| `usd_rate_provision` | number | supérieur à 0 | Provision de change du dollar. **N'est conservée** que pour les comptes du programme Remessa Conforme avec la destination `BR` ; dans les autres cas, elle est écartée silencieusement |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Paramètres — `biller` et `manufacturer`

- `biller` : même format que celui décrit dans [Expédition (v1)](api-remessa-v1.md#paramètres--biller-facturant)
  (`address_code` **ou** champs manuels). Dans le manifeste, `biller.name` contient de 5 à **355** caractères.
  Les champs non envoyés prennent `type: pj` et `foreign: false`. Sans `biller` dans le payload, le
  préréglage de facturant du compte s'applique (lorsqu'il est activé).
- `manufacturer` : même format que celui décrit dans [Cotation (v1)](api-cotacao-v1.md#paramètres--manufacturer)
  (`address_code` **ou** champs manuels). Avec l'`address_code` d'une adresse existante, les données proviennent de l'enregistrement.

### Payload minimal

Manifeste de document (`object: doc`) avec adresses manuelles — `items` est interdit dans ce cas :

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rue Principale",
    "number": null,
    "district": null,
    "complement": null,
    "name": "Boutique Exemple LTDA",
    "email": "contato@example.com",
    "phone": "11999990000",
    "phone_extension": null,
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": null,
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "number": null,
    "district": null,
    "complement": null,
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550000",
    "phone_extension": null,
    "type": "pf",
    "federal_tax_id": null,
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Enveloppe documents",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

**Variante minimale avec adresses enregistrées et kit** (`object: package` ; `items` peut être omis car la boîte utilise `kit`) :

```json
{
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": { "address_code": 1234 },
  "address_receiver": { "address_code": 5678 },
  "boxes": [
    { "kit": "KIT-001" }
  ],
  "payment": { "type": "billed" }
}
```

### Payload maximal

Tous les champs acceptés. `slug` et `carrier_code` apparaissent ensemble uniquement à titre d'illustration : en pratique, utilisez l'un
ou l'autre (le `slug` prévaut). Les boîtes montrent la forme manuelle et la forme enregistrée.

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 12,
  "carrier_code": 45,
  "slug": "fedex-br",
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
  "description": "Manifeste du lot de mars",
  "declared_freight": {
    "declared": true,
    "value": 0,
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
    "district": "Centre",
    "complement": "Hangar 2",
    "company_name": "Boutique Exemple Commerce LTDA",
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
    "phone": "13055550000",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Boite master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 1.5,
      "price": 90
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7891234567895",
      "nsmp": "MFR-TSH-000-S",
      "name": "T-shirt coton P",
      "description": "T-shirt 100% coton, manches courtes, couleur blanche, taille P",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Basket sport",
      "description": "Chaussure de course homme pointure 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Siege social",
    "street": "Rue Principale",
    "number": "100",
    "district": "Centre",
    "complement": "Bureau 1",
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
    "district": "Centre",
    "complement": "Bloc B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000181",
    "state_tax_id": "111222333444"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Exemple de requête

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": false,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rue Principale", "number": null, "district": null, "complement": null, "name": "Boutique Exemple LTDA", "email": "contato@example.com", "phone": "11999990000", "phone_extension": null, "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": null, "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "number": null, "district": null, "complement": null, "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550000", "phone_extension": null, "type": "pf", "federal_tax_id": null, "state_tax_id": null, "foreign": true},
    "boxes": [{"name": "Enveloppe documents", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Exemple de réponse

`200 OK` (sans `format`, la réponse contient `AWB` et `AWB_ZPL` ; avec `format=pdf`, uniquement `AWB` ; avec `format=zpl`, uniquement `AWB_ZPL`) :

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Test",
    "tracking_id": "271249639538",
    "tracking_packages_ids": [
      "271249639538"
    ],
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Champs de la réponse

| Champ | Type | Présence | Description |
|---|---|---|---|
| `data.shipping` | string (UUID) | toujours | **Identifiant de l'expédition.** À utiliser pour le suivi, les documents et la planification |
| `data.carrier` | string | toujours | Nom du contrat/transporteur qui a traité l'envoi |
| `data.tracking_id` | string | toujours | Numéro de suivi principal (AWB) |
| `data.tracking_packages_ids` | array de string | lorsque le transporteur l'indique | Numéro de suivi de chaque colis, dans l'ordre des boîtes |
| `data.docs` | object | toujours | Étiquettes générées, au format `nom → { url, base64 }` |
| `data.docs.AWB` | object | `format` vide ou `pdf` | Étiquette en PDF |
| `data.docs.AWB_ZPL` | object | `format` vide ou `zpl` | Étiquette en ZPL (imprimantes thermiques) |
| `data.docs.AWBPRC` / `data.docs.AWBPRCZPL` | object | comptes du programme Remessa Conforme, pour les importations (origine hors du Brésil et destination `BR`) | Étiquette spécifique au programme, en PDF / ZPL |
| `data.docs.*.url` | string | toujours | Lien du fichier. Si le compte protège les liens des documents, le lien est **temporaire** (valable environ 5 minutes) |
| `data.docs.*.base64` | string | toujours | Contenu du fichier en Base64 |
| `data.valorProvCBS`, `data.valorProvIBSEstadual`, `data.valorProvIBSMunicipal` | number | lorsque le calcul des taxes contient ces provisions | Provisions de CBS et d'IBS (d'État et municipale) |
| `data.serviceType` | string | envois FedEx | Service FedEx utilisé |
| `data.operationalDetail` | object | envois FedEx | Données opérationnelles du transporteur (`ursaPrefixCode`, `ursaSuffixCode`, `originLocationId`, `originLocationNumber`, `originServiceArea`, `destinationLocationId`, `destinationLocationNumber`, `destinationServiceArea`, `destinationLocationStateOrProvinceCode`, `deliveryDate`, `deliveryDay`, `commitDate`, `commitDay`, `ineligibleForMoneyBackGuarantee`, `astraPlannedServiceLevel`, `astraDescription`, `postalCode`, `stateOrProvinceCode`, `countryCode`, `airportId`, `serviceCode`, `packagingCode`, `publishedDeliveryTime`, `scac`). Les champs sans valeur sont `null` |

> Les documents complémentaires — invoice (envois qui ne sont pas `doc`), déclaration de contenu (lorsque
> l'expéditeur est `pf`), packing list et déclaration de produits non dangereux — sont générés en arrière-plan.
> Consultez [Documents de l'Expédition](api-documentos.md) quelques instants plus tard pour obtenir la liste complète.

### Réponse d'erreur du transporteur

Lorsque le transporteur refuse l'envoi, la réponse est `400` avec `status: "error"` et contient son retour brut :

```json
{
  "status": "error",
  "message": "[1234] Fedex - Test - Erreur renvoyee par le transporteur",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Champ | Description |
|---|---|
| `message` | `[code du contrat] nom du transporteur - message` retourné par le transporteur. Si le transporteur répond sans message reconnaissable, la valeur est `CARRIER SYNC - SHIPPING RETURN UNEXPECTED RESPONSE!` |
| `data` | Payload brut du transporteur. **La structure varie selon le transporteur** et peut être `null` |

> Enregistrez `message` et `data` dans vos logs : c'est la seule source du motif technique du refus. Un refus du
> transporteur annule la cotation et l'expédition ; corrigez le payload et envoyez un nouveau manifeste.

## Règles métier
- **Paiement :** uniquement `billed`, et le paiement facturé doit être activé sur le compte (seul ou avec
  le paiement immédiat). Une autre valeur dans `payment.type` → `422` ; compte sans facturation → `400`.
- **Transporteurs :** le manifeste ne cote que les contrats FedEx, UPS et DHL de votre compte. Le choix suit
  [Comment le transporteur est choisi](#comment-le-transporteur-est-choisi).
- **Préréglages :** appliqués après la validation, conformément à
  [Préréglages du compte dans le manifeste](#préréglages-du-compte-dans-le-manifeste). Points clés : le `tax` du
  préréglage **remplace** celui envoyé ; le `cost_center_code` du préréglage remplit le champ lorsqu'il est vide.
- **`items` et `object: doc` :** avec `object: doc`, la clé `items` **ne doit pas** être envoyée (pas même vide).
  Avec tout autre `object`, `items` est obligatoire — sauf lorsqu'une boîte utilise `kit`.
- **Kits :** lorsqu'une boîte indique `kit`, la boîte et les articles de cette position proviennent de l'enregistrement,
  et `boxes[].weight`/`boxes[].price` sont ignorés.
- **Valeur déclarée :** en `doc`, c'est la somme de `boxes[].price` ; dans les autres cas, la somme de `unit_value × quantity`
  des articles. Un article avec une valeur unitaire vide ou nulle est refusé dans la cotation (`400`).
- **`imperial` :** les poids et dimensions indiqués en livres/pouces sont convertis en kg/cm avant la cotation.
- **État (`state_code`) :** doit être un sigle d'État reconnu par la plateforme ; pour les pays sans
  division en États, envoyez le `country_code` lui-même. Sans `state_code`, la plateforme tente de le déterminer
  (par le CEP au Brésil, par le nom dans `state` ou par le code postal à l'étranger). Si elle n'y parvient pas pour
  l'expéditeur **ou** pour le destinataire, le manifeste est refusé avec `400` avant tout enregistrement.
- **`federal_tax_id` :** obligatoire lorsque le pays de l'adresse est `BR` ou `AR`.
- **`address_code` :** l'adresse doit être visible pour votre compte ; avec elle, les autres champs de
  l'objet sont ignorés. L'expéditeur et le destinataire ne peuvent pas avoir le même `address_code`.
- **Adresse manuelle :** envoyez toutes les clés listées dans
  [Adresse manuelle : envoyez toutes les clés](#adresse-manuelle--envoyez-toutes-les-clés) (utilisez `null` en l'absence de valeur).
- **`channel_code` :** doit exister et appartenir à votre compte (y compris lorsqu'il provient du préréglage).
- **`cost_center_code` :** doit exister et appartenir à votre compte, au compte principal ou à des comptes
  associés autorisés.
- **`slug` :** doit exister (`422`) et le contrat doit être actif et dans sa période de validité (`400`).
- **Assurance automatique :** si le compte souscrit automatiquement une assurance à partir d'une valeur minimale et que la valeur
  déclarée (convertie en USD) atteint ce minimum, l'assurance est souscrite même avec `insurance: false`.
- **`extra.usd_rate_provision` :** conservé uniquement pour les comptes du programme Remessa Conforme avec la destination `BR`.
- **`not_doc`** est accepté par compatibilité et traité comme `package`.

## Erreurs courantes

Les erreurs de validation (`422`) ont `status: "warning"`, un `message` égal à
`Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!`
et `data` avec la table `champ → messages`. Les erreurs de règle métier ont `status: "warning"` et ne contiennent pas de `data`.

| Statut | Signification / Message | Comment corriger |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Vérifier le token envoyé dans `Authorization` |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contacter le support |
| `403` | `Has no permission!` | Demander au support la permission de créer un manifeste |
| `422` | Validation des champs (messages dans `data`, ex. : `O campo :attribute é obrigatório.`) | Corriger les champs listés dans `data` |
| `422` | `The :attribute field must be missing when :other is :value.` (dans `items`) | Supprimer la clé `items` lorsque `object` est `doc` |
| `422` | `The :attribute field must be missing when :values are present.` (dans `address_code`) | Utiliser uniquement `address_code` **ou** uniquement les champs manuels |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Utiliser un `cost_center_code` de votre compte |
| `400` | `Tipo de pagamento não autorizado!` | Demander au support l'activation du paiement facturé |
| `400` | `O canal especificado não foi encontrado!` | Utiliser un `channel_code` enregistré sur votre compte (vérifiez également le préréglage) |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | Utiliser le `slug` d'un contrat actif |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Utiliser des `address_code` différents |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | Vérifier l'`address_code` |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Corriger le `postal_code` ou envoyer `state_code` |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Corriger `state` ou envoyer `state_code`/`postal_code` |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Envoyer `state_code` dans les deux adresses |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | Vérifier `box_code`/`box_identifier` |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | Enregistrer l'article ou envoyer les champs manuels |
| `400` | `O kit :sku é inválido!` / `A caixa do kit :sku é inválida!` / `Um ou mais itens do kit :sku são inválidos!` | Revoir l'enregistrement du kit |
| `400` | Message de la cotation (ex. : `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. ...`, `Nenhum transportador encontrado para este usuário!`) | Dans le manifeste, toute erreur de l'étape de cotation est renvoyée en `400`. Lire `message` et ajuster le payload |
| `400` | Message de l'enregistrement de l'expédition | Lire `message` et ajuster le payload |
| `404` | `A transportadora informada não está disponível para esta remessa!` | Utiliser le `carrier_code` d'un contrat FedEx, UPS ou DHL, ou omettre le champ |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Le contrat choisi ne prend pas en charge l'expédition synchrone ; contacter le support |
| `400` | `[código] transportador - mensagem` (`status: "error"`, `data` avec le retour du transporteur) | Voir [Réponse d'erreur du transporteur](#réponse-derreur-du-transporteur) |
| `500` | Erreur inattendue (`status: "error"`) | **Avant de réessayer**, vérifiez dans [Suivi](api-rastreio.md) si l'expédition a été créée |

> ⚠️ **Ne répétez jamais un manifeste automatiquement après un `500` ou après un échec réseau/timeout.** Si l'échec
> survient après que le transporteur a accepté l'envoi, l'expédition existe réellement et une répétition génère un
> envoi en double. Vérifiez d'abord par `channel_identifier_number` + `channel_code` dans [Suivi](api-rastreio.md).

## Remarques finales
- Indiquez toujours `channel_identifier_number` (avec `channel_code`) : c'est votre protection contre les doublons et le
  moyen de retrouver l'expédition si la réponse se perd.
- Le manifeste concentre beaucoup de responsabilités dans un seul appel. Pour les intégrations qui peuvent attendre,
  le flux [cotation](api-cotacao-v1.md) + [expédition](api-remessa-v1.md) est plus sûr.
- Pour le flux inverse (payload partiel, validé ultérieurement), voir [Brouillon](api-rascunho.md).
- Enregistrements utilisés par le manifeste : [Boîte](api-caixa-v1.md) et [Articles](api-itens-v1.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
