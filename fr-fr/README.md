# Guide d'Intégration Technique

> Guide d'intégration avec l'**API Publique de Shipsmart Global** : cotation internationale, création
> d'expéditions, documents, suivi et planification d'enlèvement.

- [Retour au README principal](../README.md)

> ⚠️ Tous les exemples de ce guide (payloads, réponses, codes, documents et tokens) sont **purement illustratifs**
> et **ne sont pas des données valides**.

## Commencez par ici

| Étape | Page | Ce que vous apprenez |
|---|---|---|
| 1 | [Premiers Pas](api-primeiros-passos.md) | Environnements, authentification, langue, format de réponse, erreurs et préréglages |
| 2 | [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md) | Quelle version utiliser dans votre intégration |
| 3 | [Cotation](api-cotacao-v1.md) | Calculer le fret, l'assurance et les taxes |
| 4 | [Expédition](api-remessa-v1.md) | Transformer la cotation en un envoi réel |
| 5 | [Suivi de l'Expédition](api-rastreio.md) | Suivre l'envoi jusqu'à la livraison |
| 6 | [Documents de l'Expédition](api-documentos.md) | Télécharger l'AWB, l'invoice et les autres documents |
| 7 | [Planification d'Enlèvement](api-agendamento.md) | Demander le retrait du colis à votre adresse |
| 8 | [Codes d'Erreur](api-codigos-de-erro.md) | Traiter chaque message d'erreur de l'API |

## Le flux principal

```
Cotation  ──►  Expédition  ──►  Suivi  ──►  Documents  ──►  Enlèvement
```

1. **Coter :** [Cotation](api-cotacao-v1.md) ou [Cotation V2](api-cotacao-v2.md)
2. **Expédier :** [Expédition](api-remessa-v1.md) ou [Expédition V2](api-remessa-v2.md)
3. **Suivre :** [Suivi de l'Expédition](api-rastreio.md)
4. **Documenter :** [Documents de l'Expédition](api-documentos.md)
5. **Faire enlever :** [Planification d'Enlèvement](api-agendamento.md)

## Façons de créer une expédition

| Flux | Route principale | Caractéristique |
|---|---|---|
| [Cotation](api-cotacao-v1.md) + [Expédition](api-remessa-v1.md) | `POST /v1/quotation` → `POST /v1/shipping` | Flux standard. Vous choisissez le transporteur parmi les options cotées |
| [Cotation V2](api-cotacao-v2.md) + [Expédition V2](api-remessa-v2.md) | `POST /v2/quotation` → `POST /v2/shipping` | Pour les cas où seul le pays est connu au moment de coter |
| [Manifeste](api-manifesto.md) | `POST /v1/manifest` | **Synchrone :** cote, expédie et renvoie les documents en un seul appel |
| [Brouillon](api-rascunho.md) | `POST /v1/draft` | **Asynchrone :** accepte un payload partiel et évolue vers cotation + expédition |

## APIs de support

| Page | À quoi ça sert |
|---|---|
| [Boîte](api-caixa-v1.md) | Enregistrer des emballages et les référencer dans la cotation |
| [Articles](api-itens-v1.md) | Enregistrer des produits (poids, dimensions, HS Code) et les utiliser par SKU |
| [Contrat](api-contrato-v1.md) | Lister les contrats de transport activés sur votre compte |
| [Plateforme](api-plataforma.md) | Tester le token et la connectivité ; health check |

## Sommaire complet

### Introduction
- [Premiers Pas](api-primeiros-passos.md)
- [Cotation & Expédition : v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Codes d'Erreur](api-codigos-de-erro.md)

### Cotation
- [Cotation (v1)](api-cotacao-v1.md)
  - [Cotation de Document](api-cotacao-v1.md#cotation-de-document)
  - [Cotation de Colis (Boîte) — simple](api-cotacao-v1.md#cotation-de-colis-boîte--simple)
  - [Cotation de Colis (Boîte) — avancée](api-cotacao-v1.md#cotation-de-colis-boîte--avancée)
  - [Cotation de Palette — par LLH](api-cotacao-v1.md#cotation-de-palette--par-llh)
  - [Cotation de Palette — par m³](api-cotacao-v1.md#cotation-de-palette--par-m³)
  - [Cotation avec Kit](api-cotacao-v1.md#cotation-avec-kit)
  - [Cotations non éligibles à l'expédition](api-cotacao-v1.md#cotations-non-éligibles-à-lexpédition)
- [Cotation (v2)](api-cotacao-v2.md)
  - [Scénarios de cotation v2](api-cotacao-v2.md#scénarios-de-cotation)
  - [Cotation d'Articles (exclusive à la v2)](api-cotacao-v2.md#cotation-darticles)
  - [Cotations non éligibles à l'expédition](api-cotacao-v2.md#cotations-non-éligibles-à-lexpédition)
- [Différence entre Cotation v1 et v2](api-comparativo-cotacao-remessa.md#comparatif-de-la-cotation)

### Expédition
- [Expédition (cotation v1)](api-remessa-v1.md)
  - [Cotations non éligibles à l'expédition](api-remessa-v1.md#cotations-non-éligibles-à-lexpédition)
  - [Mettre à jour une expédition](api-remessa-v1.md#endpoint--mettre-à-jour-une-expédition)
  - [Annuler une expédition](api-remessa-v1.md#endpoint--annuler-une-expédition)
- [Expédition (cotation v2)](api-remessa-v2.md)
- [Différence entre Expédition v1 et v2](api-comparativo-cotacao-remessa.md#comparatif-de-lexpédition)
- [Manifeste](api-manifesto.md)
- [Brouillon](api-rascunho.md)

### Après l'expédition
- [Suivi de l'Expédition](api-rastreio.md)
- [Documents de l'Expédition](api-documentos.md)
- [Planification d'Enlèvement](api-agendamento.md)

### Support
- [Boîte](api-caixa-v1.md)
- [Articles](api-itens-v1.md)
- [Contrat](api-contrato-v1.md)
- [Plateforme](api-plataforma.md)

## Référence rapide des endpoints

### Cotation
| Méthode | Route | Page |
|---|---|---|
| `POST` | `/v1/quotation` | [Cotation](api-cotacao-v1.md) |
| `GET` | `/v1/quotation/{quotation}/{level}` | [Cotation](api-cotacao-v1.md) |
| `POST` | `/v2/quotation` | [Cotation V2](api-cotacao-v2.md) |

### Expédition
| Méthode | Route | Page |
|---|---|---|
| `POST` | `/v1/shipping` | [Expédition](api-remessa-v1.md) |
| `POST` | `/v2/shipping` | [Expédition V2](api-remessa-v2.md) |
| `PUT` | `/v1/shipping/{shipping}` | [Expédition](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}` | [Expédition](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{channel_identifier_number}/{channel_code}` | [Expédition](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}/docs` | [Expédition](api-remessa-v1.md) |
| `POST` | `/v1/manifest` | [Manifeste](api-manifesto.md) |
| `POST` | `/v1/draft` | [Brouillon](api-rascunho.md) |
| `PUT` | `/v1/draft/{code}` | [Brouillon](api-rascunho.md) |
| `GET` | `/v1/draft` | [Brouillon](api-rascunho.md) |
| `DELETE` | `/v1/draft` | [Brouillon](api-rascunho.md) |

### Suivi et documents
| Méthode | Route | Page |
|---|---|---|
| `GET` | `/v1/shipping/{shipping}/status` | [Suivi](api-rastreio.md) |
| `GET` | `/v1/shipping/status/{channel_identifier_number}/{channel_code}` | [Suivi](api-rastreio.md) |
| `GET` | `/v1/shipping/{shipping}/docs` | [Documents](api-documentos.md) |
| `GET` | `/v1/shipping/docs/{channel_identifier_number}/{channel_code}` | [Documents](api-documentos.md) |

### Enlèvement
| Méthode | Route | Page |
|---|---|---|
| `POST` | `/v1/schedule` | [Planification](api-agendamento.md) |
| `GET` | `/v1/schedule/{shipping}` | [Planification](api-agendamento.md) |
| `DELETE` | `/v1/schedule/{shipping}/{reason}` | [Planification](api-agendamento.md) |

### Support
| Méthode | Route | Page |
|---|---|---|
| `GET` `POST` `DELETE` | `/v1/box` | [Boîte](api-caixa-v1.md) |
| `GET` `POST` `PUT` `DELETE` | `/v1/item` | [Articles](api-itens-v1.md) |
| `GET` | `/v1/contract` | [Contrat](api-contrato-v1.md) |
| `GET` | `/v1/platform/version` | [Plateforme](api-plataforma.md) |

## Environnements

| Environnement | URL de base | Documentation interactive |
|---|---|---|
| Production | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Préproduction | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

> Développez toujours en **préproduction**. Les nouvelles fonctionnalités y apparaissent en premier.
