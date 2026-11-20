# Guia de Integração Técnico

> Guia de integração com a **API Pública da Shipsmart Global**: cotação internacional, criação de
> remessas, documentos, rastreio e agendamento de coleta.

- [Voltar ao README principal](../README.md)

> ⚠️ Todos os exemplos deste guia (payloads, respostas, códigos, documentos e tokens) são **meramente ilustrativos**
> e **não são dados válidos**.

## Comece por aqui

| Passo | Página | O que você aprende |
|---|---|---|
| 1 | [Primeiros Passos](api-primeiros-passos.md) | Ambientes, autenticação, idioma, formato de resposta, erros e predefinições |
| 2 | [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md) | Qual versão usar na sua integração |
| 3 | [Cotação](api-cotacao-v1.md) | Calcular frete, seguro e impostos |
| 4 | [Remessa](api-remessa-v1.md) | Transformar a cotação em um envio real |
| 5 | [Rastreio de Remessa](api-rastreio.md) | Acompanhar o envio até a entrega |
| 6 | [Documentos de Remessa](api-documentos.md) | Baixar AWB, invoice e demais documentos |
| 7 | [Agendamento de Coleta](api-agendamento.md) | Pedir a retirada do volume no seu endereço |
| 8 | [Códigos de Erro](api-codigos-de-erro.md) | Tratar cada mensagem de erro da API |

## O fluxo principal

```
Cotação  ──►  Remessa  ──►  Rastreio  ──►  Documentos  ──►  Coleta
```

1. **Cotar:** [Cotação](api-cotacao-v1.md) ou [Cotação V2](api-cotacao-v2.md)
2. **Remeter:** [Remessa](api-remessa-v1.md) ou [Remessa V2](api-remessa-v2.md)
3. **Acompanhar:** [Rastreio de Remessa](api-rastreio.md)
4. **Documentar:** [Documentos de Remessa](api-documentos.md)
5. **Coletar:** [Agendamento de Coleta](api-agendamento.md)

## Formas de fazer remessa

| Fluxo | Rota principal | Característica |
|---|---|---|
| [Cotação](api-cotacao-v1.md) + [Remessa](api-remessa-v1.md) | `POST /v1/quotation` → `POST /v1/shipping` | Fluxo padrão. Você escolhe o transportador entre as opções cotadas |
| [Cotação V2](api-cotacao-v2.md) + [Remessa V2](api-remessa-v2.md) | `POST /v2/quotation` → `POST /v2/shipping` | Para quando só o país é conhecido no momento de cotar |
| [Manifesto](api-manifesto.md) | `POST /v1/manifest` | **Síncrono:** cota, despacha e devolve os documentos em uma chamada |
| [Rascunho](api-rascunho.md) | `POST /v1/draft` | **Assíncrono:** aceita payload parcial e evolui para cotação + remessa |

## APIs de apoio

| Página | Para que serve |
|---|---|
| [Caixa](api-caixa-v1.md) | Cadastrar embalagens e referenciá-las na cotação |
| [Itens](api-itens-v1.md) | Cadastrar produtos (peso, dimensões, HS Code) e usá-los por SKU |
| [Contrato](api-contrato-v1.md) | Listar os contratos de transporte habilitados na sua conta |
| [Plataforma](api-plataforma.md) | Testar token e conectividade; health check |

## Índice completo

### Introdução
- [Primeiros Passos](api-primeiros-passos.md)
- [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Códigos de Erro](api-codigos-de-erro.md)

