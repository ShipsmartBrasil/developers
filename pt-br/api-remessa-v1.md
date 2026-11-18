# Remessa (v1)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md) e [Cotação (v1)](api-cotacao-v1.md)

## Visão geral
- Transforma uma cotação v1 em um envio real: gera AWB, invoice, declaração e os demais documentos.
- É o módulo mais completo do ciclo: criação, consulta de status, consulta de documentos, atualização,
  remoção de anexos e cancelamento.
- Os endpoints de status, documentos e cancelamento atendem **também** remessas criadas na
  [Remessa V2](api-remessa-v2.md).

> ⚠️ **A remessa é uma operação irreversível na rota normal.** Após criada, ela entra em processamento
> no transportador. As únicas alterações posteriores admitidas são excepcionais: vincular a NF-e quando ela
> não foi informada, anexar documentos que não foram enviados na criação e informar o centro de custo.

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
1. POST /v1/quotation                 → quotation + carriers[]
2. POST /v1/shipping                  → remessa recebida (assíncrona)
3. GET  /v1/shipping/{shipping}/status→ acompanhar até "shipped"
4. GET  /v1/shipping/{shipping}/docs  → baixar AWB, invoice etc.
5. POST /v1/schedule                  → agendar coleta (opcional)
```

1. [Crie a cotação](api-cotacao-v1.md) e verifique se ela é **elegível** para remessa.
2. Envie a remessa com o `quotation` e o transportador escolhido (`carrier_code` **ou** `slug`).
3. A criação é **assíncrona**: a resposta `200` significa "recebemos", não "despachamos".
4. Consulte o [status](api-rastreio.md) até que ela apareche em `shipped`.
5. Baixe os [documentos](api-documentos.md).
6. Se necessário, [agende a coleta](api-agendamento.md).

## O identificador `{shipping}`

Em todos os endpoints de consulta/alteração, o parâmetro de path `{shipping}` aceita **três formatos**:

| Formato | Exemplo | Observação |
|---|---|---|
| UUID da cotação | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forma recomendada |
| Número do AWB | `271249639538` | Funciona para remessas criadas nos **últimos 90 dias** |
| Chave da NF-e | `12345678901234567890123456789012345678901234` | Exatamente 44 dígitos numéricos |

Para consultar sem guardar nenhum desses valores, use os endpoints **por canal**, baseados no
`channel_identifier_number` que você informou na cotação.

## Endpoint: criar remessa

- **Método e rota:** `POST /v1/shipping`
- **Quando usar:** após obter uma cotação v1 elegível, dentro do prazo de validade.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `quotation` | string | **sim** | - | - | - | UUID da cotação. Precisa ter sido criada na **v1** |
| `carrier_code` | integer | **sim** se `slug` não for enviado | - | - | - | Código do contrato (`carriers[].code` da cotação) |
| `slug` | string | **sim** se `carrier_code` não for enviado | - | 20 | - | Identificador textual do contrato (`carriers[].slug` da cotação) |
| `address_sender` | object | **sim** | - | - | - | Remetente (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | Destinatário (ver tabela abaixo) |
| `payment` | object | **sim** | - | - | - | Forma de pagamento (ver tabela abaixo) |
| `type` | string | não | - | 30 | `Commercial` | Finalidade do envio. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | não | 44 | 44 | `null` | Chave da NF-e. Quando informada, deve ter **exatamente 44 dígitos** |
| `invoice_number` | string | não | 1 | 50 | código interno | Número da fatura/invoice. Se omitido, usa o código interno da remessa |
| `shipment_reference` | string | não | 1 | 50 | ID interno | Referência do envio. Se omitido, usa o ID interno da remessa |
| `cost_center_code` | integer | não | - | - | `null` | Código do centro de custo. Precisa existir e estar liberado para o seu usuário |
| `payment_notify_shipping_billing` | boolean | não | - | - | `false` | Notificar a área de faturamento sobre a cobrança da remessa |
| `attachments` | array | não | - | - | `null` | Documentos anexados ao envio (ver tabela abaixo) |
| `biller` | object | não | - | - | preset da conta | Faturante exibido na invoice em vez do remetente (ver tabela abaixo) |

> **`carrier_code` ou `slug`:** envie **um** dos dois. O contrato informado precisa estar na lista
> `carriers[]` retornada pela cotação; caso contrário a API responde `404` com
> `Contrato especificado não foi listado para esta cotação!`.

### Parâmetros — `address_sender` e `address_receiver`

Duas formas, não misture:

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código do endereço cadastrado. **Deve ser o mesmo endereço usado na cotação** |

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
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
| `foreign` | boolean | condicional | - | - | `false` | Indica se a pessoa é estrangeira. **Obrigatório quando o país é `BR` e `address_code` não é usado** — é o que permite validar corretamente o `federal_tax_id` |

> O país, o estado, a cidade e o CEP **não** são informados aqui: eles vêm da cotação.
> Se você precisa informá-los na remessa, use a [Remessa V2](api-remessa-v2.md).

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|
| `payment.type` | string | **sim** | `billed`, `gateway`, `afterward` | Modelo financeiro do envio |

| Valor | Significado |
|---|---|
| `billed` | Faturado: o valor entra na sua fatura periódica |
| `gateway` | Pagamento imediato via gateway, na moeda definida em `currency_payment` |
| `afterward` | Pagamento posterior, acertado fora do fluxo automático |

> O tipo precisa estar **habilitado para a sua conta**. Caso contrário:
> `400 Tipo de pagamento inexistente!` ou `401 Opção de pagamento não autorizado!`.

### Parâmetros — `attachments[]`

| Campo | Tipo | Obrigatório | Máx. | Descrição |
|---|---|---|---|---|
| `description` | string | **sim** | 100 | Nome/descrição do documento (usado depois para localizar ou remover o anexo) |
| `base64` | string | **sim** | - | Conteúdo do arquivo codificado em Base64 |

| Limite | Valor |
|---|---|
| Formatos aceitos | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamanho por arquivo | **5 MB** |
| Tamanho total da requisição | **90 MB** |

> Arquivo fora do formato: `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.`
> Arquivo acima do limite: `O arquivo enviado excede o limite máximo de 5MB.`
> XML corrompido: `O arquivo XML enviado é inválido ou está corrompido.`

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
| `city` | string | - | 100 | Cidade |
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

> `federal_tax_id`, `state_tax_id` e `foreign` aparecem no payload mínimo porque o remetente é brasileiro.
> Para remetente de outro país (exceto `AR`), esses três campos podem ser omitidos.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

> A resposta **não** traz AWB nem documentos: a remessa foi apenas recebida. Use o
> [rastreio](api-rastreio.md) para acompanhar e os [documentos](api-documentos.md) para baixar os arquivos.

### Erros específicos da criação

| Status | Mensagem | Causa / correção |
|---|---|---|
| `400` | `Tipo de pagamento inexistente!` | `payment.type` fora de `billed`/`gateway`/`afterward` ou não habilitado |
| `400` | `Esta cotação está expirada. Porfavor, faça outra cotação...` | `valid_until` ultrapassado — refaça a cotação |
| `400` | `Cotação já foi utilizada em outra remessa!` | Uma cotação gera **uma** remessa |
| `400` | `Cotação Simples é inelegível para remessa!` | Use `type: advanced` ou `items` na cotação |
| `400` | `Chave da Nota não foi informada!!` | O contrato exige `nfkey` para este cenário |
| `400` | `O endereço especificado, não foi encontrado!` | `address_code` inválido ou de outra conta |
| `400` | `O país :country é diferente do enviado na cotação!` | O endereço da remessa precisa bater com o da cotação |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | País `BR`/`AR` sem `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | País `BR` sem `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ com dígito verificador incorreto |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF com dígito verificador incorreto |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` com `state_tax_id` isento |
| `401` | `Opção de pagamento não autorizado!` | Habilite o tipo de pagamento na sua conta |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fora da lista da cotação |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | Cotação v2 sendo usada em `/v1/shipping` |

