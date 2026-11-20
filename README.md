# Shipsmart Global — Technical Integration Guide

Integration guide for the **Shipsmart Global Public API**: international quoting, shipment creation,
documents, tracking and pickup scheduling.

> ⚠️ All examples in this guide (payloads, responses, codes, documents and tokens) are **for illustration only**
> and are **not valid data**.

## Choose your language

| Language | Guide |
|---|---|
| 🇺🇸 English (US) | [en-us/README.md](en-us/README.md) |
| 🇧🇷 Português (Brasil) | [pt-br/README.md](pt-br/README.md) |
| 🇪🇸 Español | [es-es/README.md](es-es/README.md) |
| 🇫🇷 Français | [fr-fr/README.md](fr-fr/README.md) |
| 🇮🇹 Italiano | [it-it/README.md](it-it/README.md) |

## Environments

| Environment | Base URL | Interactive documentation |
|---|---|---|
| Production | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Staging | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

## What you will find in the guide

- **Getting started:** environments, authentication, response format, account presets, error handling
- **Quoting:** v1 and v2, with one section per scenario — document, box (simple and advanced),
  pallet (by LWH and by m³), kits and — **v2 only** — quotation by registered items (SKU only, no boxes),
  each with field-level reference and minimum/maximum payloads
- **Ineligible quotations:** which quotations cannot become a shipment, and why
- **Shipping:** v1 and v2, plus the synchronous (Manifest) and asynchronous (Draft) alternatives
- **After shipping:** tracking, document retrieval, updates, cancellation and pickup scheduling
- **Supporting APIs:** boxes, items, contracts and platform version
- **Error codes:** a consolidated list of every error message, with cause and fix

> All language versions have the same structure and content. Portuguese (Brazil) is the source language.
