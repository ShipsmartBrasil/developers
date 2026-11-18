# Cotação (v1)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Calcula frete, seguro e impostos para um envio internacional, em todos os transportadores habilitados na sua conta.
- É o **primeiro passo obrigatório** do fluxo: a remessa, o rastreio, os documentos e o agendamento de coleta
  são todos amarrados ao código de cotação (`quotation`).
- A v1 é a versão completa: aceita endereço detalhado na origem/destino e possui endpoint de reconsulta do resultado.

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
1. `POST /v1/quotation` com origem, destino, objeto, tipo e caixas/itens.
2. A resposta já traz o resultado calculado (`data.carriers`) e o código `data.quotation`.
3. Se precisar reconsultar (ou mudar o nível de detalhe), use `GET /v1/quotation/{quotation}/{level}`.
4. Escolha um transportador da lista e use `code` (ou `slug`) + `quotation` para criar a
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
| `doc` | Documento | Cotação por caixa, informando `weight` e `price` |
| `package` | Pacote/caixa | Fluxo mais comum |
| `pallet` | Palete | Permite informar `volume` em vez de altura/largura/profundidade |
| `not_doc` | Legado | Mantido por compatibilidade; é convertido internamente para `package`. **Não use em integrações novas** |

### `type` — como o cálculo é montado
| Valor | O que você envia | Quando usar |
|---|---|---|
| `simple` | Apenas `boxes[]` com `weight` e `price` | Simulação rápida de frete. **Não é elegível para remessa quando `object` é `package` ou `pallet`** |
| `advanced` | `boxes[]` (dimensões) + `items[]` com `box_id` apontando a posição da caixa | Fluxo padrão para remessa: você controla a distribuição dos itens nas caixas |
| `items` | Apenas `items[]` com `sku` (as caixas são escolhidas pela plataforma) | Quando os itens e as caixas já estão cadastrados e você quer que a plataforma faça o encaixe |

> **Atenção:** `type: simple` com `object: package` ou `object: pallet` serve apenas para simulação.
> Ao tentar criar a remessa, a API responde `Cotação Simples é inelegível para remessa!`.
> Para remessa use `advanced` ou `items`.

### `tax` — quem paga os impostos
| Valor | Significado |
|---|---|
| `sender` | DDP: o remetente paga impostos e taxas no destino |
| `receiver` | DDU: o destinatário paga impostos e taxas no destino |

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

Enviado como **query string** no `POST` e como **segmento de path** no `GET`.

## Endpoint: criar cotação

- **Método e rota:** `POST /v1/quotation`
- **Query string opcional:** `level=simple|detailed`
- **Quando usar:** sempre que precisar de preço, prazo e impostos, e antes de criar qualquer remessa.

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
| `ior` | boolean | não | - | - | `false` | `true`, `false` | Importer of Record. Quando `true`, `items[].selling_unit_value` passa a ser necessário |
| `residential_delivery` | boolean | não | - | - | `false` | `true`, `false` | Entrega em endereço residencial |
| `non_stackable` | boolean | não | - | - | `false` | `true`, `false` | Volumes não empilháveis |
| `description` | string | não | - | 155 | `null` | - | Descrição livre da cotação, para sua referência |
| `channel_code` | integer | não | - | - | preset da conta | - | Código do canal de venda cadastrado na sua conta |
| `channel_identifier_number` | string | não | - | - | `null` | - | Seu identificador comercial (nº do pedido). **Recomendado**: permite consultar status/documentos por canal |
| `channel_order_number` | string | não | - | 100 | `null` | - | Número do pedido no canal de venda |
| `declared_freight` | object | não | - | - | preset da conta | - | Frete declarado (ver tabela abaixo) |
| `manufacturer` | object | não | - | - | `null` | - | Fabricante exibido na Commercial Invoice (ver tabela abaixo) |
| `level` | string | não (query) | - | - | `simple` | `simple`, `detailed` | Nível de detalhe do resultado |

### Parâmetros — `address_sender` e `address_receiver`

