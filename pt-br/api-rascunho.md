# Rascunho

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Permite enviar um envio **incompleto**: você manda o que já tem, e a plataforma valida depois.
- O rascunho é processado de forma **assíncrona** e, quando completo, evolui para cotação + remessa
  automaticamente.
- É o fluxo indicado quando a integração recebe os dados do pedido em etapas (ex.: o pedido chega antes
  do endereço final ou da nota fiscal).

> **Praticamente todos os campos são opcionais no envio.** A validação rígida acontece no momento do
> despacho. Enquanto faltarem dados, o rascunho fica com status `incomplete` e você pode completá-lo.

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

## Rascunho vs. outros fluxos

| Tema | Cotação + Remessa | Manifesto | Rascunho |
|---|---|---|---|
| Payload | Completo em 2 etapas | Completo em 1 chamada | **Parcial, em quantas etapas precisar** |
| Validação | Síncrona, por etapa | Síncrona, total | **Assíncrona, no despacho** |
| Resposta | Cotação com preços | Documentos prontos | Confirmação de recebimento |
| Pagamento | `billed`, `gateway`, `afterward` | `billed` | **`billed` ou `afterward`** |
| Correção posterior | Nova cotação | Não | **Sim, via `PUT`** |

## Ciclo de vida do rascunho

```
POST /v1/draft
     │
     ▼
  received  ──►  validating  ──►  incomplete  ──┐
                      │                          │ PUT /v1/draft/{code}
                      ▼                          │
                   complete  ◄───────────────────┘
                      │
                      ▼
                 processing  ──►  cotação + remessa
                      │
                      └──►  error
```

| Status | Significado | Pode editar? |
|---|---|---|
| `received` | Rascunho recebido, aguardando validação | Não |
| `validating` | Em validação | Não |
| `incomplete` | **Faltam dados.** O motivo está em `status_descricao` | **Sim** |
| `edited` | Rascunho atualizado via `PUT`, aguardando nova validação | Não |
| `complete` | Todos os dados presentes, pronto para despacho | Não |
| `processing` | Em processamento: gerando cotação e remessa | Não |
| `error` | Falha no processamento | Não |

> **Somente rascunhos com status `incomplete` podem ser editados.** Tentar editar em outro status
> retorna `400 Apenas rascunhos com o status incomplete podem ser editados`.

## Fluxo de uso
1. `POST /v1/draft` com os dados que você já tem.
2. Aguarde o processamento assíncrono.
3. `GET /v1/draft` para ver o status e, se `incomplete`, ler o `status_descricao`.
4. `PUT /v1/draft/{code}` para completar o que faltou.
5. Quando ficar `complete`, a plataforma gera a cotação e a remessa.
6. Acompanhe pelo [rastreio](api-rastreio.md) e baixe os [documentos](api-documentos.md).
7. `DELETE /v1/draft` para descartar rascunhos que não serão usados.

## Endpoint: criar rascunho

- **Método e rota:** `POST /v1/draft`
- **Processamento:** assíncrono.

### Parâmetros — raiz do payload

Nenhum campo é obrigatório, mas **pelo menos um** precisa ser enviado.

