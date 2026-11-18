# Cotação (v2)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Mesma engine de cálculo da [Cotação v1](api-cotacao-v1.md), com um **contrato de endereço muito mais enxuto**.
- Use a v2 quando, no momento da cotação, você ainda **não tem** cidade e CEP do destinatário — apenas o país
  (e opcionalmente o estado).
- A cidade, o CEP e o endereço completo são informados depois, na [Remessa V2](api-remessa-v2.md).

> **Importante:** uma cotação criada na v2 só pode gerar remessa na v2 (`POST /v2/shipping`).
> Veja o [comparativo](api-comparativo-cotacao-remessa.md) para escolher a versão.

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
Detalhes em [Primeiros Passos](api-primeiros-passos.md).

## Fluxo de uso
1. `POST /v2/quotation` informando apenas país (e estado, se souber) na origem e no destino.
2. A resposta já traz o resultado calculado (`data.carriers`) e o código `data.quotation`.
3. Escolha um transportador e crie a [Remessa V2](api-remessa-v2.md) com o endereço completo,
   **antes** do `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
   (só país/estado)            │
                               ▼
                    POST /v2/shipping
              (endereço completo + carrier_code|slug)
```

## Diferenças em relação à v1

| Tema | v1 | v2 |
|---|---|---|
| Endereço na cotação | `country_code` + `city` + (`postal_code` ou `state_code`/`state`) | Apenas `country_code` (+ `state_code` opcional) |
| Endereço na remessa | Logradouro, nome, e‑mail, telefone, tipo | Logradouro, nome, e‑mail, telefone, tipo **+ cidade, estado e CEP** |
| Consulta do resultado | `GET /v1/quotation/{quotation}/{level}` | **Não existe.** O resultado vem no próprio `POST` |
| `ior` (Importer of Record) | Suportado | **Não faz parte do contrato da v2** |
| `items[].selling_unit_value` | Suportado | **Não faz parte do contrato da v2** |
| Precisão do cálculo | Maior: cidade/CEP permitem aplicar taxas regionais já na cotação | O valor pode ser refinado na remessa, quando o endereço completo é informado |

> Todo o restante (objeto, tipo, caixas, itens, kits, moedas, frete declarado, fabricante, warnings)
> é idêntico à v1.

## Conceitos essenciais

### `object` — o que está sendo enviado
| Valor | Significado | Observações |
|---|---|---|
| `doc` | Documento | Cotação por caixa, informando `weight` e `price` |
| `package` | Pacote/caixa | Fluxo mais comum |
| `pallet` | Palete | Permite informar `volume` em vez de altura/largura/profundidade |
| `not_doc` | Legado | Convertido internamente para `package`. **Não use em integrações novas** |