Há **duas formas** de informar o endereço. Escolha uma e não misture:

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código de um endereço cadastrado na sua conta. Quando usado, não envie os demais campos |

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `country_code` | string | **sim** | 2 | 2 | País em ISO 3166-1 alpha-2 (ex.: `BR`, `US`) |
| `city` | string | **sim** | - | 100 | Cidade |
| `postal_code` | string | condicional | - | 12 | CEP/ZIP. Obrigatório se `state` não for informado. **Para endereços no Brasil é sempre obrigatório** |
| `state` | string | condicional | - | - | Nome do estado/província. Obrigatório se `state_code` e `postal_code` não forem informados |
| `state_code` | string | condicional | - | 10 | Sigla do estado/província (ex.: `SP`, `FL`). Obrigatório se `state` e `postal_code` não forem informados |

> **Regra prática:** informe sempre `country_code` + `city` + (`postal_code` **ou** `state_code`).
> Essa combinação atende todos os países.

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

> `weight` e `price` são **ignorados** quando a caixa usa `kit`.

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `quantity` | integer | **sim** | 1 | 1000 | Quantidade do item |
| `unit_value` | number | **sim** | - | - | Valor unitário na moeda da cotação |
| `box_id` | integer | **sim**, exceto quando `type` é `items` | - | - | **Posição** da caixa no array `boxes` (1 = primeira caixa, 2 = segunda...). Não é o `box_code` |
| `sku` | string | **sim** quando `type` é `items` | - | 50 | SKU do item cadastrado ([Itens](api-itens-v1.md)). Quando informado, os dados do cadastro são usados e os demais campos podem ser omitidos |
| `ean` | string | não | - | 50 | Código EAN do item |
| `name` | string | **sim** quando `sku` não é informado | 6 | 50 | Nome do item (aparece na invoice) |
| `description` | string | não | - | 155 | Descrição detalhada do item |
| `country_code` | string | **sim** quando `sku` não é informado | 2 | 2 | País de origem/fabricação do item |
| `hscode` | string | **sim** quando `sku` não é informado | 6 | 10 | NCM/HS Code do item. Define a tabela de impostos aplicada |
| `weight` | number | **sim** quando `sku` não é informado | 0.001 | - | Peso unitário do item |
| `selling_unit_value` | number | condicional | - | - | Valor de venda unitário. **Necessário quando `ior` é `true`** |
| `is_groupable` | boolean | não | - | - | Padrão `true`. Quando `false`, o item recebe uma caixa exclusiva no agrupamento `smartbox` |

> **Descrições iguais em itens diferentes** geram um `warning` na resposta
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> A cotação é feita, mas vale revisar as descrições para evitar retenção na aduana.

### Parâmetros — `declared_freight`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `value` | number | não | 0 | - | Valor do frete declarado |
| `currency` | string | **sim** quando `value` é informado | 3 | 3 | Moeda do frete declarado. Mesma lista de `currency_quote` |

### Parâmetros — `manufacturer`

Dados do fabricante impressos na Commercial Invoice. Envie `address_code` **ou** os campos manuais.

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa aos manuais | - | - | Código de endereço cadastrado. Quando usado, os demais campos são opcionais |
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

> Para itens com `hscode` iniciando em `64` (calçados), o nome do fabricante é exigido pela aduana.

### Payload mínimo

Cotação de documento, tipo simples — o menor payload aceito:

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
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
    "state_code": "FL",
    "postal_code": "33101"
  },
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

Para gerar remessa de pacote é preciso usar `advanced` (ou `items`) e declarar os itens:

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
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

Todos os campos aceitos pelo endpoint:

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
        "url_image": "https://example.s3.amazonaws.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
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
| `data.finished` | boolean | Indica se o processamento da cotação foi concluído |
| `data.date` | string | Data/hora da cotação |
| `data.channel` | string | Nome do canal de venda, quando `channel_code` foi informado |
| `data.warnings` | array | **Opcional.** Só aparece quando há alertas (ex.: descrições duplicadas, valores declarados altos) |
| `data.carriers[]` | array | Opções de envio calculadas |
| `carriers[].code` | integer | Código do contrato. Use em `carrier_code` na remessa |
| `carriers[].slug` | string | Identificador textual do contrato. Alternativa ao `carrier_code` na remessa |
| `carriers[].name` | string | Nome comercial do transportador |
| `carriers[].url_image` | string | Logo do transportador |
| `carriers[].valid_until` | string | **Validade da cotação.** A remessa precisa ser criada antes desta data/hora |
| `carriers[].transit_days` | integer | Prazo estimado de trânsito em dias |
| `carriers[].weight_details.type` | string | `real` (peso real) ou `cubed` (peso cubado) — qual foi usado na tarifação |
| `carriers[].weight_details.weight` | string | Peso considerado |
| `carriers[].freight` / `freight_final` | string | Frete antes / depois das taxas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / depois das taxas administrativas |
| `carriers[].tax` / `tax_final` | string | Impostos antes / depois das taxas administrativas |
| `carriers[].currency_quote_price` | string | Total na moeda da cotação |
| `carriers[].currency_quote_amount` | string | Total a pagar na moeda da cotação |
| `carriers[].currency_payment_amount` | string | Total a pagar na moeda de pagamento |