### Cotação
- [Cotação (v1)](api-cotacao-v1.md)
  - [Cotação de Documento](api-cotacao-v1.md#cotação-de-documento)
  - [Cotação de Pacote (Caixa) — simples](api-cotacao-v1.md#cotação-de-pacote-caixa--simples)
  - [Cotação de Pacote (Caixa) — avançado](api-cotacao-v1.md#cotação-de-pacote-caixa--avançado)
  - [Cotação de Pallet — por CLA](api-cotacao-v1.md#cotação-de-pallet--por-cla)
  - [Cotação de Pallet — por m³](api-cotacao-v1.md#cotação-de-pallet--por-m³)
  - [Cotação com Kit](api-cotacao-v1.md#cotação-com-kit)
  - [Cotações inelegíveis para remessa](api-cotacao-v1.md#cotações-inelegíveis-para-remessa)
- [Cotação (v2)](api-cotacao-v2.md)
  - [Cenários de cotação v2](api-cotacao-v2.md#cenários-de-cotação)
  - [Cotação de Itens (exclusiva da v2)](api-cotacao-v2.md#cotação-de-itens)
  - [Cotações inelegíveis para remessa](api-cotacao-v2.md#cotações-inelegíveis-para-remessa)
- [Diferença entre Cotação v1 e v2](api-comparativo-cotacao-remessa.md#comparativo-da-cotação)

### Remessa
- [Remessa (cotação v1)](api-remessa-v1.md)
  - [Cotações inelegíveis para remessa](api-remessa-v1.md#cotações-inelegíveis-para-remessa)
  - [Atualizar remessa](api-remessa-v1.md#endpoint-atualizar-remessa)
  - [Cancelar remessa](api-remessa-v1.md#endpoint-cancelar-remessa)
- [Remessa (cotação v2)](api-remessa-v2.md)
- [Diferença entre Remessa v1 e v2](api-comparativo-cotacao-remessa.md#comparativo-da-remessa)
- [Manifesto](api-manifesto.md)
- [Rascunho](api-rascunho.md)

### Pós-remessa
- [Rastreio de Remessa](api-rastreio.md)
- [Documentos de Remessa](api-documentos.md)
- [Agendamento de Coleta](api-agendamento.md)

### Apoio
- [Caixa](api-caixa-v1.md)
- [Itens](api-itens-v1.md)
- [Contrato](api-contrato-v1.md)
- [Plataforma](api-plataforma.md)

## Referência rápida de endpoints

### Cotação
| Método | Rota | Página |
|---|---|---|
| `POST` | `/v1/quotation` | [Cotação](api-cotacao-v1.md) |
| `GET` | `/v1/quotation/{quotation}/{level}` | [Cotação](api-cotacao-v1.md) |
| `POST` | `/v2/quotation` | [Cotação V2](api-cotacao-v2.md) |

### Remessa
| Método | Rota | Página |
|---|---|---|
| `POST` | `/v1/shipping` | [Remessa](api-remessa-v1.md) |
| `POST` | `/v2/shipping` | [Remessa V2](api-remessa-v2.md) |
| `PUT` | `/v1/shipping/{shipping}` | [Remessa](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}` | [Remessa](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{channel_identifier_number}/{channel_code}` | [Remessa](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}/docs` | [Remessa](api-remessa-v1.md) |
| `POST` | `/v1/manifest` | [Manifesto](api-manifesto.md) |
| `POST` | `/v1/draft` | [Rascunho](api-rascunho.md) |
| `PUT` | `/v1/draft/{code}` | [Rascunho](api-rascunho.md) |
| `GET` | `/v1/draft` | [Rascunho](api-rascunho.md) |
| `DELETE` | `/v1/draft` | [Rascunho](api-rascunho.md) |

### Rastreio e documentos
| Método | Rota | Página |
|---|---|---|
| `GET` | `/v1/shipping/{shipping}/status` | [Rastreio](api-rastreio.md) |
| `GET` | `/v1/shipping/status/{channel_identifier_number}/{channel_code}` | [Rastreio](api-rastreio.md) |
| `GET` | `/v1/shipping/{shipping}/docs` | [Documentos](api-documentos.md) |
| `GET` | `/v1/shipping/docs/{channel_identifier_number}/{channel_code}` | [Documentos](api-documentos.md) |

### Coleta
| Método | Rota | Página |
|---|---|---|
| `POST` | `/v1/schedule` | [Agendamento](api-agendamento.md) |
| `GET` | `/v1/schedule/{shipping}` | [Agendamento](api-agendamento.md) |
| `DELETE` | `/v1/schedule/{shipping}/{reason}` | [Agendamento](api-agendamento.md) |

### Apoio
| Método | Rota | Página |
|---|---|---|
| `GET` `POST` `DELETE` | `/v1/box` | [Caixa](api-caixa-v1.md) |
| `GET` `POST` `PUT` `DELETE` | `/v1/item` | [Itens](api-itens-v1.md) |
| `GET` | `/v1/contract` | [Contrato](api-contrato-v1.md) |
| `GET` | `/v1/platform/version` | [Plataforma](api-plataforma.md) |

## Ambientes

| Ambiente | Base URL | Documentação interativa |
|---|---|---|
| Produção | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Homologação | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

> Desenvolva sempre em **homologação**. Features novas aparecem lá primeiro.
