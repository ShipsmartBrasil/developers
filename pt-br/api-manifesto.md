# Manifesto

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Executa **cotação + remessa + geração de documentos em uma única chamada síncrona**.
- A resposta já traz o número de rastreio e os documentos (AWB, e demais quando aplicável).
- É o fluxo indicado para operações que precisam da etiqueta **imediatamente**, sem esperar
  processamento assíncrono.

> ⚠️ **O manifesto é síncrono e irreversível.** Como ele cota e despacha na mesma requisição, não há
> etapa intermediária de revisão. Valide bem o payload antes de enviar.
>
> ⚠️ **Só aceita pagamento faturado** (`payment.type: billed`).

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
Content-Type: application/json
lang: pt
```

## Manifesto vs. cotação + remessa

| Tema | Cotação + Remessa | Manifesto |
|---|---|---|
| Chamadas | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Escolha do transportador | Você escolhe na lista retornada | A plataforma resolve (ou você fixa com `carrier_code`/`slug`) |
| Documentos | Consulta posterior em `/docs` | **Vêm na resposta** |
| Pagamento | `billed`, `gateway`, `afterward` | **Apenas `billed`** |
| Tipo de resposta | Assíncrono | **Síncrono** |
| Revisão antes do despacho | Sim | Não |
| Campos obrigatórios | Menos (divididos em 2 etapas) | **Mais** (tudo de uma vez) |

Use o manifesto quando: a operação é de alto volume, o transportador já está definido por regra de
negócio e você precisa imprimir a etiqueta na hora.

## Fluxo de uso
1. Monte o payload completo (endereços, caixas, itens, pagamento).
2. `POST /v1/manifest?format=pdf` (ou `zpl`).
3. Imprima a etiqueta recebida em `data.docs`.
4. Use o `data.shipping` para [rastreio](api-rastreio.md), [documentos](api-documentos.md) e
   [agendamento de coleta](api-agendamento.md).

## Endpoint: criar manifesto

- **Método e rota:** `POST /v1/manifest`

### Parâmetros de query

| Nome | Local | Tipo | Obrigatório | Padrão | Valores | Descrição |
|---|---|---|---|---|---|---|
| `format` | query | string | não | todos | `pdf`, `zpl` | Formato dos documentos na resposta. Vazio retorna todos os formatos disponíveis |

> `format` também pode ser enviado **no corpo** do payload. Quando enviado nos dois lugares,
> **o valor do corpo prevalece**.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Valores / Padrão | Descrição |
|---|---|---|---|---|---|---|
| `object` | string | **sim** | - | - | `doc`, `package`, `pallet`, `not_doc` | O que está sendo enviado |
| `type` | string | **sim** | - | 30 | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalidade do envio |
| `tax` | string | **sim** | - | - | `sender`, `receiver` | Quem paga os impostos |
| `insurance` | boolean | **sim** | - | - | - | Contratar seguro |
| `residential_delivery` | boolean | **sim** | - | - | - | Entrega em endereço residencial |
| `non_stackable` | boolean | **sim** | - | - | - | Volumes não empilháveis |
| `measurement` | string | **sim** | - | - | `metric`, `imperial` | Sistema de medida |
| `ior` | boolean | **sim** | - | - | - | Importer of Record |
| `currency_quote` | string | **sim** | 3 | 3 | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda da cotação |
| `currency_payment` | string | **sim** | 3 | 3 | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda do pagamento |
| `address_sender` | object | **sim** | - | - | - | Remetente (ver tabela) |
| `address_receiver` | object | **sim** | - | - | - | Destinatário (ver tabela) |
| `boxes` | array | **sim** | 1 item | - | - | Volumes (ver tabela) |
| `payment` | object | **sim** | - | - | - | Pagamento (ver tabela) |
| `items` | array | condicional | - | - | - | Obrigatório quando `object` **não** é `doc` e nenhuma caixa usa `kit`. **Proibido quando `object` é `doc`** |
| `carrier_code` | integer | não | - | - | - | Fixa o contrato a ser usado |
| `slug` | string | não | - | 20 | - | Fixa o contrato pelo identificador textual |
| `channel_code` | integer | não | - | - | - | Código do canal de venda cadastrado |
| `channel_identifier_number` | string | não | - | - | - | Seu identificador comercial. **Recomendado** |
| `nfkey` | string | não | 44 | 44 | - | Chave da NF-e |
| `description` | string | não | 5 | 155 | - | Descrição livre do manifesto |
| `declared_freight` | object | não | - | - | - | Frete declarado (ver tabela) |
| `biller` | object | não | - | - | - | Faturante exibido na invoice |
| `manufacturer` | object | não | - | - | - | Fabricante exibido na invoice |
| `extra` | object | não | - | - | - | Dados auxiliares em pares chave/valor (ver tabela) |
| `format` | string | não | - | - | `pdf`, `zpl` | Alternativa à query string |
| `size` | string | não | - | - | `label`, `letter` | Tamanho da etiqueta |

### Parâmetros — `address_sender` e `address_receiver`

Diferente da remessa v1/v2, aqui o endereço é informado **completo**, porque a cotação acontece
na mesma chamada.

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código do endereço cadastrado. **Não envie nenhum outro campo de endereço junto** |

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `country_code` | string | **sim** | 2 | 2 | - | País em ISO 3166-1 alpha-2 |
| `city` | string | condicional | 1 | 100 | - | Cidade. Obrigatória se `postal_code` não for informado |
| `postal_code` | string | condicional | - | 12 | - | CEP/ZIP. Obrigatório se `state` e `state_code` não forem informados |
| `state` | string | condicional | - | 100 | - | Nome do estado. Obrigatório se `state_code` e `postal_code` não forem informados |
| `state_code` | string | condicional | 1 | 10 | - | Sigla do estado, validada contra o país. Obrigatória se `state` e `postal_code` não forem informados |
| `street` | string | **sim** | 5 | 70 | - | Logradouro |
| `name` | string | **sim** | 1 | 70 | - | Nome do contato/razão social |
| `email` | string | **sim** | - | 60 | - | E-mail válido |
| `type` | string | **sim** | - | - | `pj` | `pf` ou `pj` |
| `foreign` | boolean | **sim** | - | - | - | Indica se a pessoa é estrangeira (usado para validar o `federal_tax_id`) |
| `federal_tax_id` | string | condicional | - | 20 | - | **Obrigatório quando o país é `BR` ou `AR`** |
| `phone` | string | não | 1 | 25 | - | Telefone |
| `phone_extension` | string | não | - | 5 | - | Ramal |
| `number` | string | não | 1 | 6 | - | Número do imóvel |
| `district` | string | não | 1 | 30 | - | Bairro |
| `complement` | string | não | 1 | 20 | - | Complemento |
| `description` | string | não | 1 | 155 | - | Rótulo interno do endereço |
| `state_tax_id` | string | não | - | 20 | - | Inscrição estadual |
| `company_name` | string | não | 5 | 50 | - | **Somente em `address_sender`.** Razão social exigida por alguns envios FedEx |

### Parâmetros — `boxes[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `kit` | string | alternativa às demais formas | - | 50 | SKU do kit cadastrado. Preenche a caixa **e** os itens daquela posição |
| `box_code` | integer | alternativa | - | - | Código da caixa cadastrada ([Caixa](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa | - | 50 | Identificador da caixa cadastrada |
| `name` | string | **sim** sem `kit`/`box_code`/`box_identifier` | 5 | 155 | Nome da caixa |
| `height` | number | **sim** sem `kit`/`box_code`/`box_identifier` | 1 | - | Altura (cm em `metric`, in em `imperial`) |
| `width` | number | **sim** sem `kit`/`box_code`/`box_identifier` | 1 | - | Largura |
| `depth` | number | **sim** sem `kit`/`box_code`/`box_identifier` | 1 | - | Profundidade |
| `weight` | number | **sim** quando `object` é `doc` | 0.001 | - | Peso do volume (kg em `metric`, lb em `imperial`) |
| `price` | number | **sim** quando `object` é `doc` | 0 | - | Valor declarado do volume |

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `box_id` | integer | **sim** | - | - | **Posição** da caixa no array `boxes` (1 = primeira) |
| `quantity` | integer | **sim** | 1 | 99 | Quantidade do item |
| `sku` | string | alternativa aos campos manuais | 1 | 50 | SKU do item cadastrado ([Itens](api-itens-v1.md)) |
| `name` | string | **sim** sem `sku` | 6 | 50 | Nome do item |
| `country_code` | string | **sim** sem `sku` | 2 | 2 | País de origem do item |
| `hscode` | string | **sim** sem `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **sim** sem `sku` | 0.001 | 99 | Peso unitário |
| `unit_value` | number | **sim** sem `sku` | 0 | - | Valor unitário |
| `description` | string | não | 5 | 450 | Descrição detalhada do item |
| `selling_unit_value` | number | condicional | 0 | - | Valor de venda unitário. **Necessário quando `ior` é `true`** |

> **Atenção às diferenças em relação à cotação:** aqui `quantity` tem máximo **99** (não 1000) e
> `description` aceita até **450** caracteres (não 155).

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|
| `payment` | object | **sim** | - | Envelope |
| `payment.type` | string | **sim** | **`billed`** | O manifesto aceita **somente** pagamento faturado |

### Parâmetros — `declared_freight`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `declared` | boolean | não | - | - | Habilita o frete declarado. Com `true`, o valor pode ser `0` e os impostos são calculados com frete declarado zero |
| `value` | number | não | 0 | - | Valor do frete declarado |
| `currency` | string | **sim** quando `value` é informado | 3 | 3 | Moeda do frete declarado |

### Parâmetros — `extra`

Pares chave/valor auxiliares, repassados ao processo do manifesto.

| Regra | Detalhe |
|---|---|
| Estrutura | Objeto **plano** (um nível, sem aninhamento) |
| Chaves | Apenas letras minúsculas e `_` |
| Valores | Texto, inteiro ou decimal |

| Chave conhecida | Tipo | Regra | Descrição |
|---|---|---|---|
| `federal_tax_provision` | number | - | Provisão de imposto federal |
| `state_tax_provision` | number | - | Provisão de imposto estadual |
| `usd_rate_provision` | number | > 0 | Provisão de câmbio do dólar |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parâmetros — `biller` e `manufacturer`

- `biller`: mesmo formato descrito em [Remessa (v1)](api-remessa-v1.md#parâmetros--biller-faturante).
  No manifesto, `biller.name` aceita até **355** caracteres.
- `manufacturer`: mesmo formato descrito em [Cotação (v1)](api-cotacao-v1.md#parâmetros--manufacturer).

### Payload mínimo

Manifesto de documento (`object: doc`) — `items` é proibido neste caso:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
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
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Envelope documentos",
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

### Payload máximo

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
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
  "description": "Manifesto do lote de marco",
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
    "description": "Centro de distribuicao SP",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpao 2",
    "company_name": "Loja Exemplo Comercio LTDA",
    "name": "Loja Exemplo LTDA",
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
    "description": "Escritorio Miami",
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
    "foreign": true
  },
  "boxes": [
    {
      "name": "Caixa master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao, manga curta, cor branca, tamanho P",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Tenis esportivo",
      "description": "Tenis de corrida masculino numero 44",
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
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "name": "Holding Exemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "type": "pj"
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

### Exemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "type": "pj", "federal_tax_id": "12345678000199", "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "type": "pf", "foreign": true},
    "boxes": [{"name": "Envelope documentos", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "271249639538",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.shipping` | string (UUID) | **Identificador da remessa.** Use em rastreio, documentos e agendamento |
| `data.carrier` | string | Transportador que processou o envio |
| `data.tracking_id` | string | Número de rastreio (AWB) |
| `data.docs` | object | Documentos gerados, no formato `nome → { url, base64 }` |

> Documentos adicionais (invoice, packing list, declaração) podem ser gerados em segundo plano.
> Consulte [Documentos de Remessa](api-documentos.md) alguns instantes depois para obter a lista completa.

## Erro do transportador

Quando o transportador recusa o envio, a resposta é `400` e traz o retorno bruto dele:

```json
{
  "status": "error",
  "message": "[1234] CARRIER - Carrier error message",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Campo | Descrição |
|---|---|
| `message` | `[código] TRANSPORTADOR - mensagem` retornada pelo transportador |
| `data` | Payload bruto do transportador. **A estrutura varia por transportador** |

> Registre `message` e `data` nos seus logs: é a única fonte do motivo técnico da recusa.

## Regras de negócio
- **Pagamento:** somente `billed`. Qualquer outro valor é rejeitado na validação.
- **`items` e `object: doc`:** com `object: doc`, o campo `items` **não pode** ser enviado.
  Com qualquer outro `object`, `items` é obrigatório — exceto quando alguma caixa usa `kit`.
- **Kits:** quando uma caixa informa `kit`, a caixa e os itens daquela posição vêm do cadastro,
  e `boxes[].weight`/`boxes[].price` são ignorados.
- **`state_code`:** validado contra o `country_code` do mesmo endereço.
- **`federal_tax_id`:** obrigatório para `BR` e `AR`.
- **`address_code`:** quando usado, **nenhum** outro campo de endereço pode ser enviado no mesmo objeto.
- **Predefinições da conta** são aplicadas da mesma forma que no rascunho: o payload sempre vence o preset.
  Informar `declared_freight` descarta o preset de frete declarado; informar
  `address_sender.country_code` descarta o preset de endereço do remetente.
- **`not_doc`** é aceito por compatibilidade e convertido internamente para `package`.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Erro do transportador ou regra de negócio | Ler `message` e `data`; ajustar o payload ou acionar o suporte |
| `403` | Sem permissão para criar manifesto | Solicitar a permissão ao suporte |
| `422` | Validação de campos | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | **Antes de repetir**, confira em [Rastreio](api-rastreio.md) se a remessa foi criada |

> ⚠️ **Nunca repita um manifesto automaticamente após `500`.** Como a chamada cria a remessa de verdade,
> uma repetição pode gerar um envio duplicado. Verifique primeiro pelo `channel_identifier_number`.

## Observações finais
- Informe sempre `channel_identifier_number`: é a sua proteção contra duplicidade e a forma de localizar
  a remessa se a resposta se perder.
- O manifesto concentra muita responsabilidade em uma chamada. Para integrações que podem esperar,
  o fluxo de [cotação](api-cotacao-v1.md) + [remessa](api-remessa-v1.md) é mais seguro.
- Para o fluxo oposto (payload parcial, validado depois), veja [Rascunho](api-rascunho.md).