## Endpoint: consultar resultado da cotação

- **Método e rota:** `GET /v1/quotation/{quotation}/{level}`
- **Quando usar:** para reconsultar um resultado já calculado ou trocar o nível de detalhe.

| Parâmetro | Local | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|---|
| `quotation` | path | string | **sim** | UUID | Código retornado na criação |
| `level` | path | string | não | `simple`, `detailed` | Padrão `simple` |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

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

> Se o `quotation` não existir ou não pertencer à sua conta, a API responde `400` com
> `Não foi encontrada a cotação especificada!`.

## Regras de negócio

### Elegibilidade para remessa
Para que uma cotação possa gerar remessa, todas as condições abaixo precisam ser verdadeiras:

1. O `quotation` existe e pertence à sua conta.
2. A cotação **não está expirada** (respeite `valid_until` do transportador escolhido).
3. A cotação **ainda não foi usada** em outra remessa (`Cotação já foi utilizada em outra remessa!`).
4. A cotação foi criada na **mesma versão** da API usada para a remessa (v1 → v1).
5. Não é uma cotação `simple` de `package`/`pallet` (`Cotação Simples é inelegível para remessa!`).
6. O transportador informado está na lista retornada pela cotação
   (`Contrato especificado não foi listado para esta cotação!`).

### Restrições de carga
A cotação pode ser recusada com `400` quando:

| Mensagem | Causa |
|---|---|
| `A caixa :name tem a altura/largura/profundidade maior do que a permitida!` | Dimensão acima do limite dos contratos |
| `A caixa :name está com o peso maior que o permitido!` | Peso por volume acima do limite |
| `Remessa está com o peso acima do permitido!` | Peso total acima do limite |
| `Nenhum item foi associado para a caixa :name !` | Caixa sem item em `advanced` |
| `A cotação excedeu o máximo de itens permitido!` | Número de itens acima do limite |
| `Nenhum transportador encontrado para este usuário!` | Conta sem contrato habilitado |
| `Não há contratos que atendem as especificações do pacote enviado...` | Nenhum contrato cobre a combinação peso/dimensão/rota |
| `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Carga fora da faixa automática — fale com o time comercial |
| `Endereço de Remetente e Destinatário são iguais!` | Origem e destino idênticos |
| `O país de origem é o mesmo de destino!` | A API atende envios internacionais |
| `Não foi passado um postal code para o endereço do Brasil no :name !` | Endereço brasileiro sem CEP |
| `Este país de Origem/Destino está temporariamente inabilitado para envios!` | Rota bloqueada no momento |
| `Não existe tabela de duty para o país de destino :name com o hscode :hscode...` | HS Code sem tabela de imposto no destino — revise o `hscode` |

### Cadastros referenciados
Quando você usa códigos de cadastro, eles precisam existir e estar ativos:

| Campo | Cadastro | Página |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Caixa | [Caixa](api-caixa-v1.md) |
| `items[].sku` | Item | [Itens](api-itens-v1.md) |
| `address_*.address_code` | Endereço | Cadastrado no painel |
| `channel_code` | Canal de venda | Cadastrado no painel |
| `boxes[].kit` | Kit | Cadastrado no painel |

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio (restrição de carga, rota, contrato, HS Code) | Ler `message` e ajustar o cenário |
| `403` | Sem permissão de cotação | Solicitar a permissão ao suporte |
| `406` | Estrutura do payload incompatível | Revisar tipos e aninhamento |
| `422` | Validação de campos | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- A resposta do `POST` já traz o resultado calculado — não é obrigatório chamar o `GET` em seguida.
- `data.warnings` é informativo: a cotação foi feita, mas há risco aduaneiro a revisar.
- Para um payload de cotação ainda mais enxuto (apenas país e estado), veja [Cotação V2](api-cotacao-v2.md).
- Para cotação + remessa em uma única chamada síncrona, veja [Manifesto](api-manifesto.md).