| Campo | Tipo | Mín. | Máx. | Padrão | Valores | Descrição |
|---|---|---|---|---|---|---|
| `object` | string | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` | O que está sendo enviado |
| `type` | string | - | 30 | `Commercial` | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalidade do envio |
| `tax` | string | - | - | preset da conta | `sender`, `receiver` | Quem paga os impostos |
| `insurance` | boolean | - | - | preset da conta | - | Contratar seguro |
| `measurement` | string | - | - | `metric` | `metric`, `imperial` | Sistema de medida |
| `ior` | boolean | - | - | `false` | - | Importer of Record |
| `residential_delivery` | boolean | - | - | `false` | - | Entrega residencial |
| `non_stackable` | boolean | - | - | `false` | - | Volumes não empilháveis |
| `currency_quote` | string | 3 | 3 | preset da conta | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda da cotação |
| `currency_payment` | string | 3 | 3 | preset da conta | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda do pagamento |
| `nfkey` | string | 44 | 44 | - | - | Chave da NF-e |
| `description` | string | - | 155 | - | - | Descrição livre do rascunho |
| `channel_code` | integer | - | - | preset da conta | - | Código do canal de venda cadastrado |
| `channel_identifier_number` | string | - | - | - | - | Seu identificador comercial. **Recomendado** |
| `carrier_code` | integer | - | - | - | - | Contrato preferido, se disponível na cotação |
| `address_sender` | object | - | - | preset da conta | - | Remetente (ver tabela) |
| `address_receiver` | object | - | - | - | - | Destinatário (ver tabela) |
| `boxes` | array | - | - | - | - | Volumes (ver tabela) |
| `items` | array | - | - | - | - | Itens (ver tabela) |
| `payment` | object | - | - | - | - | Pagamento (ver tabela) |
| `biller` | object | - | - | preset da conta | - | Faturante exibido na invoice |
| `manufacturer` | object | - | - | - | - | Fabricante exibido na invoice |

### Parâmetros — `address_sender` e `address_receiver`

Todos opcionais no envio. As colunas "Obrigatório no despacho" indicam o que a plataforma exige para o
rascunho virar `complete`.

| Campo | Tipo | Mín. | Máx. | Obrigatório no despacho | Descrição |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Código de endereço cadastrado (apenas em `address_sender`) |
| `country_code` | string | 2 | 2 | **Sim** | País em ISO 3166-1 alpha-2 |
| `city` | string | - | 100 | **Sim** | Cidade |
| `postal_code` | string | - | 12 | Sim, se `state` não informado | CEP/ZIP |
| `state` | string | - | - | Sim, se `state_code`/`postal_code` não informados | Nome do estado |
| `state_code` | string | - | - | Sim, se `state`/`postal_code` não informados | Sigla do estado, validada contra o país |
| `street` | string | - | 70 | **Sim** | Logradouro |
| `name` | string | 5 | 35 | **Sim** | Nome do contato/razão social |
| `email` | string | - | 60 | **Sim** | E-mail válido |
| `phone` | string | - | 25 | **Sim** | Telefone |
| `type` | string | - | - | **Sim** | `pf` ou `pj` (padrão `pj`) |
| `federal_tax_id` | string | - | 20 | Sim quando o país é `BR` | CPF/CNPJ ou documento federal |
| `state_tax_id` | string | - | 20 | Sim quando o país é `BR` | Inscrição estadual |
| `number` | string | - | 100 | - | Número do imóvel |
| `district` | string | - | 30 | - | Bairro |
| `complement` | string | - | 20 | - | Complemento |
| `description` | string | - | 155 | - | Rótulo interno do endereço |
| `phone_extension` | string | - | 5 | - | Ramal |
| `foreign` | boolean | - | - | - | Indica se a pessoa é estrangeira |

### Parâmetros — `boxes[]`

| Campo | Tipo | Mín. | Máx. | Descrição |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU do kit cadastrado. Preenche a caixa **e** os itens daquela posição no despacho |
| `box_code` | integer | - | - | Código da caixa cadastrada ([Caixa](api-caixa-v1.md)) |
| `box_identifier` | string | - | - | Identificador da caixa cadastrada |
| `name` | string | 4 | 155 | Nome da caixa |
| `height` | number | 1 | - | Altura |
| `width` | number | 1 | - | Largura |
| `depth` | number | 1 | - | Profundidade |
| `volume` | number | 0.001 | - | Volume (alternativa às dimensões, para `pallet`) |
| `weight` | number | 0.1 | - | Peso do volume. **Obrigatório quando `object` é `doc`.** Até 3 casas decimais. Ignorado quando a caixa usa `kit` |
| `price` | number | 0 | - | Valor declarado. **Obrigatório quando `object` é `doc`.** Ignorado quando a caixa usa `kit` |

### Parâmetros — `items[]`

| Campo | Tipo | Mín. | Máx. | Obrigatório no despacho | Descrição |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU do item cadastrado ([Itens](api-itens-v1.md)) |
| `ean` | string | - | 50 | - | Código EAN |
| `box_id` | integer | - | - | - | **Posição** da caixa no array `boxes` (1 = primeira) |
| `name` | string | 6 | 35 | Sim, sem `sku` | Nome do item |
| `description` | string | 4 | 155 | - | Descrição do item |
| `country_code` | string | 2 | 2 | Sim, sem `sku` | País de origem do item |
| `hscode` | string | 6 | 10 | Sim, sem `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Sim, sem `sku` | Peso unitário |
| `quantity` | integer | 1 | 1000 | **Sim** | Quantidade |
| `unit_value` | number | - | - | **Sim** | Valor unitário |
| `selling_unit_value` | number | - | - | Sim quando `ior` é `true` | Valor de venda unitário |
| `is_groupable` | boolean | - | - | - | Permite agrupar o item com outros |
| `height`, `width`, `depth` | number | 1 | - | - | Dimensões do item |
| `is_foldable` | boolean | - | - | - | Item dobrável (usado no agrupamento `smartbox`) |

> `items` passa a ser **obrigatório no despacho** quando `object` é `package`, `pallet` ou `not_doc`.

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório no despacho | Valores | Descrição |
|---|---|---|---|---|
| `payment.type` | string | **Sim** | `billed`, `afterward` | O rascunho **não aceita** `gateway` |

### Payload mínimo

O menor payload útil — qualquer campo serve, mas o rascunho precisa ter pelo menos um:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package"
}
```

Um exemplo mais realista, com o que normalmente já se tem no momento do pedido:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
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
  ],
  "payment": { "type": "billed" }
}
```

### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "carrier_code": 1,
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
  "description": "Rascunho do pedido do marketplace",
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
    "state_tax_id": "987654321",
    "foreign": true
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
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao manga curta branca",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
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
  "payment": { "type": "billed" }
}
```

### Exemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodao", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Necessário informar pelo menos um parâmetro para criar o rascunho` | Payload vazio |