### `type` — como o cálculo é montado
| Valor | O que você envia | Quando usar |
|---|---|---|
| `simple` | Apenas `boxes[]` com `weight` e `price` | Simulação rápida. **Não é elegível para remessa quando `object` é `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensões) + `items[]` com `box_id` | Fluxo padrão para remessa |
| `items` | Apenas `items[]` com `sku` | Quando itens e caixas já estão cadastrados e a plataforma faz o encaixe |

### `measurement` — sistema de medida
| Valor | Dimensões | Peso | Volume |
|---|---|---|---|
| `metric` (padrão) | centímetros | quilogramas | m³ |
| `imperial` | polegadas | libras | ft³ |

### `level` — nível de detalhe do resultado
| Valor | Retorno |
|---|---|
| `simple` (padrão) | Lista enxuta de transportadores com totais |
| `detailed` | Inclui a composição detalhada (frete, seguro, imposto, antes e depois das taxas administrativas) |

Enviado como **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: criar cotação

- **Método e rota:** `POST /v2/quotation`
- **Query string opcional:** `level=simple|detailed`

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Valores aceitos | Descrição |
|---|---|---|---|---|---|---|---|
| `object` | string | **sim** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | O que está sendo enviado |
| `type` | string | **sim** | - | - | - | `simple`, `advanced`, `items` | Como o cálculo é montado |
| `tax` | string | **sim** | - | - | - | `sender`, `receiver` | Quem paga os impostos |
| `insurance` | boolean | **sim** | - | - | - | `true`, `false` | Contratar seguro |
| `currency_quote` | string | **sim** | 3 | 3 | - | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda em que a cotação é apresentada |
| `currency_payment` | string | **sim** | 3 | 3 | - | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda em que o pagamento será feito |
| `address_sender` | object | **sim** | - | - | - | - | Origem (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | - | Destino (ver tabela abaixo) |
| `boxes` | array | condicional | - | - | - | - | Obrigatório quando `type` **não** é `items` |
| `items` | array | condicional | - | - | - | - | Obrigatório quando `type` **não** é `simple` e nenhuma caixa usa `kit` |
| `measurement` | string | não | - | - | `metric` | `metric`, `imperial` | Sistema de medida |
| `residential_delivery` | boolean | não | - | - | `false` | `true`, `false` | Entrega em endereço residencial |
| `non_stackable` | boolean | não | - | - | `false` | `true`, `false` | Volumes não empilháveis |
| `description` | string | não | - | 155 | `null` | - | Descrição livre da cotação |
| `channel_code` | integer | não | - | - | preset da conta | - | Código do canal de venda cadastrado |
| `channel_identifier_number` | string | não | - | - | `null` | - | Seu identificador comercial (nº do pedido). **Recomendado** |
| `channel_order_number` | string | não | - | 100 | `null` | - | Número do pedido no canal de venda |
| `declared_freight` | object | não | - | - | preset da conta | - | Frete declarado (ver tabela abaixo) |
| `manufacturer` | object | não | - | - | `null` | - | Fabricante exibido na Commercial Invoice (ver tabela abaixo) |
| `level` | string | não (query) | - | - | `simple` | `simple`, `detailed` | Nível de detalhe do resultado |

### Parâmetros — `address_sender` e `address_receiver`

**Esta é a principal diferença da v2.** Escolha uma das duas formas:

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código de um endereço cadastrado na sua conta |

**Forma B — apenas país (e estado, se souber):**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `country_code` | string | **sim** | 2 | 2 | País em ISO 3166-1 alpha-2 (ex.: `BR`, `US`) |
| `state_code` | string | não | - | - | Sigla do estado/província (ex.: `SP`, `FL`). Quando informado, melhora a precisão do cálculo de impostos regionais |

> Não envie `city`, `postal_code` nem `state` na cotação v2 — esses dados são informados na remessa.

### Parâmetros — `boxes[]`

Cada item do array representa um volume. Há **três formas** de descrever a caixa:

**Forma A — kit cadastrado** (preenche caixa **e** itens automaticamente):

| Campo | Tipo | Obrigatório | Máx. | Descrição |
|---|---|---|---|---|
| `kit` | string | **sim** nesta forma | 50 | SKU do kit cadastrado. Os demais campos da caixa e os itens daquela posição são preenchidos pelo cadastro do kit |

**Forma B — caixa cadastrada:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `box_code` | integer | **sim** nesta forma | Código da caixa cadastrada ([Caixa](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa ao `box_code` | Identificador da caixa cadastrada (ex.: `CX0236`) |

**Forma C — caixa informada manualmente:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `name` | string | não | 4 | 60 | Nome/rótulo da caixa |
| `height` | number | **sim** | 1 | - | Altura |
| `width` | number | **sim** | 1 | - | Largura |
| `depth` | number | **sim** | 1 | - | Profundidade |
| `volume` | number | condicional | 0.001 | - | **Somente quando `object` é `pallet`.** Alternativa a altura/largura/profundidade |

**Campos válidos em qualquer forma (exceto `kit`):**

| Campo | Tipo | Obrigatório | Mín. | Descrição |
|---|---|---|---|---|
| `weight` | number | **sim** quando `object` é `doc`/`package` **e** `type` é `simple` | > 0 | Peso do volume |
| `price` | number | **sim** quando `object` é `doc`/`package` **e** `type` é `simple` | > 0 | Valor declarado do volume |

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `quantity` | integer | **sim** | 1 | 1000 | Quantidade do item |
| `unit_value` | number | **sim** | - | - | Valor unitário na moeda da cotação |
| `box_id` | integer | **sim**, exceto quando `type` é `items` | - | - | **Posição** da caixa no array `boxes` (1 = primeira caixa). Não é o `box_code` |
| `sku` | string | **sim** quando `type` é `items` | - | 50 | SKU do item cadastrado ([Itens](api-itens-v1.md)). Quando informado, os dados do cadastro são usados |
| `ean` | string | não | - | 50 | Código EAN do item |
| `name` | string | **sim** quando `sku` não é informado | 6 | 50 | Nome do item (aparece na invoice) |
| `description` | string | não | - | 155 | Descrição detalhada do item |
| `country_code` | string | **sim** quando `sku` não é informado | 2 | 2 | País de origem/fabricação do item |
| `hscode` | string | **sim** quando `sku` não é informado | 6 | 10 | NCM/HS Code. Define a tabela de impostos aplicada |
| `weight` | number | **sim** quando `sku` não é informado | 0.001 | - | Peso unitário do item |
| `is_groupable` | boolean | não | - | - | Padrão `true`. Quando `false`, o item recebe caixa exclusiva no agrupamento `smartbox` |

### Parâmetros — `declared_freight`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `value` | number | não | 0 | - | Valor do frete declarado |
| `currency` | string | **sim** quando `value` é informado | 3 | 3 | Moeda do frete declarado |

### Parâmetros — `manufacturer`

Dados do fabricante impressos na Commercial Invoice. Envie `address_code` **ou** os campos manuais.

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa aos manuais | - | - | Código de endereço cadastrado |
| `name` | string | **sim** sem `address_code` | 5 | 30 | Razão social/nome do fabricante |
| `country_code` | string | **sim** sem `address_code` | 2 | 2 | País |
| `state_code` | string | **sim** sem `address_code` | 1 | 10 | Sigla do estado |
| `state` | string | não | - | 155 | Nome do estado |
| `postal_code` | string | **sim** sem `address_code` | - | 12 | CEP/ZIP |
| `city` | string | **sim** sem `address_code` | - | 100 | Cidade |
| `street` | string | **sim** sem `address_code` | - | 70 | Logradouro |
| `number` | string | não | - | 6 | Número |
| `district` | string | não | - | 30 | Bairro |
| `complement` | string | não | - | 20 | Complemento |
| `email` | string | **sim** sem `address_code` | - | 60 | E-mail |
| `phone` | string | **sim** sem `address_code` | - | 25 | Telefone |
| `phone_extension` | string | não | - | 6 | Ramal |
| `type` | string | **sim** sem `address_code` | - | - | `pf` ou `pj` |
| `federal_tax_id` | string | não | - | 20 | CPF/CNPJ ou equivalente |
| `state_tax_id` | string | não | - | 20 | Inscrição estadual |

### Payload mínimo

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.5,
      "price": 100
    }
  ]
}
```

