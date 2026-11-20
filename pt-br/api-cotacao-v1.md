# Cotação (v1)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Calcula frete, seguro e impostos para um envio internacional, em todos os transportadores habilitados na sua conta.
- É o **primeiro passo obrigatório** do fluxo: a remessa, o rastreio, os documentos e o agendamento de coleta
  são todos amarrados ao código de cotação (`quotation`).
- A v1 é a versão completa: recebe o endereço (país, cidade, CEP/estado) já na cotação, valida esse endereço
  e possui endpoint de reconsulta do resultado.
- Atende cinco cenários de carga: documento, pacote simples, pacote avançado, pallet por CLA e pallet por m³ —
  veja [Cenários de cotação](#cenários-de-cotação). A cotação **por itens** (só `sku`, sem caixas) é exclusiva da
  [Cotação (v2)](api-cotacao-v2.md#cotação-de-itens).

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
2. `POST /v1/quotation` com origem, destino, objeto, tipo e caixas/itens.
3. A resposta já traz o resultado calculado (`data.carriers`) e o código `data.quotation`.
4. Se precisar reconsultar (ou mudar o nível de detalhe), use `GET /v1/quotation/{quotation}/{level}`.
5. Confira se a cotação é [elegível para remessa](#cotações-inelegíveis-para-remessa).
6. Escolha um transportador da lista e use `code` (ou `slug`) + `quotation` para criar a
   [remessa](api-remessa-v1.md) **antes** do `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Conceitos essenciais

### `object` — o que está sendo enviado
| Valor | Significado | Observações |
|---|---|---|
| `doc` | Documento | Cotado por volume, informando `weight` e `price` de cada envelope/caixa |
| `package` | Pacote/caixa | Fluxo mais comum |
| `pallet` | Pallet | Aceita dimensões (CLA) **ou** `volume` em m³/ft³ |
| `not_doc` | Legado | Mantido por compatibilidade; é convertido internamente para `package`. **Não use em integrações novas** |

### `type` — como o cálculo é montado
| Valor | O que você envia | Quando usar |
|---|---|---|
| `simple` | Apenas `boxes[]` com `weight` e `price` | Documentos e simulações rápidas de frete. **Não é elegível para remessa quando `object` é `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensões) + `items[]` com `box_id` apontando a posição da caixa | Fluxo padrão para remessa: você controla a distribuição dos itens nas caixas |
| `items` | **Não disponível na v1** | Exclusivo da [Cotação (v2)](api-cotacao-v2.md#cotação-de-itens) |

### Matriz de elegibilidade

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Cotação e remessa | Não recomendado | ❌ Só na v2 |
| `package` | ⚠️ Só simulação (inelegível para remessa) | ✅ Cotação e remessa | ❌ Só na v2 |
| `pallet` | ⚠️ Só simulação (inelegível para remessa) | ✅ Cotação e remessa | ❌ Só na v2 |

> **Atenção:** `type: simple` com `object: package` ou `object: pallet` serve apenas para simulação.
> Ao tentar criar a remessa, a API responde `403` com `Cotação Simples é inelegível para remessa!`.
> Detalhes em [Cotações inelegíveis para remessa](#cotações-inelegíveis-para-remessa).

### `tax` — quem paga os impostos
| Valor | Significado |
|---|---|
| `sender` | DDP: o remetente paga impostos e taxas no destino |
| `receiver` | DDU: o destinatário paga impostos e taxas no destino |

> Se a sua conta tiver uma **predefinição de imposto** configurada, ela **substitui** o valor enviado em `tax`.
> Veja [Predefinições aplicadas na cotação](#predefinições-aplicadas-na-cotação).

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

Enviado como **query string** no `POST` e como **segmento de path** no `GET`.

## Endpoint: criar cotação

- **Método e rota:** `POST /v1/quotation`
- **Query string opcional:** `level=simple|detailed`
- **Quando usar:** sempre que precisar de preço, prazo e impostos, e antes de criar qualquer remessa.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Valores aceitos | Descrição |
|---|---|---|---|---|---|---|---|
| `object` | string | **sim** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | O que está sendo enviado |
| `type` | string | **sim** | - | - | - | `simple`, `advanced` | Como o cálculo é montado (`items` é exclusivo da v2) |
| `tax` | string | **sim** | - | - | - | `sender`, `receiver` | Quem paga os impostos (pode ser substituído pela predefinição da conta) |
| `insurance` | boolean | **sim** | - | - | - | `true`, `false` | Contratar seguro (pode ser ativado automaticamente pela conta) |
| `currency_quote` | string | **sim** | 3 | 3 | - | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda em que a cotação é apresentada e em que `unit_value`/`price` são informados |
| `currency_payment` | string | **sim** | 3 | 3 | - | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda em que o pagamento será feito |
| `address_sender` | object | **sim** | - | - | - | - | Origem (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | - | Destino (ver tabela abaixo) |
| `boxes` | array | **sim** | - | - | - | - | Volumes da cotação (ver tabela abaixo) |
| `items` | array | condicional | - | - | - | - | Obrigatório quando `type` **não** é `simple` e nenhuma caixa usa `kit` |
| `measurement` | string | não | - | - | preset da conta ou `metric` | `metric`, `imperial` | Sistema de medida de dimensões, pesos e volumes |
| `ior` | boolean | não | - | - | `false` | `true`, `false` | Importer of Record. Quando `true`, `items[].selling_unit_value` passa a ser considerado |
| `residential_delivery` | boolean | não | - | - | `false` | `true`, `false` | Entrega em endereço residencial |
| `non_stackable` | boolean | não | - | - | `false` | `true`, `false` | Volumes não empilháveis |
| `description` | string | não | - | 155 | `null` | - | Descrição livre da cotação, para sua referência |
| `channel_code` | integer | não | - | - | preset da conta | - | Código do canal de venda cadastrado e liberado para o seu usuário |
| `channel_identifier_number` | string | não | - | - | `null` | - | Seu identificador comercial (nº do pedido). **Recomendado**: permite consultar status/documentos e cancelar por canal |
| `channel_order_number` | string | não | - | 100 | `null` | - | Número do pedido no canal de venda |
| `declared_freight` | object | não | - | - | preset da conta | - | Frete declarado (ver tabela abaixo). Exige permissão específica |
| `manufacturer` | object | não | - | - | `null` | - | Fabricante exibido na Commercial Invoice (ver tabela abaixo) |
| `level` | string | não (query) | - | - | `simple` | `simple`, `detailed` | Nível de detalhe do resultado |

### Parâmetros — `address_sender` e `address_receiver`

Há **duas formas** de informar o endereço. Escolha uma e não misture:

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código de um endereço cadastrado no painel da sua conta. Quando usado, não envie os demais campos |

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `country_code` | string | **sim** | 2 | 2 | País em ISO 3166-1 alpha-2 (ex.: `BR`, `US`) |
| `city` | string | **sim** | - | 100 | Cidade |
| `postal_code` | string | condicional | - | 12 | CEP/ZIP. Obrigatório se `state` não for informado. **Para endereços no Brasil é sempre obrigatório** |
| `state` | string | condicional | - | - | Nome do estado/província. Obrigatório se `state_code` e `postal_code` não forem informados |
| `state_code` | string | condicional | - | - | Sigla do estado/província (ex.: `SP`, `FL`). Obrigatório se `state` e `postal_code` não forem informados |

> **Regra prática:** informe sempre `country_code` + `city` + `postal_code` (+ `state_code`, quando souber).
> Essa combinação atende todos os países e é a que gera o cálculo mais preciso.

> **Remetente cadastrado obrigatório em alguns tipos de conta:** contas de pessoa física e contas empresariais
> com um único centro de distribuição precisam informar o remetente pela **Forma A** (`address_code`).
> Sem ele, a API responde `406` com `Codigo de endereço do remetente não encontrado!`.

### Parâmetros — `boxes[]`

Cada item do array representa um volume (caixa, envelope ou pallet). Há **três formas** de descrever o volume:

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
| `name` | string | não | 4 | 60 | Nome/rótulo do volume (usado nas mensagens de erro) |
| `height` | number | **sim** (exceto pallet com `volume`) | 1 | - | Altura |
| `width` | number | **sim** (exceto pallet com `volume`) | 1 | - | Largura |
| `depth` | number | **sim** (exceto pallet com `volume`) | 1 | - | Profundidade (comprimento) |
| `volume` | number | condicional | 0.001 | - | **Somente quando `object` é `pallet`.** Alternativa a altura/largura/profundidade, em m³ (`metric`) ou ft³ (`imperial`) |

**Campos válidos nas formas B e C:**

| Campo | Tipo | Obrigatório | Mín. | Descrição |
|---|---|---|---|---|
| `weight` | number | **sim** quando `object` é `doc`/`package` **e** `type` é `simple` | > 0 | Peso do volume. Em `advanced`, quando omitido, é a soma dos pesos dos itens daquela caixa |
| `price` | number | **sim** quando `object` é `doc`/`package` **e** `type` é `simple` | > 0 | Valor declarado do volume. Em `advanced`, quando omitido, é a soma de `unit_value × quantity` dos itens daquela caixa |

> `weight` e `price` são **ignorados** quando a caixa usa `kit`.

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `quantity` | integer | **sim** | 1 | 1000 | Quantidade do item |
| `unit_value` | number | **sim** | > 0 | - | Valor unitário na moeda da cotação. Zero ou negativo é recusado |
| `box_id` | integer | **sim** | 1 | nº de caixas | **Posição** da caixa no array `boxes` (1 = primeira caixa, 2 = segunda...). Não é o `box_code` |
| `sku` | string | não | - | 50 | SKU do item cadastrado ([Itens](api-itens-v1.md)). Os campos omitidos são completados pelo cadastro |
| `ean` | string | não | - | 50 | Código EAN do item |
| `nsmp` | string | não | - | 50 | *Non Standard Manufacturer Product Id*: identificador do produto no fabricante. Impresso na invoice e usado nas informações regulatórias exigidas por alguns transportadores para destinos na União Europeia |
| `name` | string | **sim** quando `sku` não é informado | 6 | 50 | Nome do item (aparece na invoice) |
| `description` | string | não | - | 155 | Descrição detalhada do item |
| `country_code` | string | **sim** quando `sku` não é informado | 2 | 2 | País de origem/fabricação do item |
| `hscode` | string | **sim** quando `sku` não é informado | 6 | 10 | NCM/HS Code do item. Define a tabela de impostos aplicada |
| `weight` | number | **sim** quando `sku` não é informado | 0.001 | - | Peso unitário do item |
| `selling_unit_value` | number | não | - | - | Valor de venda unitário. Considerado quando `ior` é `true`; se omitido, usa o valor do cadastro do item e, na falta dele, o `unit_value` |
| `is_groupable` | boolean | não | - | - | Padrão `true`. Usado nos agrupamentos automáticos de caixas da cotação por itens (v2) |

> **Itens com o mesmo `name`** geram um `warning` na resposta
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> A cotação é feita, mas vale diferenciar os nomes para evitar retenção na aduana.

### Parâmetros — `declared_freight`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `value` | number | não | 0 | - | Valor do frete declarado |
| `currency` | string | **sim** quando `value` é informado | 3 | 3 | Moeda do frete declarado. Mesma lista de `currency_quote` |

> O frete declarado só é aplicado se o seu perfil tiver a **permissão de frete declarado**. Sem ela, o campo é
> ignorado em silêncio (a cotação é feita normalmente, sem frete declarado).

### Parâmetros — `manufacturer`

Dados do fabricante impressos na Commercial Invoice. Envie `address_code` **ou** os campos manuais.

| Campo | Tipo | Recomendado | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa aos manuais | - | - | Código de endereço cadastrado. Quando usado, os dados do cadastro substituem os campos manuais |
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

> A API **não rejeita** um `manufacturer` incompleto na cotação, mas os dados vão impressos na invoice:
> envie todos os campos marcados como recomendados para evitar problemas no desembaraço.
> Para itens com `hscode` iniciando em `64` (calçados), o nome do fabricante é exigido pela aduana.

### Payload mínimo

O menor payload aceito pelo endpoint (cotação de documento). Os payloads mínimos de cada tipo de carga estão
em [Cenários de cotação](#cenários-de-cotação).

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "postal_code": "33101"
  },
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
  "ior": true,
  "description": "Pedido do marketplace, lote de outubro",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    {
      "name": "Caixa master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 2.5,
      "price": 150
    },
    {
      "box_identifier": "CX0236"
    },
    {
      "box_code": 3
    },
    {
      "kit": "KIT-001"
    }
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
      "selling_unit_value": 49.9,
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
      "selling_unit_value": 349.9,
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

> Em `type: advanced`, **toda caixa precisa de pelo menos um item** (exceto a caixa de `kit`, que já traz os seus).
> Por isso o exemplo associa um item às caixas 1, 2 e 3.

### Exemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/quotation?level=detailed' \
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
    "address_sender": {"country_code": "BR", "city": "Diadema", "postal_code": "09941070"},
    "address_receiver": {"country_code": "US", "city": "Miami", "state_code": "FL", "postal_code": "33101"},
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
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
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
      }
    ]
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.quotation` | string (UUID) | **Código da cotação.** Guarde: é a chave do fluxo todo |
| `data.finished` | boolean | `true` quando a cotação **já foi usada** para criar uma remessa; `false` enquanto ainda está disponível |
| `data.date` | string | Data/hora da cotação (`AAAA-MM-DD HH:MM:SS`) |
| `data.description` | string | Descrição enviada em `description` |
| `data.channel` | string | Nome do canal de venda, quando há `channel_code` |
| `data.channel_identifier_number` | string | Identificador comercial enviado na cotação |
| `data.object` / `data.type` / `data.tax` / `data.insurance` | - | Valores efetivamente usados no cálculo (após predefinições da conta) |
| `data.currency_quote` / `data.currency_payment` | string | Moedas da cotação e do pagamento |
| `data.currency_quote_exchange_value` | string | **Só em `detailed`.** Câmbio aplicado à moeda da cotação |
| `data.currency_payment_exchange_value` | string | **Só em `detailed`.** Câmbio aplicado à moeda de pagamento |
| `data.warnings` | array | **Opcional.** Só aparece quando há alertas (ver [Alertas da cotação](#alertas-da-cotação-warnings)) |
| `data.carriers[]` | array | Opções de envio calculadas |
| `carriers[].code` | integer | Código do contrato. Use em `carrier_code` na remessa |
| `carriers[].slug` | string | Identificador textual do contrato. Alternativa ao `carrier_code` na remessa |
| `carriers[].name` | string | Nome comercial do transportador/contrato |
| `carriers[].url_image` | string | Logo do transportador |
| `carriers[].valid_until` | string | **Validade da cotação.** A remessa precisa ser criada antes desta data/hora |
| `carriers[].transit_days` | integer | Prazo estimado de trânsito em dias |
| `carriers[].weight_details.type` | string | `real` (peso real) ou `cubed` (peso cubado) — qual foi usado na tarifação |
| `carriers[].weight_details.weight` | string | Peso considerado |
| `carriers[].freight` / `freight_final` | string | Frete antes / depois das taxas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / depois das taxas administrativas |
| `carriers[].tax` / `tax_final` | string | Impostos antes / depois das taxas administrativas |
| `carriers[].*_subsidy_rule_code` | integer | **Só em `detailed`.** Código da regra de subsídio aplicada ao frete, seguro ou imposto (`null` quando não há) |
| `carriers[].*_details` | object | **Só em `detailed`.** Composição detalhada do frete, do seguro ou do imposto (`null` quando não há) |
| `carriers[].currency_quote_price` | string | Total na moeda da cotação, antes das taxas administrativas |
| `carriers[].currency_quote_amount` | string | Total a pagar na moeda da cotação |
| `carriers[].currency_payment_amount` | string | Total a pagar na moeda de pagamento |

## Endpoint: consultar resultado da cotação

- **Método e rota:** `GET /v1/quotation/{quotation}/{level}`
- **Quando usar:** para reconsultar um resultado já calculado, trocar o nível de detalhe ou saber se a cotação
  já foi usada (`finished`).

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|---|
| `quotation` | path | string | **sim** | UUID | Código retornado na criação |
| `level` | path | string | não | `simple`, `detailed` | Padrão `simple` |

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

> A resposta tem a mesma estrutura de `data` do `POST` (exceto `warnings`).

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Não foi encontrada a cotação especificada!` | O `quotation` não existe ou não pertence à sua conta |
| `400` | `Nivel de detalhe da cotação não existe` | `level` diferente de `simple`/`detailed` |
| `403` | `Has no permission!` | Seu perfil não tem permissão de consulta de cotação |

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
| [Itens](#cotação-de-itens) | `package` | `items` | - | - | **Exclusivo da v2** |

### Cotação de Documento

- **Quando usar:** envio de papéis/documentos sem valor comercial relevante (contratos, certidões, impressos).
- **Como montar:** `object: doc` + `type: simple`. Cada volume (envelope/caixa) informa dimensões, `weight` e `price`.
- **Não usa** `items[]`, `ior` nem `manufacturer`: documentos não passam pelo cálculo de impostos por item.
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  [Pacote — avançado](#cotação-de-pacote-caixa--avançado) (ou, na v2, [Itens](api-cotacao-v2.md#cotação-de-itens)).

#### Payload mínimo

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": true,
  "description": "Pedido com duas caixas",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
      "selling_unit_value": 49.9,
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
      "selling_unit_value": 349.9
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Dois pallets de pecas industriais",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Pallet por cubagem e pallet por medidas",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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

> **Exclusiva da [Cotação (v2)](api-cotacao-v2.md#cotação-de-itens).** A cotação por itens (`type: items`), em que você
> envia apenas o `sku` dos itens cadastrados e a plataforma monta as caixas, não faz parte do contrato da v1.
> Na v1, informe sempre as caixas e use [Pacote — avançado](#cotação-de-pacote-caixa--avançado) com `items[].sku`
> para aproveitar o cadastro de itens.

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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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

Uma cotação calculada com sucesso **nem sempre** pode virar remessa. Antes de chamar `POST /v1/shipping`,
confira as condições abaixo. Elas são verificadas nesta ordem no momento da remessa:

| # | Condição | Status | Mensagem quando falha |
|---|---|---|---|
| 1 | A cotação existe e foi criada na **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | A cotação pertence à sua conta | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Conta com imposto travado em DDP: cotação DDU (`tax: receiver`) sem pagamento aprovado | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | A cotação **ainda não foi usada** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | Não é `type: simple` com `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | A cotação **não expirou** (`valid_until` do transportador) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | O transportador escolhido está em `carriers[]` desta cotação | `404` | `Contrato especificado não foi listado para esta cotação!` |

> **Como evitar:** use `advanced` para pacote/pallet, crie a remessa logo após a cotação,
> use o `code`/`slug` retornado **nesta** cotação e faça **uma cotação por remessa**.
> Todas as validações da remessa estão em [Remessa (v1)](api-remessa-v1.md#cotações-inelegíveis-para-remessa).

## Regras de negócio

### Predefinições aplicadas na cotação
As [predefinições da conta](api-primeiros-passos.md#predefinições-da-conta-presets) interferem no cálculo:

| Predefinição | Efeito na cotação |
|---|---|
| Imposto (`tax`) | Quando configurada, **substitui** o `tax` enviado. Países de destino listados como exceção recebem o oposto do padrão. Se a conta estiver travada em DDP, o imposto é sempre `sender` |
| Seguro automático | Se ativo, o seguro é contratado automaticamente quando o valor declarado (convertido para USD) atinge o mínimo configurado, mesmo com `insurance: false` |
| Frete declarado | Usado quando `declared_freight` não é enviado. Exige permissão de frete declarado |
| Sistema de medida | Usado quando `measurement` não é enviado. Na falta dele, `metric` |
| Entrega residencial / não empilhável | Usados quando `residential_delivery` / `non_stackable` não são enviados |
| Canal de venda | Usado quando `channel_code` não é enviado |
| Agrupamento de caixas | Não se aplica à v1 (usado só na cotação por itens da v2) |

### Validação do endereço
Na v1 o endereço manual é **validado já na cotação**:

| Caso | Status | Mensagem |
|---|---|---|
| Endereço no Brasil sem `postal_code` | `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` |
| CEP brasileiro inexistente/inválido (quando `state_code` não é enviado) | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Endereço internacional não localizado | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Estado/província não reconhecido | `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |
| `address_code` inexistente ou de outra conta | `400` | `Nenhum endereco encontrado para o usuário especificado!` |
| Remetente e destinatário idênticos | `400` | `Endereço de Remetente e Destinatário são iguais!` |

### Restrições de carga
A cotação pode ser recusada quando a carga não se encaixa nos contratos da sua conta:

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Caixa sem item em `advanced` |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` maior que o número de caixas ou menor que 1 |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Item manual sem `box_id` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | `weight` acima do peso máximo da caixa cadastrada (`simple`) |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Soma dos itens acima do peso máximo da caixa cadastrada |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inexistente ou de outra conta |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` não cadastrado e dados manuais incompletos |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...` | `unit_value` igual ou menor que zero |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` de outra conta |
| `404` | `Nenhum transportador encontrado para este usuário!` | Conta sem contrato habilitado para a rota/objeto |
| `404` | `Não há contratos que atendem as especificações do pacote enviado...` | Nenhum contrato cobre a combinação peso/dimensão/rota |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Carga fora da faixa automática — fale com o time comercial |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Nenhum transportador retornou resultado |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item)...` | HS Code sem tabela de imposto no destino — revise o `hscode` |

> Recusas relacionadas a **peso** podem vir com `status: "error"` em vez de `"warning"`, mantendo o código HTTP
> da tabela. Trate pelo código HTTP e pela `message`.

### Alertas da cotação (`warnings`)
A cotação é feita, mas `data.warnings` aparece com:

| Alerta | Quando |
|---|---|
| `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` | Dois ou mais itens com o mesmo `name` |
| `Valores declarados dos itens podem ocasionar problemas aduaneiros` | Valor declarado acima de USD 800 com transportador expresso (DHL/FedEx) no resultado |

### Cadastros referenciados
Quando você usa códigos de cadastro, eles precisam existir, estar ativos e pertencer à sua conta:

| Campo | Cadastro | Onde gerenciar |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Caixa | [Caixa](api-caixa-v1.md) |
| `items[].sku` | Item | [Itens](api-itens-v1.md) |
| `boxes[].kit` | Kit | Painel da Shipsmart Global |
| `address_*.address_code` / `manufacturer.address_code` | Endereço | Painel da Shipsmart Global |
| `channel_code` | Canal de venda | Painel da Shipsmart Global |

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio (carga, caixa, item, kit, endereço) | Ler `message` e ajustar o cenário |
| `403` | Sem permissão de cotação ou canal de outra conta | Solicitar a permissão ao suporte / revisar `channel_code` |
| `404` | Endereço não localizado ou nenhum contrato elegível | Revisar endereço e cenário |
| `406` | Carga exige atendimento personalizado, ou remetente cadastrado obrigatório para o tipo de conta | Falar com o time comercial / usar `address_sender.address_code` |
| `422` | Validação de campos ou HS Code sem tabela de imposto | Corrigir os campos listados em `data` / revisar `hscode` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- A resposta do `POST` já traz o resultado calculado — não é obrigatório chamar o `GET` em seguida.
- `data.warnings` é informativo: a cotação foi feita, mas há risco aduaneiro a revisar.
- Para um payload de cotação ainda mais enxuto (apenas país e estado), veja [Cotação V2](api-cotacao-v2.md).
- Para escolher entre as versões, veja [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Para cotação + remessa em uma única chamada síncrona, veja [Manifesto](api-manifesto.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