## Endpoint: consultar status da remessa

- **Método e rota:** `GET /v1/shipping/{shipping}/status`
- **Por canal:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `shipping` | path | string | **sim** | UUID da cotação, AWB (últimos 90 dias) ou chave NF-e (44 dígitos) |
| `channel_identifier_number` | path | string | **sim** | Identificador comercial informado na cotação |
| `channel_code` | path | integer | não | Código do canal de venda |

Detalhes, formatos de resposta e códigos de status em [Rastreio de Remessa](api-rastreio.md).

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: consultar documentos

- **Método e rota:** `GET /v1/shipping/{shipping}/docs`
- **Por canal:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | UUID, AWB ou chave NF-e |
| `attachments` | query | boolean | não | `true` | Incluir os anexos enviados por você |
| `embedded` | query | boolean | não | `true` | Incluir documentos incorporados (requer permissão específica) |
| `base64` | query | boolean | não | `true` | Incluir o conteúdo em Base64 além da `url` |
| `documents` | query | string | não | todos | Lista separada por vírgula com os tipos desejados |

Tipos aceitos em `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

Detalhes completos em [Documentos de Remessa](api-documentos.md).

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: atualizar remessa

- **Método e rota:** `PUT /v1/shipping/{shipping}`
- **Quando usar:** apenas em casos excepcionais — vincular a NF-e, anexar documentos faltantes
  ou informar o centro de custo.
- **Regra:** envie `nfkey` e/ou `attachments` e/ou `cost_center_code`. **Pelo menos um** precisa estar presente.

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `nfkey` | string | condicional | 44 | 44 | Chave da NF-e (44 dígitos numéricos) |
| `cost_center_code` | integer | condicional | - | - | Código do centro de custo liberado para o seu usuário |
| `attachments` | array | condicional | - | - | Arquivos a anexar |
| `attachments[].description` | string | **sim** se `attachments` for enviado | - | 100 | Descrição do documento |
| `attachments[].base64` | string | **sim** se `attachments` for enviado | - | - | Conteúdo em Base64. Formatos: `PDF`, `XML`, `JPEG`, `PNG`. Máx. **5 MB** por arquivo |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | Nenhum dos três campos foi enviado |
| `404` | `Remessa não encontrada!` | `{shipping}` inválido |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | A remessa já avançou demais no fluxo |

## Endpoint: remover anexos

- **Método e rota:** `DELETE /v1/shipping/{shipping}/docs`
- **Quando usar:** para apagar um anexo enviado por engano.

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `attachments` | array | **sim** | 1 item | - | Lista dos anexos a remover |
| `attachments[].id` | integer | **sim** se `name` não for enviado | 1 | - | ID do anexo (retornado em `GET .../docs`) |
| `attachments[].name` | string | **sim** se `id` não for enviado | - | 100 | Descrição exata do anexo |

> Envie **`id` OU `name`** por item, nunca os dois.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!",
  "data": {
    "deleted": [
      { "id": 123456, "name": "Nota Fiscal" }
    ],
    "failed": [
      { "id": 789, "name": null, "reason": "Nenhum documento encontrado!" }
    ]
  }
}
```

