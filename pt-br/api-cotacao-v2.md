# Cotação (v2)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Mesmo cálculo da [Cotação v1](api-cotacao-v1.md), com um **contrato de endereço muito mais enxuto**.
- Use a v2 quando, no momento da cotação, você ainda **não tem** cidade e CEP — apenas o país
  (e opcionalmente o estado).
- A cidade, o CEP e o endereço completo são informados e validados depois, na [Remessa V2](api-remessa-v2.md).
- Atende os cenários de carga da v1 **e** a **cotação por itens** (`type: items`), **exclusiva da v2**: você envia
  apenas o `sku` dos itens cadastrados, sem informar caixas — veja [Cenários de cotação](#cenários-de-cotação).

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
1. Escolha o [cenário de cotação](#cenários-de-cotação) que corresponde à sua carga (`object` + `type`).
2. `POST /v2/quotation` informando apenas país (e estado, se souber) na origem e no destino.
3. A resposta já traz o resultado calculado (`data.carriers`) e o código `data.quotation`. **Guarde a resposta**:
   não há endpoint de reconsulta na v2.
4. Confira se a cotação é [elegível para remessa](#cotações-inelegíveis-para-remessa).
5. Escolha um transportador e crie a [Remessa V2](api-remessa-v2.md) com o endereço completo,
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
| Cotação por itens (`type: items`) | **Não disponível**: as caixas são sempre informadas | **Disponível**: só o `sku` dos itens cadastrados, sem caixas |
| Validação do endereço | Na cotação (CEP, cidade, estado) | Na remessa, quando o endereço completo é enviado |
| Endereço na remessa | Logradouro, nome, e‑mail, telefone, tipo | Logradouro, nome, e‑mail, telefone, tipo **+ cidade, estado e CEP** |
| Consulta do resultado | `GET /v1/quotation/{quotation}/{level}` | **Não existe.** O resultado vem no próprio `POST` |
| `ior` (Importer of Record) | Suportado | **Não faz parte do contrato da v2** (ignorado) |
| `items[].selling_unit_value` | Suportado | **Não faz parte do contrato da v2** (ignorado) |
| Precisão do cálculo | Maior: cidade/CEP permitem aplicar taxas regionais já na cotação | O valor considera país/estado; o endereço completo entra na remessa |

> Estas são as duas razões de existir da v2: **endereço reduzido** (só país/estado) e **cotação por itens**.
> Todo o restante (objeto, caixas, kits, moedas, frete declarado, fabricante, warnings, predefinições da conta) é idêntico à v1.

## Conceitos essenciais

### `object` — o que está sendo enviado
| Valor | Significado | Observações |
|---|---|---|
| `doc` | Documento | Cotado por volume, informando `weight` e `price` de cada envelope/caixa |
| `package` | Pacote/caixa | Fluxo mais comum |
| `pallet` | Pallet | Aceita dimensões (CLA) **ou** `volume` em m³/ft³ |
| `not_doc` | Legado | Convertido internamente para `package`. **Não use em integrações novas** |

### `type` — como o cálculo é montado
| Valor | O que você envia | Quando usar |
|---|---|---|
| `simple` | Apenas `boxes[]` com `weight` e `price` | Documentos e simulações rápidas. **Não é elegível para remessa quando `object` é `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensões) + `items[]` com `box_id` | Fluxo padrão para remessa |
| `items` | `items[]` com `sku` (e, conforme o agrupamento da conta, `boxes[]` candidatas) | **Exclusivo da v2.** Quando os itens já estão cadastrados e a plataforma monta as caixas |

### Matriz de elegibilidade

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Cotação e remessa | Não recomendado | Não recomendado |
| `package` | ⚠️ Só simulação (inelegível para remessa) | ✅ Cotação e remessa | ✅ Cotação e remessa |
| `pallet` | ⚠️ Só simulação (inelegível para remessa) | ✅ Cotação e remessa | ✅ Cotação e remessa |

### `tax` — quem paga os impostos
| Valor | Significado |
|---|---|
| `sender` | DDP: o remetente paga impostos e taxas no destino |
| `receiver` | DDU: o destinatário paga impostos e taxas no destino |

> Se a sua conta tiver uma **predefinição de imposto** configurada, ela **substitui** o valor enviado em `tax`.
> Veja [Predefinições aplicadas na cotação](api-cotacao-v1.md#predefinições-aplicadas-na-cotação).

### `measurement` — sistema de medida
| Valor | Dimensões | Peso | Volume |
|---|---|---|---|
| `metric` (padrão) | centímetros | quilogramas | m³ |
| `imperial` | polegadas | libras | ft³ |

### `level` — nível de detalhe do resultado
| Valor | Retorno |
|---|---|
| `simple` (padrão) | Lista enxuta de transportadores com totais |
| `detailed` | Inclui câmbio e a composição detalhada de frete, seguro e imposto, com as regras de subsídio aplicadas |

Enviado como **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: criar cotação

- **Método e rota:** `POST /v2/quotation`
- **Query string opcional:** `level=simple|detailed`
- **Quando usar:** quando, no momento de cotar, você conhece apenas o país (e talvez o estado) de origem e destino.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Valores aceitos | Descrição |
|---|---|---|---|---|---|---|---|
| `object` | string | **sim** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | O que está sendo enviado |
| `type` | string | **sim** | - | - | - | `simple`, `advanced`, `items` | Como o cálculo é montado (`items` só existe na v2) |
| `tax` | string | **sim** | - | - | - | `sender`, `receiver` | Quem paga os impostos (pode ser substituído pela predefinição da conta) |
| `insurance` | boolean | **sim** | - | - | - | `true`, `false` | Contratar seguro (pode ser ativado automaticamente pela conta) |
| `currency_quote` | string | **sim** | 3 | 3 | - | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda em que a cotação é apresentada e em que `unit_value`/`price` são informados |
| `currency_payment` | string | **sim** | 3 | 3 | - | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda em que o pagamento será feito |
| `address_sender` | object | **sim** | - | - | - | - | Origem (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | - | Destino (ver tabela abaixo) |
| `boxes` | array | condicional | - | - | - | - | Obrigatório quando `type` **não** é `items`. Em `type: items` com agrupamento `smartbox`, também é obrigatório |
| `items` | array | condicional | - | - | - | - | Obrigatório quando `type` **não** é `simple` e nenhuma caixa usa `kit` |
| `measurement` | string | não | - | - | preset da conta ou `metric` | `metric`, `imperial` | Sistema de medida |
| `residential_delivery` | boolean | não | - | - | `false` | `true`, `false` | Entrega em endereço residencial |
| `non_stackable` | boolean | não | - | - | `false` | `true`, `false` | Volumes não empilháveis |
| `description` | string | não | - | 155 | `null` | - | Descrição livre da cotação |
| `channel_code` | integer | não | - | - | preset da conta | - | Código do canal de venda cadastrado e liberado para o seu usuário |
| `channel_identifier_number` | string | não | - | - | `null` | - | Seu identificador comercial (nº do pedido). **Recomendado** |
| `channel_order_number` | string | não | - | 100 | `null` | - | Número do pedido no canal de venda |
| `declared_freight` | object | não | - | - | preset da conta | - | Frete declarado. Exige permissão específica |
| `manufacturer` | object | não | - | - | `null` | - | Fabricante exibido na Commercial Invoice |
| `level` | string | não (query) | - | - | `simple` | `simple`, `detailed` | Nível de detalhe do resultado |

### Parâmetros — `address_sender` e `address_receiver`

**Esta é a principal diferença da v2.** Escolha uma das duas formas e não misture:

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código de um endereço cadastrado no painel da sua conta |

**Forma B — apenas país (e estado, se souber):**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `country_code` | string | **sim** | 2 | 2 | País em ISO 3166-1 alpha-2 (ex.: `BR`, `US`) |
| `state_code` | string | não | - | - | Sigla do estado/província (ex.: `SP`, `FL`). Quando informado, melhora a precisão do cálculo de impostos regionais e é reaproveitado na remessa |

> Não envie `city`, `postal_code` nem `state` na cotação v2: esses campos são **ignorados** e devem ser
> informados na [Remessa V2](api-remessa-v2.md).

> **Remetente cadastrado obrigatório em alguns tipos de conta:** contas de pessoa física e contas empresariais
> com um único centro de distribuição precisam informar o remetente pela **Forma A** (`address_code`).
> Sem ele, a API responde `406` com `Codigo de endereço do remetente não encontrado!`.

### Parâmetros — `boxes[]`

Idêntico à v1. Há **três formas** de descrever o volume:

**Forma A — kit cadastrado** (preenche caixa **e** itens automaticamente):

| Campo | Tipo | Obrigatório | Máx. | Descrição |
|---|---|---|---|---|
| `kit` | string | **sim** nesta forma | 50 | SKU do kit cadastrado. A caixa e os itens daquela posição são preenchidos pelo cadastro do kit |

**Forma B — caixa cadastrada:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `box_code` | integer | **sim** nesta forma | Código da caixa cadastrada ([Caixa](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa ao `box_code` | Identificador da caixa cadastrada (ex.: `CX0236`) |

**Forma C — caixa informada manualmente:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `name` | string | não | 4 | 60 | Nome/rótulo do volume |
| `height` | number | **sim** (exceto pallet com `volume`) | 1 | - | Altura |
| `width` | number | **sim** (exceto pallet com `volume`) | 1 | - | Largura |
| `depth` | number | **sim** (exceto pallet com `volume`) | 1 | - | Profundidade (comprimento) |
| `volume` | number | condicional | 0.001 | - | **Somente quando `object` é `pallet`.** Alternativa a altura/largura/profundidade, em m³ ou ft³ |

**Campos válidos nas formas B e C:**

| Campo | Tipo | Obrigatório | Mín. | Descrição |
|---|---|---|---|---|
| `weight` | number | **sim** quando `object` é `doc`/`package` **e** `type` é `simple` | > 0 | Peso do volume. Em `advanced`/`items`, quando omitido, é a soma dos itens |
| `price` | number | **sim** quando `object` é `doc`/`package` **e** `type` é `simple` | > 0 | Valor declarado do volume. Em `advanced`/`items`, quando omitido, é a soma dos itens |

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `quantity` | integer | **sim** | 1 | 1000 | Quantidade do item |
| `unit_value` | number | **sim** | > 0 | - | Valor unitário na moeda da cotação. Zero ou negativo é recusado |
| `box_id` | integer | **sim**, exceto quando `type` é `items` | 1 | nº de caixas | **Posição** da caixa no array `boxes` (1 = primeira caixa). Não é o `box_code` |
| `sku` | string | **sim** quando `type` é `items` | - | 50 | SKU do item cadastrado ([Itens](api-itens-v1.md)). Os campos omitidos são completados pelo cadastro |
| `ean` | string | não | - | 50 | Código EAN do item |
| `nsmp` | string | não | - | 50 | *Non Standard Manufacturer Product Id*: identificador do produto no fabricante. Impresso na invoice e usado nas informações regulatórias exigidas por alguns transportadores para destinos na União Europeia |
| `name` | string | **sim** quando `sku` não é informado | 6 | 50 | Nome do item (aparece na invoice) |
| `description` | string | não | - | 155 | Descrição detalhada do item |
| `country_code` | string | **sim** quando `sku` não é informado | 2 | 2 | País de origem/fabricação do item |
| `hscode` | string | **sim** quando `sku` não é informado | 6 | 10 | NCM/HS Code. Define a tabela de impostos aplicada |
| `weight` | number | **sim** quando `sku` não é informado | 0.001 | - | Peso unitário do item |
| `is_groupable` | boolean | não | - | - | Padrão `true`. Quando `false`, o item recebe caixa exclusiva nos agrupamentos de `type: items` |

### Parâmetros — `declared_freight`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `value` | number | não | 0 | - | Valor do frete declarado |
| `currency` | string | **sim** quando `value` é informado | 3 | 3 | Moeda do frete declarado |

> Aplicado somente se o seu perfil tiver a **permissão de frete declarado**; caso contrário é ignorado em silêncio.

### Parâmetros — `manufacturer`

Idêntico à v1: envie `address_code` **ou** os campos manuais (`name`, `country_code`, `state_code`, `state`,
`postal_code`, `city`, `street`, `number`, `district`, `complement`, `email`, `phone`, `phone_extension`, `type`,
`federal_tax_id`, `state_tax_id`). Limites e recomendações na
[tabela do fabricante da v1](api-cotacao-v1.md#parâmetros--manufacturer).

| Campo | Tipo | Recomendado | Descrição |
|---|---|---|---|
| `address_code` | integer | alternativa aos manuais | Endereço cadastrado do fabricante |
| `name` | string | **sim** sem `address_code` | Razão social/nome do fabricante (5 a 30 caracteres) |

### Payload mínimo

O menor payload aceito pelo endpoint (cotação de documento):

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 30,
      "width": 25,
      "depth": 2,
      "weight": 0.5,
      "price": 10
    }
  ]
}
```

### Payload máximo

Todos os campos aceitos pelo endpoint, com as três formas de caixa:

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
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
    { "name": "Caixa master", "height": 20, "width": 20, "depth": 20, "weight": 2.5, "price": 150 },
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao, manga curta, cor branca",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Tenis esportivo",
      "description": "Tenis de corrida masculino",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    },
    {
      "box_id": 3,
      "sku": "CNC-010-M",
      "quantity": 2,
      "unit_value": 15
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
        "url_image": "https://files.example.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.45",
        "insurance_final": "11.45",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "28153.894",
        "tax_final": "28153.894",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "url_image": "https://files.example.com/contratos/fedex.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.55",
        "insurance_final": "11.55",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "24924.09",
        "tax_final": "24924.09",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Campos da resposta

Mesma estrutura da v1 — descrição completa em [Campos da resposta da v1](api-cotacao-v1.md#campos-da-resposta).
Os campos mais usados:

| Campo | Tipo | Descrição |
|---|---|---|
| `data.quotation` | string (UUID) | **Código da cotação.** Use em `POST /v2/shipping` |
| `data.finished` | boolean | `true` quando a cotação **já foi usada** para criar uma remessa |
| `data.warnings` | array | **Opcional.** Só aparece quando há alertas aduaneiros |
| `carriers[].code` | integer | Código do contrato. Use em `carrier_code` na remessa |
| `carriers[].slug` | string | Identificador textual do contrato. Alternativa ao `carrier_code` |
| `carriers[].valid_until` | string | **Validade da cotação.** Crie a remessa antes desta data/hora |
| `carriers[].transit_days` | integer | Prazo estimado de trânsito em dias |
| `carriers[].weight_details.type` | string | `real` ou `cubed` — qual peso foi usado na tarifação |
| `carriers[].freight` / `freight_final` | string | Frete antes / depois das taxas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / depois das taxas administrativas |
| `carriers[].tax` / `tax_final` | string | Impostos antes / depois das taxas administrativas |
| `carriers[].currency_quote_amount` | string | Total a pagar na moeda da cotação |
| `carriers[].currency_payment_amount` | string | Total a pagar na moeda de pagamento |

## Cenários de cotação

Cada cenário abaixo traz o que muda no payload, o payload mínimo e o payload máximo **daquele cenário**.
Os campos de raiz comuns (moedas, imposto, seguro, canal, frete declarado etc.) seguem a
[tabela da raiz do payload](#parâmetros--raiz-do-payload).

| Cenário | `object` | `type` | `boxes[]` | `items[]` | Elegível para remessa |
|---|---|---|---|---|---|
| [Documento](#cotação-de-documento) | `doc` | `simple` | Dimensões + `weight` + `price` | Não usa | ✅ |
| [Pacote — simples](#cotação-de-pacote-caixa--simples) | `package` | `simple` | Dimensões + `weight` + `price` | Não usa | ❌ só simulação |
| [Pacote — avançado](#cotação-de-pacote-caixa--avançado) | `package` | `advanced` | Dimensões (peso/valor opcionais) | Obrigatório, com `box_id` | ✅ |
| [Pallet — por CLA](#cotação-de-pallet--por-cla) | `pallet` | `advanced` | `height` + `width` + `depth` | Obrigatório, com `box_id` | ✅ |
| [Pallet — por m³](#cotação-de-pallet--por-m³) | `pallet` | `advanced` | `volume` | Obrigatório, com `box_id` | ✅ |
| [Itens](#cotação-de-itens) — **exclusivo da v2** | `package` | `items` | Depende do agrupamento da conta | Obrigatório, com `sku` | ✅ |

### Cotação de Documento

- **Quando usar:** envio de papéis/documentos sem valor comercial relevante (contratos, certidões, impressos).
- **Como montar:** `object: doc` + `type: simple`. Cada volume (envelope/caixa) informa dimensões, `weight` e `price`.
- **Não usa** `items[]` nem `manufacturer`: documentos não passam pelo cálculo de impostos por item.
- **É elegível para remessa.**

#### Payload mínimo

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0002",
  "channel_order_number": "334456",
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Envio de contratos assinados",
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "declared_freight": { "value": 20, "currency": "BRL" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Envelope A4", "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 },
    { "box_code": 7, "weight": 1.2, "price": 15 }
  ]
}
```

> Em `doc`, `weight` e `price` são obrigatórios **em todo volume**, inclusive quando a caixa é cadastrada (`box_code`/`box_identifier`).

### Cotação de Pacote (Caixa) — simples

- **Quando usar:** simulação rápida de frete quando você só sabe o peso e o valor de cada caixa.
- **Como montar:** `object: package` + `type: simple`. Cada caixa informa dimensões, `weight` e `price`. Sem itens.
- ⚠️ **Não é elegível para remessa.** Para enviar de fato, refaça a cotação como
  [Pacote — avançado](#cotação-de-pacote-caixa--avançado) ou [Itens](#cotação-de-itens).

#### Payload mínimo

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "SIM-2026-0001",
  "channel_order_number": "334457",
  "object": "package",
  "type": "simple",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": true,
  "description": "Simulacao de frete para a vitrine",
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "declared_freight": { "value": 30, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Caixa media", "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 },
    { "box_identifier": "CX0236", "weight": 3.5, "price": 180 },
    { "box_code": 3, "weight": 1, "price": 60 },
    { "kit": "KIT-001" }
  ]
}
```

> Na caixa de `kit`, o peso e as dimensões vêm do cadastro da caixa do kit e o `price` é a soma dos valores dos itens do kit.
> Se a caixa cadastrada tiver **peso máximo** configurado, o `weight` informado não pode ultrapassá-lo.

### Cotação de Pacote (Caixa) — avançado

- **Quando usar:** é o **fluxo padrão para remessa** de mercadorias. Você declara os itens e diz em qual caixa
  cada um está.
- **Como montar:** `object: package` + `type: advanced`. As caixas trazem as dimensões; cada item aponta a
  posição da sua caixa em `box_id` (1 = primeira caixa).
- O peso e o valor de cada caixa são calculados pela soma dos itens, se não forem informados.
- **Toda caixa precisa de ao menos um item**; um `box_id` fora do intervalo de caixas é recusado.
- **É elegível para remessa.**

#### Payload mínimo

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0003",
  "channel_order_number": "334458",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Pedido com duas caixas",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Caixa vestuario", "height": 30, "width": 25, "depth": 15, "weight": 1.2, "price": 120 },
    { "box_code": 3 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao, manga curta, cor branca",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 4,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Tenis esportivo",
      "description": "Tenis de corrida masculino",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9
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

> Item com `sku` cadastrado pode ser enviado só com `box_id`, `sku`, `quantity` e `unit_value`: o restante
> (nome, país, HS Code, peso, EAN, `nsmp`) é completado pelo [cadastro de itens](api-itens-v1.md).
> O que você enviar no payload tem prioridade sobre o cadastro.

### Cotação de Pallet — por CLA

- **Quando usar:** carga paletizada cujas medidas você conhece — **C**omprimento (`depth`), **L**argura (`width`)
  e **A**ltura (`height`).
- **Como montar:** `object: pallet` + `type: advanced`, com `height`, `width` e `depth` em cada pallet e os itens
  apontando o pallet em `box_id`.
- A plataforma calcula o volume (`C × L × A ÷ 1.000.000` = m³) a partir das medidas.
- O peso do pallet, se omitido, é a soma dos pesos dos itens. Informe `weight` para incluir o peso da base/embalagem.
- `type: simple` é aceito para simulação, mas é **inelegível para remessa**.

#### Payload mínimo

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 150, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Peca de reposicao industrial",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0001",
  "channel_order_number": "990011",
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Dois pallets de pecas industriais",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Pallet 01", "height": 150, "width": 100, "depth": 120, "weight": 280, "price": 900 },
    { "name": "Pallet 02", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Peca de reposicao industrial",
      "description": "Engrenagem de aco para maquina de embalagem",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Rolamento industrial",
      "description": "Rolamento de esferas em aco",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

### Cotação de Pallet — por m³

- **Quando usar:** carga paletizada da qual você só conhece o **volume** (cubagem), sem as medidas individuais.
- **Como montar:** `object: pallet` + `type: advanced`, com `volume` em cada pallet (m³ em `metric`, ft³ em `imperial`).
- A plataforma converte o volume em um cubo equivalente para a tarifação:
  `lado (cm) = ∛volume (m³) × 100`. Ex.: `1.0` m³ → 100 × 100 × 100 cm; `1.8` m³ → 121,64 × 121,64 × 121,64 cm.
- Se você enviar `volume` **e** as três medidas, as medidas (CLA) têm prioridade.
- Você pode misturar pallets por CLA e por m³ no mesmo payload (cada posição de `boxes[]` decide a sua forma).

#### Payload mínimo

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "volume": 1.8 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Peca de reposicao industrial",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0002",
  "channel_order_number": "990012",
  "object": "pallet",
  "type": "advanced",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Pallet por cubagem e pallet por medidas",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Pallet cubagem", "volume": 1.8, "weight": 300, "price": 900 },
    { "name": "Pallet medido", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Peca de reposicao industrial",
      "description": "Engrenagem de aco para maquina de embalagem",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Rolamento industrial",
      "description": "Rolamento de esferas em aco",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Pallet sem medidas **e** sem `volume` é recusado com `422` (validação) ou com
> `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³`.

### Cotação de Itens

- **Exclusiva da v2.** É um dos motivos para escolher a v2: o payload não precisa descrever o pacote (caixa).
- **Quando usar:** você já tem os produtos no [cadastro de itens](api-itens-v1.md) e quer que a plataforma
  monte as caixas sozinha.
- **Como montar:** `object: package` + `type: items`. Cada item informa `sku`, `quantity` e `unit_value`; **não**
  informe `box_id`.
- Como as caixas são montadas depende do **agrupamento** configurado na sua conta
  ([Primeiros Passos](api-primeiros-passos.md#agrupamento-de-caixas)):

| Agrupamento | `boxes[]` no payload | Pré-requisito no cadastro do item | Como as caixas são montadas |
|---|---|---|---|
| `default` | Não é usado | Item vinculado a uma caixa cadastrada (`box_code` ou `box_identifier`) | Itens agrupáveis que usam a mesma caixa são reunidos; itens não agrupáveis recebem uma caixa por unidade |
| `all_in_one` | Não é usado | Item com dimensões cadastradas | Itens agrupáveis vão para **uma** caixa virtual com o volume total; não agrupáveis recebem uma caixa por unidade |
| `smartbox` | **Obrigatório** (caixas candidatas) | Item com dimensões cadastradas | Os itens são encaixados nas caixas enviadas, considerando volume e dimensões com 10% de folga e `is_groupable`/`is_foldable` |

#### Payload mínimo

Agrupamento `default` ou `all_in_one`:

```json
{
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "items": [
    { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
  ]
}
```

#### Payload máximo

Agrupamento `smartbox` (caixas candidatas informadas):

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0004",
  "channel_order_number": "334459",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Pedido montado pelo cadastro de itens",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 25, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "name": "Caixa grande", "height": 40, "width": 40, "depth": 40 }
  ],
  "items": [
    {
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao, manga curta, cor branca",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "sku": "TEN-042-M",
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Erros típicos deste cenário: `O item com o sku :name não existe!` (`404`),
> `A caixa com o box_code/box_identifier :name não existe ou está desativada` (`404`),
> `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (`400`) e
> `Nenhuma caixa comporta o item :name (Smart Box).` (`400`).

### Cotação com Kit

Um **kit** é um conjunto pré-cadastrado no painel (uma caixa + itens com quantidades). Ao informar
`boxes[].kit`, aquela posição é substituída pela caixa do kit e os itens do kit são incluídos automaticamente.

- Pode ser combinado com caixas comuns no mesmo payload.
- Se **todas** as caixas forem kits, `items[]` pode ser omitido.
- Em `type: simple`, o kit vira uma caixa com `price` = soma dos itens do kit; em `advanced`, os itens do kit
  entram na cotação apontando para a posição do kit.

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "kit": "KIT-001" }
  ]
}
```

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `O kit :sku é inválido!` | Kit inexistente, inativo ou de outra conta |
| `400` | `A caixa do kit :sku é inválida!` | A caixa vinculada ao kit não existe ou foi desativada |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Algum item do kit foi removido do cadastro |

## Cotações inelegíveis para remessa

As mesmas condições da v1 valem para a v2, verificadas no `POST /v2/shipping` nesta ordem:

| # | Condição | Status | Mensagem quando falha |
|---|---|---|---|
| 1 | A cotação existe e foi criada na **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | A cotação pertence à sua conta | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Conta com imposto travado em DDP: cotação DDU (`tax: receiver`) sem pagamento aprovado | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | A cotação **ainda não foi usada** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | Não é `type: simple` com `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | A cotação **não expirou** (`valid_until` do transportador) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | O transportador escolhido está em `carriers[]` desta cotação | `404` | `Contrato especificado não foi listado para esta cotação!` |

> Na v2 o endereço completo só é validado na remessa. Um CEP ou endereço inválido informado na
> [Remessa V2](api-remessa-v2.md#validação-do-endereço-completo) também impede a criação.

## Regras de negócio

### Predefinições e restrições de carga
Valem as mesmas regras da v1:
[Predefinições aplicadas na cotação](api-cotacao-v1.md#predefinições-aplicadas-na-cotação),
[Restrições de carga](api-cotacao-v1.md#restrições-de-carga),
[Alertas da cotação](api-cotacao-v1.md#alertas-da-cotação-warnings) e
[Cadastros referenciados](api-cotacao-v1.md#cadastros-referenciados).

### Endereço sem validação na cotação
- Na v2, a cotação **não** valida CEP, cidade nem estado; apenas o país (e o `state_code`, quando enviado).
- Não há exigência de CEP para endereços no Brasil na cotação v2.
- O `state_code` informado na cotação é reaproveitado na remessa v2.

### Sem endpoint de consulta
A v2 **não possui** `GET /v2/quotation/{quotation}`. O resultado completo é entregue na resposta do `POST`,
portanto **armazene a resposta** se precisar reapresentá-la ao usuário.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio (carga, caixa, item, kit) | Ler `message` e ajustar o cenário |
| `403` | Sem permissão de cotação ou canal de outra conta | Solicitar a permissão ao suporte / revisar `channel_code` |
| `404` | Item/caixa não encontrado no agrupamento ou nenhum contrato elegível | Revisar cadastros e cenário |
| `406` | Carga exige atendimento personalizado, ou remetente cadastrado obrigatório para o tipo de conta | Falar com o time comercial / usar `address_sender.address_code` |
| `422` | Validação de campos ou HS Code sem tabela de imposto | Corrigir os campos listados em `data` / revisar `hscode` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- A v2 troca precisão por simplicidade: o endereço completo entra só na remessa.
- Fluxos operacionais de **status, documentos, cancelamento e agendamento de coleta** são os mesmos
  endpoints `/v1/...`, mesmo para remessas criadas na v2 — veja
  [Rastreio](api-rastreio.md), [Documentos](api-documentos.md) e [Agendamento](api-agendamento.md).
- Guarde a resposta do `POST`: não há endpoint de reconsulta na v2.
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
