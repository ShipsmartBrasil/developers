# Remessa (v2)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md) e [Cotação (v2)](api-cotacao-v2.md)

## Visão geral
- Transforma uma cotação **v2** em um envio real.
- A diferença central em relação à [Remessa v1](api-remessa-v1.md): aqui você informa o **endereço completo**
  (cidade, estado e CEP), porque a cotação v2 recebeu apenas o país.
- A v2 expõe **somente a criação**. Status, documentos, cancelamento, atualização e agendamento de coleta
  usam os endpoints `/v1/...`, que funcionam normalmente para remessas criadas na v2.

> ⚠️ **A remessa é uma operação irreversível na rota normal.** Após criada, ela entra em processamento
> no transportador.

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

## Fluxo de uso

```
1. POST /v2/quotation                 → quotation + carriers[]   (só país/estado)
2. POST /v2/shipping                  → remessa recebida          (endereço completo)
3. GET  /v1/shipping/{shipping}/status→ acompanhar até "shipped"
4. GET  /v1/shipping/{shipping}/docs  → baixar AWB, invoice etc.
5. POST /v1/schedule                  → agendar coleta (opcional)
```

1. [Crie a cotação v2](api-cotacao-v2.md) e verifique se ela é elegível para remessa.
2. Envie a remessa com o `quotation`, o transportador escolhido e o endereço completo.
3. A criação é **assíncrona**: `200` significa "recebemos", não "despachamos".
4. Acompanhe pelo [rastreio](api-rastreio.md) e baixe os [documentos](api-documentos.md).

## Mapa de endpoints por versão

