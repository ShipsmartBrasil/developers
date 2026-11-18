# Technical Integration Guide

> Integration guide for the **Shipsmart Global Public API**: international quotation, shipment
> creation, documents, tracking and pickup scheduling.

- [Back to the main README](../README.md)

## Start here

| Step | Page | What you learn |
|---|---|---|
| 1 | [Getting Started](api-primeiros-passos.md) | Environments, authentication, language, response format, errors and account presets |
| 2 | [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md) | Which version to use in your integration |
| 3 | [Quotation](api-cotacao-v1.md) | Calculate freight, insurance and taxes |
| 4 | [Shipment](api-remessa-v1.md) | Turn the quotation into a real shipment |
| 5 | [Shipment Tracking](api-rastreio.md) | Follow the shipment until delivery |
| 6 | [Shipment Documents](api-documentos.md) | Download the AWB, invoice and other documents |
| 7 | [Pickup Scheduling](api-agendamento.md) | Request the pickup of the package at your address |

## The main flow

```
Quotation  ──►  Shipment  ──►  Tracking  ──►  Documents  ──►  Pickup
```

1. **Quote:** [Quotation](api-cotacao-v1.md) or [Quotation V2](api-cotacao-v2.md)
2. **Ship:** [Shipment](api-remessa-v1.md) or [Shipment V2](api-remessa-v2.md)
3. **Follow:** [Shipment Tracking](api-rastreio.md)
4. **Document:** [Shipment Documents](api-documentos.md)
5. **Collect:** [Pickup Scheduling](api-agendamento.md)

## Ways to create a shipment

| Flow | Main route | Characteristic |
|---|---|---|
| [Quotation](api-cotacao-v1.md) + [Shipment](api-remessa-v1.md) | `POST /v1/quotation` → `POST /v1/shipping` | Standard flow. You choose the carrier among the quoted options |
| [Quotation V2](api-cotacao-v2.md) + [Shipment V2](api-remessa-v2.md) | `POST /v2/quotation` → `POST /v2/shipping` | For when only the country is known at quotation time |
| [Manifest](api-manifesto.md) | `POST /v1/manifest` | **Synchronous:** quotes, dispatches and returns the documents in a single call |
| [Draft](api-rascunho.md) | `POST /v1/draft` | **Asynchronous:** accepts a partial payload and evolves into quotation + shipment |

## Support APIs

| Page | What it is for |
|---|---|
| [Box](api-caixa-v1.md) | Register packaging and reference it in the quotation |
| [Items](api-itens-v1.md) | Register products (weight, dimensions, HS Code) and use them by SKU |
| [Contract](api-contrato-v1.md) | List the shipping contracts enabled for your account |
| [Platform](api-plataforma.md) | Test the token and connectivity; health check |

## Full index

### Introduction
- [Getting Started](api-primeiros-passos.md)
- [Quotation & Shipment: v1 vs v2](api-comparativo-cotacao-remessa.md)

### Quotation
- [Quotation](api-cotacao-v1.md)
- [Quotation V2](api-cotacao-v2.md)

### Shipment
- [Shipment](api-remessa-v1.md)
- [Shipment V2](api-remessa-v2.md)
- [Manifest](api-manifesto.md)
- [Draft](api-rascunho.md)

### Post-shipment
- [Shipment Tracking](api-rastreio.md)
- [Shipment Documents](api-documentos.md)
- [Pickup Scheduling](api-agendamento.md)

### Support
- [Box](api-caixa-v1.md)
- [Items](api-itens-v1.md)
- [Contract](api-contrato-v1.md)
- [Platform](api-plataforma.md)

## Quick endpoint reference

### Quotation
| Method | Route | Page |
|---|---|---|
| `POST` | `/v1/quotation` | [Quotation](api-cotacao-v1.md) |
| `GET` | `/v1/quotation/{quotation}/{level}` | [Quotation](api-cotacao-v1.md) |
| `POST` | `/v2/quotation` | [Quotation V2](api-cotacao-v2.md) |

### Shipment
| Method | Route | Page |
|---|---|---|
| `POST` | `/v1/shipping` | [Shipment](api-remessa-v1.md) |
| `POST` | `/v2/shipping` | [Shipment V2](api-remessa-v2.md) |
| `PUT` | `/v1/shipping/{shipping}` | [Shipment](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}` | [Shipment](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{channel_identifier_number}/{channel_code}` | [Shipment](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}/docs` | [Shipment](api-remessa-v1.md) |
| `POST` | `/v1/manifest` | [Manifest](api-manifesto.md) |
| `POST` | `/v1/draft` | [Draft](api-rascunho.md) |
| `PUT` | `/v1/draft/{code}` | [Draft](api-rascunho.md) |
| `GET` | `/v1/draft` | [Draft](api-rascunho.md) |
| `DELETE` | `/v1/draft` | [Draft](api-rascunho.md) |

### Tracking and documents
| Method | Route | Page |
|---|---|---|
| `GET` | `/v1/shipping/{shipping}/status` | [Tracking](api-rastreio.md) |
| `GET` | `/v1/shipping/status/{channel_identifier_number}/{channel_code}` | [Tracking](api-rastreio.md) |
| `GET` | `/v1/shipping/{shipping}/docs` | [Documents](api-documentos.md) |
| `GET` | `/v1/shipping/docs/{channel_identifier_number}/{channel_code}` | [Documents](api-documentos.md) |

### Pickup
| Method | Route | Page |
|---|---|---|
| `POST` | `/v1/schedule` | [Scheduling](api-agendamento.md) |
| `GET` | `/v1/schedule/{shipping}` | [Scheduling](api-agendamento.md) |
| `DELETE` | `/v1/schedule/{shipping}/{reason}` | [Scheduling](api-agendamento.md) |

### Support
| Method | Route | Page |
|---|---|---|
| `GET` `POST` `DELETE` | `/v1/box` | [Box](api-caixa-v1.md) |
| `GET` `POST` `PUT` `DELETE` | `/v1/item` | [Items](api-itens-v1.md) |
| `GET` | `/v1/contract` | [Contract](api-contrato-v1.md) |
| `GET` | `/v1/platform/version` | [Platform](api-plataforma.md) |

## Environments

| Environment | Base URL | Interactive documentation |
|---|---|---|
| Production | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Staging | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

> Always develop against **staging**. New features show up there first.
