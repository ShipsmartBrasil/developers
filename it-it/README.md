# Guida Tecnica all'Integrazione

> Guida all'integrazione con l'**API Pubblica di Shipsmart Global**: quotazione internazionale, creazione di
> spedizioni, documenti, tracciamento e pianificazione del ritiro.

- [Torna al README principale](../README.md)

## Inizia da qui

| Passo | Pagina | Cosa impari |
|---|---|---|
| 1 | [Primi Passi](api-primeiros-passos.md) | Ambienti, autenticazione, lingua, formato della risposta, errori e preimpostazioni |
| 2 | [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md) | Quale versione usare nella tua integrazione |
| 3 | [Quotazione](api-cotacao-v1.md) | Calcolare nolo, assicurazione e imposte |
| 4 | [Spedizione](api-remessa-v1.md) | Trasformare la quotazione in una spedizione reale |
| 5 | [Tracciamento della Spedizione](api-rastreio.md) | Seguire la spedizione fino alla consegna |
| 6 | [Documenti della Spedizione](api-documentos.md) | Scaricare AWB, invoice e gli altri documenti |
| 7 | [Pianificazione del Ritiro](api-agendamento.md) | Richiedere il prelievo del collo presso il tuo indirizzo |

## Il flusso principale

```
Quotazione  ──►  Spedizione  ──►  Tracciamento  ──►  Documenti  ──►  Ritiro
```

1. **Quotare:** [Quotazione](api-cotacao-v1.md) o [Quotazione V2](api-cotacao-v2.md)
2. **Spedire:** [Spedizione](api-remessa-v1.md) o [Spedizione V2](api-remessa-v2.md)
3. **Seguire:** [Tracciamento della Spedizione](api-rastreio.md)
4. **Documentare:** [Documenti della Spedizione](api-documentos.md)
5. **Ritirare:** [Pianificazione del Ritiro](api-agendamento.md)

## Modi per effettuare una spedizione

| Flusso | Rotta principale | Caratteristica |
|---|---|---|
| [Quotazione](api-cotacao-v1.md) + [Spedizione](api-remessa-v1.md) | `POST /v1/quotation` → `POST /v1/shipping` | Flusso standard. Scegli il corriere tra le opzioni quotate |
| [Quotazione V2](api-cotacao-v2.md) + [Spedizione V2](api-remessa-v2.md) | `POST /v2/quotation` → `POST /v2/shipping` | Per quando al momento della quotazione si conosce solo il paese |
| [Manifesto](api-manifesto.md) | `POST /v1/manifest` | **Sincrono:** quota, spedisce e restituisce i documenti in una sola chiamata |
| [Bozza](api-rascunho.md) | `POST /v1/draft` | **Asincrono:** accetta un payload parziale ed evolve verso quotazione + spedizione |

## API di supporto

| Pagina | A cosa serve |
|---|---|
| [Scatola](api-caixa-v1.md) | Registrare imballaggi e farvi riferimento nella quotazione |
| [Articoli](api-itens-v1.md) | Registrare prodotti (peso, dimensioni, HS Code) e usarli tramite SKU |
| [Contratto](api-contrato-v1.md) | Elencare i contratti di trasporto abilitati sul tuo account |
| [Piattaforma](api-plataforma.md) | Testare token e connettività; health check |

## Indice completo

### Introduzione
- [Primi Passi](api-primeiros-passos.md)
- [Quotazione & Spedizione: v1 vs v2](api-comparativo-cotacao-remessa.md)

### Quotazione
- [Quotazione](api-cotacao-v1.md)
- [Quotazione V2](api-cotacao-v2.md)

### Spedizione
- [Spedizione](api-remessa-v1.md)
- [Spedizione V2](api-remessa-v2.md)
- [Manifesto](api-manifesto.md)
- [Bozza](api-rascunho.md)

### Post-spedizione
- [Tracciamento della Spedizione](api-rastreio.md)
- [Documenti della Spedizione](api-documentos.md)
- [Pianificazione del Ritiro](api-agendamento.md)

### Supporto
- [Scatola](api-caixa-v1.md)
- [Articoli](api-itens-v1.md)
- [Contratto](api-contrato-v1.md)
- [Piattaforma](api-plataforma.md)

## Riferimento rapido degli endpoint

### Quotazione
| Metodo | Rotta | Pagina |
|---|---|---|
| `POST` | `/v1/quotation` | [Quotazione](api-cotacao-v1.md) |
| `GET` | `/v1/quotation/{quotation}/{level}` | [Quotazione](api-cotacao-v1.md) |
| `POST` | `/v2/quotation` | [Quotazione V2](api-cotacao-v2.md) |

### Spedizione
| Metodo | Rotta | Pagina |
|---|---|---|
| `POST` | `/v1/shipping` | [Spedizione](api-remessa-v1.md) |
| `POST` | `/v2/shipping` | [Spedizione V2](api-remessa-v2.md) |
| `PUT` | `/v1/shipping/{shipping}` | [Spedizione](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}` | [Spedizione](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{channel_identifier_number}/{channel_code}` | [Spedizione](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}/docs` | [Spedizione](api-remessa-v1.md) |
| `POST` | `/v1/manifest` | [Manifesto](api-manifesto.md) |
| `POST` | `/v1/draft` | [Bozza](api-rascunho.md) |
| `PUT` | `/v1/draft/{code}` | [Bozza](api-rascunho.md) |
| `GET` | `/v1/draft` | [Bozza](api-rascunho.md) |
| `DELETE` | `/v1/draft` | [Bozza](api-rascunho.md) |

### Tracciamento e documenti
| Metodo | Rotta | Pagina |
|---|---|---|
| `GET` | `/v1/shipping/{shipping}/status` | [Tracciamento](api-rastreio.md) |
| `GET` | `/v1/shipping/status/{channel_identifier_number}/{channel_code}` | [Tracciamento](api-rastreio.md) |
| `GET` | `/v1/shipping/{shipping}/docs` | [Documenti](api-documentos.md) |
| `GET` | `/v1/shipping/docs/{channel_identifier_number}/{channel_code}` | [Documenti](api-documentos.md) |

### Ritiro
| Metodo | Rotta | Pagina |
|---|---|---|
| `POST` | `/v1/schedule` | [Pianificazione](api-agendamento.md) |
| `GET` | `/v1/schedule/{shipping}` | [Pianificazione](api-agendamento.md) |
| `DELETE` | `/v1/schedule/{shipping}/{reason}` | [Pianificazione](api-agendamento.md) |

### Supporto
| Metodo | Rotta | Pagina |
|---|---|---|
| `GET` `POST` `DELETE` | `/v1/box` | [Scatola](api-caixa-v1.md) |
| `GET` `POST` `PUT` `DELETE` | `/v1/item` | [Articoli](api-itens-v1.md) |
| `GET` | `/v1/contract` | [Contratto](api-contrato-v1.md) |
| `GET` | `/v1/platform/version` | [Piattaforma](api-plataforma.md) |

## Ambienti

| Ambiente | Base URL | Documentazione interattiva |
|---|---|---|
| Produzione | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Staging | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

> Sviluppa sempre in **staging**. Le nuove feature compaiono prima lì.