> A operação é **parcial**: o que foi removido aparece em `deleted` e o que falhou em `failed`,
> com o motivo. Verifique sempre o array `failed`.

## Endpoint: cancelar remessa

- **Método e rota:** `DELETE /v1/shipping/{shipping}`
- **Por canal:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Quando usar:** para solicitar o cancelamento. O processamento é **assíncrono**.

| Campo | Local | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | - | UUID, AWB ou chave NF-e |
| `channel_identifier_number` | path | string | **sim** (variante por canal) | - | - | Identificador comercial |
| `channel_code` | path | integer | não | - | - | Código do canal. Só valores **numéricos** são interpretados como canal |
| `reason` | body | string | não | 10 | 255 | Motivo do cancelamento, em texto livre |

> **Atenção à ambiguidade de rota:** `DELETE /v1/shipping/{valor}/{numero}` é interpretado como
> **cancelamento por canal**. Para cancelar por identificador da remessa use sempre
> `DELETE /v1/shipping/{shipping}` (um único segmento).

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Cliente solicitou o cancelamento do pedido"}'
```

```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Remessa especificada não foi encontrada ou não é possivel mais cancelar a mesma` | Identificador inválido ou remessa em status que não permite cancelamento |
| `422` | Validação | `reason` com menos de 10 ou mais de 255 caracteres |

> Após solicitar o cancelamento, consulte o [status](api-rastreio.md): ele passa por `cancelling`
> e, quando confirmado pelo transportador, vai para `cancelled`.

## Regras de negócio

### Elegibilidade da cotação
| Condição | Mensagem quando falha |
|---|---|
| Cotação existe e pertence à conta | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Cotação não expirada | `Esta cotação está expirada...` |
| Cotação não usada antes | `Cotação já foi utilizada em outra remessa!` |
| Mesma versão de API | `Não é possivel finalizar, pois seu quotation pode estar inválido...` |
| Não é `simple` de `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| Contrato está na lista da cotação | `Contrato especificado não foi listado para esta cotação!` |

### Coerência entre cotação e remessa
O endereço informado na remessa precisa ser compatível com o da cotação. Divergências retornam:
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
- O `foreign` existe para que a validação do `federal_tax_id` seja feita no formato correto
  (documento brasileiro vs. documento estrangeiro).

### Restrições de transportador
- `Cotações e envios via FedEx entre Europa e Oriente Médio estão temporariamente suspensos pela transportadora.`
- Para alguns contratos (ex.: DHL com remetente pessoa jurídica no Brasil), a `nfkey` é obrigatória.
  A ausência retorna `Chave da Nota não foi informada!!`.

### Cancelamento automático
Remessas que ficam **29 dias ou mais** sem evento de rastreio em status elegível podem ser canceladas
automaticamente pela plataforma.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio (cotação, endereço, pagamento, documento fiscal) | Ler `message` e corrigir o cenário |
| `401` | Pagamento não autorizado | Habilitar o tipo de pagamento na conta |
| `403` | Sem permissão para a ação | Solicitar a permissão ao suporte |
| `404` | Remessa/contrato não encontrado | Conferir o identificador e o `carrier_code`/`slug` |
| `422` | Validação de campos ou remessa em status não alterável | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Guarde sempre o `quotation`: ele é o identificador mais estável do envio.
- Informar `channel_identifier_number` na cotação é a forma mais prática de consultar e cancelar
  usando o número do seu próprio pedido.
- Para informar cidade, estado e CEP **na remessa** (e não na cotação), use a
  [Remessa V2](api-remessa-v2.md).
- Para cotação + remessa + documentos em **uma única chamada síncrona**, veja [Manifesto](api-manifesto.md).