## Endpoint: listar rascunhos

- **Método e rota:** `GET /v1/draft`
- **Itens por página:** 20

| Campo | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `drafts` | body | object | não | Envelope do filtro |
| `drafts.drafts_ids` | body | array | não | Lista de códigos de rascunho. **Vazio ou ausente retorna todos** |
| `page` | query | integer | não | Página da listagem |

> Este `GET` aceita um corpo JSON para o filtro. Se o seu cliente HTTP não suportar body em `GET`,
> liste todos e filtre do seu lado.

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "object": "package",
          "type": "Commercial",
          "tax": "sender",
          "insurance": true,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "payment": { "type": "billed" },
          "address_receiver": {
            "country_code": "US",
            "postal_code": "33101",
            "city": "Miami",
            "street": "Main Avenue",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf"
          },
          "items": [
            {
              "name": "Camiseta algodao",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9,
              "box_id": 1
            }
          ]
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender.country code é obrigatório. O campo address sender.street é obrigatório.",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `codigo` | integer | **Código do rascunho.** Use em `PUT /v1/draft/{code}` e em `drafts.drafts_ids` |
| `data` | object | O payload do rascunho, já mesclado com as predefinições da conta |
| `status` | string | Status atual (ver ciclo de vida) |
| `status_descricao` | string | **Quando `incomplete`, lista exatamente o que falta.** É o campo mais importante para corrigir o rascunho |
| `created_at`, `updated_at` | string | Datas de criação e última alteração |

## Endpoint: atualizar rascunho

- **Método e rota:** `PUT /v1/draft/{code}`
- **Restrição:** apenas rascunhos com status `incomplete`.
- **Estrutura do payload:** idêntica ao `POST`.

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `code` | path | integer | **sim** | Código do rascunho (`codigo` na listagem) |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "address_sender": {
      "country_code": "BR",
      "postal_code": "09941070",
      "city": "Diadema",
      "street": "Rua Principal",
      "name": "Loja Exemplo LTDA",
      "email": "contato@example.com",
      "phone": "11989677465",
      "type": "pj",
      "federal_tax_id": "12345678000199",
      "state_tax_id": "123456789012"
    }
  }'
```

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso."
}
```

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | O rascunho está em outro status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | `code` inválido ou de outra conta |

## Endpoint: remover rascunhos

- **Método e rota:** `DELETE /v1/draft`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `drafts` | object | **sim** | Envelope |
| `drafts.drafts_ids` | array | **sim** | Lista de códigos de rascunho a remover |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Apenas rascunhos da sua própria conta são removidos. Se nenhum código corresponder, a resposta é
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Regras de negócio

### Validação no despacho
Quando o rascunho é despachado, a plataforma aplica um conjunto **rígido** de regras. O que falta aparece
em `status_descricao`. Resumo dos campos exigidos:

| Grupo | Campos exigidos |
|---|---|
| Geral | `tax`, `insurance` |
| Remetente | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`state`, `state_code` ou `postal_code`) |
| Remetente BR | `federal_tax_id` e `state_tax_id` |
| Destinatário | Os mesmos campos do remetente |
| Itens (`object` ≠ `doc`) | `items` com `quantity` e `unit_value`; e `name`, `country_code`, `hscode`, `weight` quando não há `sku` |
| Caixas (`object` = `doc`) | `boxes[].weight` e `boxes[].price` |
| Pagamento | `payment.type` (`billed` ou `afterward`) |

### Fabricante
Se você enviar `manufacturer` **sem** `address_code`, os campos `name`, `country_code`, `state_code`,
`postal_code`, `city`, `street`, `email`, `phone` e `type` passam a ser obrigatórios no despacho.

Para itens com `hscode` iniciando em **`64`** (calçados), o `manufacturer.name` é obrigatório.

### Predefinições da conta
As predefinições são mescladas ao rascunho. **O payload sempre vence o preset**, com estas exceções:
- Informar `address_sender.country_code` descarta o preset de endereço do remetente.
- Se a conta tem o imposto travado (`tax locked`), o valor é forçado para `sender`.
- O preset de faturante só é aplicado quando você envia `biller` vazio.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Rascunho vazio, não encontrado ou não editável | Enviar ao menos um campo, conferir o `code` e o status |
| `403` | Sem permissão para a ação | Solicitar a permissão ao suporte |
| `422` | Validação de campos | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- O campo **`status_descricao` é o seu guia**: ele diz exatamente o que falta para o rascunho avançar.
- Informe sempre `channel_identifier_number`: é o que amarra o rascunho ao seu pedido e permite
  consultar a remessa depois.
- O rascunho **não aceita** pagamento via `gateway`. Para esse cenário use
  [cotação](api-cotacao-v1.md) + [remessa](api-remessa-v1.md).
- Para o fluxo oposto (tudo validado de uma vez, documentos imediatos), veja [Manifesto](api-manifesto.md).