| Operação | Rota |
|---|---|
| Criar remessa | `POST /v2/shipping` |
| Consultar status | `GET /v1/shipping/{shipping}/status` |
| Consultar status por canal | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` |
| Consultar documentos | `GET /v1/shipping/{shipping}/docs` |
| Consultar documentos por canal | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` |
| Atualizar (NF-e, anexos, centro de custo) | `PUT /v1/shipping/{shipping}` |
| Remover anexos | `DELETE /v1/shipping/{shipping}/docs` |
| Cancelar | `DELETE /v1/shipping/{shipping}` |
| Cancelar por canal | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` |
| Agendar coleta | `POST /v1/schedule` |

> Não existe `GET /v2/shipping/...` nem `DELETE /v2/shipping/...`. Use as rotas `/v1`.

## Endpoint: criar remessa

- **Método e rota:** `POST /v2/shipping`
- **Quando usar:** após obter uma cotação **v2** elegível, dentro do prazo de validade.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `quotation` | string | **sim** | - | - | - | UUID da cotação. Precisa ter sido criada na **v2** |
| `carrier_code` | integer | **sim** se `slug` não for enviado | - | - | - | Código do contrato (`carriers[].code` da cotação) |
| `slug` | string | **sim** se `carrier_code` não for enviado | - | 20 | - | Identificador textual do contrato (`carriers[].slug` da cotação) |
| `address_sender` | object | **sim** | - | - | - | Remetente (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | Destinatário (ver tabela abaixo) |
| `payment` | object | **sim** | - | - | - | Forma de pagamento |
| `type` | string | não | - | 30 | `Commercial` | Finalidade do envio. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | não | 44 | 44 | `null` | Chave da NF-e. **Exatamente 44 dígitos** |
| `invoice_number` | string | não | 1 | 50 | código interno | Número da fatura/invoice |
| `shipment_reference` | string | não | 1 | 50 | ID interno | Referência do envio |
| `cost_center_code` | integer | não | - | - | `null` | Código do centro de custo. Precisa existir e estar liberado para o seu usuário |
| `payment_notify_shipping_billing` | boolean | não | - | - | `false` | Notificar a área de faturamento sobre a cobrança |
| `attachments` | array | não | - | - | `null` | Documentos anexados ao envio |
| `biller` | object | não | - | - | preset da conta | Faturante exibido na invoice em vez do remetente |

### Parâmetros — `address_sender` e `address_receiver`

**Esta é a principal diferença da v2:** cidade, estado e CEP são obrigatórios aqui.

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código do endereço cadastrado. Quando usado, não envie os demais campos |

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `city` | string | **sim** | 3 | 100 | - | Cidade |
| `postal_code` | string | condicional | - | 12 | - | CEP/ZIP. Obrigatório se `state` não for informado. **Sempre obrigatório para endereços no Brasil** |
| `state` | string | condicional | - | - | - | Nome do estado/província. Obrigatório se `postal_code` não for informado |
| `state_code` | string | condicional | - | - | - | Sigla do estado/província. Obrigatório se `state` e `postal_code` não forem informados |
| `street` | string | **sim** | - | 70 | - | Logradouro |
| `name` | string | **sim** | 5 | 35 | - | Nome do contato/razão social |
| `email` | string | **sim** | - | 60 | - | E-mail válido (recebe notificações do transportador) |
| `phone` | string | **sim** | - | 25 | - | Telefone |
| `type` | string | **sim** | - | - | `pj` | `pf` (pessoa física) ou `pj` (pessoa jurídica) |
| `number` | string | não | - | 6 | `null` | Número do imóvel |
| `district` | string | não | - | 30 | `null` | Bairro |
| `complement` | string | não | - | 20 | `null` | Complemento |
| `description` | string | não | - | 155 | `null` | Rótulo interno do endereço |
| `phone_extension` | string | não | - | 5 | `null` | Ramal |
| `federal_tax_id` | string | condicional | - | 20 | `null` | CPF/CNPJ ou documento federal. **Obrigatório quando o país é `BR` ou `AR`** |
| `state_tax_id` | string | condicional | - | 20 | `null` | Inscrição estadual. **Obrigatório quando o país é `BR`** |
| `foreign` | boolean | condicional | - | - | `false` | Indica se a pessoa é estrangeira. **Obrigatório quando o país é `BR` e `address_code` não é usado** |

> **Regra prática:** informe `city` + (`postal_code` **ou** `state_code`) + `street` + `name` + `email` +
> `phone` + `type`. O país vem da cotação.

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|
| `payment.type` | string | **sim** | `billed`, `gateway`, `afterward` | Modelo financeiro do envio |

| Valor | Significado |
|---|---|
| `billed` | Faturado: o valor entra na sua fatura periódica |
| `gateway` | Pagamento imediato via gateway, na moeda definida em `currency_payment` |
| `afterward` | Pagamento posterior, acertado fora do fluxo automático |

### Parâmetros — `attachments[]`

| Campo | Tipo | Obrigatório | Máx. | Descrição |
|---|---|---|---|---|
| `description` | string | **sim** | 100 | Nome/descrição do documento (usado depois para localizar ou remover o anexo) |
| `base64` | string | **sim** | - | Conteúdo do arquivo codificado em Base64 |

| Limite | Valor |
|---|---|
| Formatos aceitos | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamanho por arquivo | **5 MB** |

### Parâmetros — `biller` (faturante)

Quando informado, este endereço aparece na invoice em vez do endereço do remetente.
Envie `address_code` **ou** os campos manuais. **Se não for usar faturante, não envie o campo.**

| Campo | Tipo | Mín. | Máx. | Descrição |
|---|---|---|---|---|
| `address_code` | integer | - | - | Código de endereço cadastrado |
| `country_code` | string | 2 | 2 | País em ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nome do estado |
| `state_code` | string | - | 10 | Sigla do estado |
| `postal_code` | string | - | 12 | CEP/ZIP |
| `city` | string | 3 | 100 | Cidade |
| `street` | string | - | 70 | Logradouro |
| `number` | string | - | 6 | Número |
| `district` | string | - | 30 | Bairro |
| `complement` | string | - | 20 | Complemento |
| `description` | string | - | 155 | Rótulo interno |
| `name` | string | 5 | 35 | Nome/razão social |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Telefone |
| `phone_extension` | string | - | 5 | Ramal |
| `type` | string | - | - | `pf` ou `pj` |
| `federal_tax_id` | string | - | 20 | Documento federal |
| `state_tax_id` | string | - | 20 | Inscrição estadual |
| `foreign` | boolean | - | - | Indica se é estrangeiro |

### Payload mínimo

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "state_code": "FL",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

### Payload máximo

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "slug": "dhl-br",
  "type": "Commercial",
  "nfkey": "12345678901234567890123456789012345678901234",
  "invoice_number": "NF-000123",
  "shipment_reference": "REF-2026-0001",
  "cost_center_code": 1,
  "payment_notify_shipping_billing": true,
  "address_sender": {
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
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Matriz",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Sala 1",
    "name": "Holding Exemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcKJcTl8uXrp..."
    },
    {
      "description": "XML da Nota Fiscal",
      "base64": "PD94bWwgdmVyc2lvbj0i..."
    }
  ]
}
```

### Exemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "state_code": "FL", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

## Diferenças em relação à v1

| Tema | v1 | v2 |
|---|---|---|
| Endereço na remessa | Logradouro, nome, e‑mail, telefone, tipo (+ docs fiscais) | Os mesmos campos **+ `city`, `state`/`state_code`, `postal_code`** |
| `city` | Não aceito (vem da cotação) | **Obrigatório**, mín. 3 e máx. 100 caracteres |
| `biller` | Campos de localidade aceitos | Campos de localidade **validados** (`city` mín. 3) |
| Endpoints do ciclo | Criação, status, documentos, atualização, anexos, cancelamento | **Somente criação.** O restante usa `/v1` |
| Origem da cotação | `POST /v1/quotation` | `POST /v2/quotation` |

## Regras de negócio

### Elegibilidade da cotação
| Condição | Mensagem quando falha |
|---|---|
| Cotação existe e pertence à conta | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Cotação não expirada | `Esta cotação está expirada...` |
| Cotação não usada antes | `Cotação já foi utilizada em outra remessa!` |
| Criada na **v2** | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Não é `simple` de `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| Contrato está na lista da cotação | `Contrato especificado não foi listado para esta cotação!` |

### Coerência entre cotação e remessa
O país informado na remessa precisa ser o mesmo da cotação:
- `O país :country é diferente do enviado na cotação!`
- `O Estado :state é diferente do enviado na cotação!`
- `A cidade :city é diferente do enviado na cotação!`

### Documentos fiscais obrigatórios
| País do endereço | `federal_tax_id` | `state_tax_id` | `foreign` |
|---|---|---|---|
| `BR` | Obrigatório (CPF ou CNPJ válido) | Obrigatório | Obrigatório sem `address_code` |
| `AR` | Obrigatório | - | - |
| Outros | Opcional | Opcional | Opcional |

- Para `type: pf`, a inscrição estadual não pode ser `ISENTO`/`ISENTA`.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio (cotação, endereço, pagamento, documento fiscal) | Ler `message` e corrigir o cenário |
| `401` | Pagamento não autorizado | Habilitar o tipo de pagamento na conta |
| `403` | Sem permissão para a ação | Solicitar a permissão ao suporte |
| `404` | `Contrato especificado não foi listado para esta cotação!` | Conferir `carrier_code`/`slug` |
| `422` | Validação de campos ou cotação de versão incompatível | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Use a v2 quando o endereço completo só existir no momento do envio.
- Todo o ciclo operacional pós-criação é feito pelos endpoints `/v1` — eles não distinguem a versão
  em que a remessa foi criada.
- Para o detalhamento de cada operação pós-criação, veja [Remessa (v1)](api-remessa-v1.md),
  [Rastreio](api-rastreio.md), [Documentos](api-documentos.md) e [Agendamento](api-agendamento.md).