### Payload mínimo elegível para remessa

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
  "boxes": [
    { "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Camiseta algodao",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ]
}
```

### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "description": "Pedido do marketplace, lote de outubro",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao, manga curta, cor branca",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    }
  ],
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpao 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

### Exemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v2/quotation?level=detailed' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "USD",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodao", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}]
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2026-02-21 14:28:10",
    "description": null,
    "channel": "Marketplace Exemplo",
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_quote_exchange_value": "1",
    "currency_payment": "EUR",
    "currency_payment_exchange_value": "0.92",
    "warnings": [
      "Itens com descrições iguais podem causar problemas na fiscalização aduaneira"
    ],
    "carriers": [
      {
        "code": 1,
        "slug": "dhl-br",
        "name": "DHL",
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "insurance": "11.55",
        "insurance_final": "11.55",
        "tax": "24924.09",
        "tax_final": "24924.09",
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.quotation` | string (UUID) | **Código da cotação.** Use em `POST /v2/shipping` |
| `data.finished` | boolean | Indica se o processamento da cotação foi concluído |
| `data.warnings` | array | **Opcional.** Só aparece quando há alertas aduaneiros |
| `carriers[].code` | integer | Código do contrato. Use em `carrier_code` na remessa |
| `carriers[].slug` | string | Identificador textual do contrato. Alternativa ao `carrier_code` |
| `carriers[].valid_until` | string | **Validade da cotação.** Crie a remessa antes desta data/hora |
| `carriers[].transit_days` | integer | Prazo estimado de trânsito em dias |
| `carriers[].weight_details.type` | string | `real` ou `cubed` — qual peso foi usado na tarifação |
| `carriers[].freight` / `freight_final` | string | Frete antes / depois das taxas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / depois das taxas administrativas |
| `carriers[].tax` / `tax_final` | string | Impostos antes / depois das taxas administrativas |
| `carriers[].currency_payment_amount` | string | Total a pagar na moeda de pagamento |

## Regras de negócio

### Elegibilidade para remessa
1. O `quotation` existe e pertence à sua conta.
2. A cotação **não está expirada** (respeite `valid_until`).
3. A cotação **ainda não foi usada** em outra remessa.
4. A cotação foi criada na **v2** e será usada em `POST /v2/shipping`.
5. Não é uma cotação `simple` de `package`/`pallet`.
6. O transportador informado está na lista retornada pela cotação.

### Restrições de carga
Valem as mesmas restrições da v1 (dimensões, peso por volume, peso total, limite de itens,
disponibilidade de contrato, rota bloqueada, HS Code sem tabela de imposto).
Consulte a seção **Restrições de carga** em [Cotação (v1)](api-cotacao-v1.md#restrições-de-carga).

### Sem endpoint de consulta
A v2 **não possui** `GET /v2/quotation/{quotation}`. O resultado completo é entregue na resposta do `POST`,
portanto **armazene a resposta** se precisar reapresentá-la ao usuário.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio (restrição de carga, rota, contrato, HS Code) | Ler `message` e ajustar o cenário |
| `403` | Sem permissão de cotação | Solicitar a permissão ao suporte |
| `406` | Estrutura do payload incompatível | Revisar tipos e aninhamento |
| `422` | Validação de campos | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- A v2 troca precisão por simplicidade: o endereço completo entra só na remessa.
- Fluxos operacionais de **status, documentos, cancelamento e agendamento de coleta** são os mesmos
  endpoints `/v1/...`, mesmo para remessas criadas na v2 — veja
  [Rastreio](api-rastreio.md), [Documentos](api-documentos.md) e [Agendamento](api-agendamento.md).
- Guarde a resposta do `POST`: não há endpoint de reconsulta na v2.
