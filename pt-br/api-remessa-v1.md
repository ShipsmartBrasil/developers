# Remessa (v1)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md) e [Cotação (v1)](api-cotacao-v1.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Transforma uma **cotação v1** em um envio real: gera AWB, invoice, declaração e os demais documentos.
- É o módulo mais completo do ciclo: criação, consulta de status, consulta de documentos, atualização,
  remoção de anexos e cancelamento.
- Os endpoints de status, documentos, atualização e cancelamento atendem **também** remessas criadas na
  [Remessa V2](api-remessa-v2.md).
- A criação é **assíncrona**: a API registra a remessa e o despacho no transportador acontece em seguida.

> ⚠️ **A remessa é uma operação irreversível na rota normal.** Após criada, ela entra em processamento
> no transportador. As únicas alterações posteriores admitidas são excepcionais: vincular a NF-e quando ela
> não foi informada, anexar documentos que não foram enviados na criação e informar/trocar o centro de custo.

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
1. POST /v1/quotation                  → quotation + carriers[]
2. POST /v1/shipping                   → remessa recebida (assíncrona)
3. GET  /v1/shipping/{shipping}/status → acompanhar até "shipped"
4. GET  /v1/shipping/{shipping}/docs   → baixar AWB, invoice etc.
5. POST /v1/schedule                   → agendar coleta (opcional)
```

1. [Crie a cotação v1](api-cotacao-v1.md) e verifique se ela é [elegível](#cotações-inelegíveis-para-remessa).
2. Envie a remessa com o `quotation` e o transportador escolhido (`carrier_code` **ou** `slug`).
3. A criação é **assíncrona**: a resposta `200` significa "recebemos", não "despachamos".
4. Consulte o [status](api-rastreio.md) até que ela apareça em `shipped`.
5. Baixe os [documentos](api-documentos.md).
6. Se necessário, [agende a coleta](api-agendamento.md).

## Conceitos essenciais

### Remessa a partir de uma cotação v1
- A cotação v1 já trouxe **país, cidade, estado e CEP** de origem e destino. Por isso, na remessa v1 você
  informa apenas **quem** envia e **quem** recebe (nome, contato, logradouro, documentos fiscais).
- Campos de localidade (`city`, `state`, `state_code`, `postal_code`, `country_code`) enviados em
  `address_sender`/`address_receiver` são **ignorados** na v1: a localidade é sempre a da cotação.
- Uma cotação gera **uma única** remessa.

### O identificador `{shipping}`

Nos endpoints de documentos, atualização, remoção de anexos e cancelamento, o parâmetro de path `{shipping}`
aceita **três formatos**:

| Formato | Exemplo | Observação |
|---|---|---|
| UUID da cotação | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forma recomendada. Qualquer valor com hífen é tratado como UUID |
| Chave da NF-e | `12345678901234567890123456789012345678901234` | Exatamente 44 dígitos numéricos |
| Número do AWB | `271249639538` | Remessas criadas nos **últimos 90 dias** |

> O endpoint de **status** aceita apenas UUID ou AWB. Detalhes em [Rastreio de Remessa](api-rastreio.md).

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
| `slug` | string | **sim** se `carrier_code` não for enviado | - | 20 | - | Identificador textual do contrato (`carriers[].slug` da cotação). Quando enviado, prevalece sobre `carrier_code` |
| `address_sender` | object | **sim** | - | - | - | Remetente (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | Destinatário (ver tabela abaixo) |
| `payment` | object | **sim** | - | - | - | Forma de pagamento (ver tabela abaixo) |
| `type` | string | não | - | - | preset da conta ou `Commercial` | Finalidade do envio. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | condicional | 44 | 44 | `null` | Chave da NF-e. Quando informada, deve ter **exatamente 44 caracteres**. Obrigatória em alguns contratos (ver [Regras de negócio](#restrições-de-transportador)) |
| `invoice_number` | string | não | 1 | 50 | código interno | Número da fatura/invoice. Se omitido, usa o código interno da remessa |
| `shipment_reference` | string | não | 1 | 50 | ID interno | Referência do envio. Se omitido, usa o ID interno da remessa |
| `cost_center_code` | integer | não | - | - | preset da conta | Código do centro de custo. Precisa existir e estar liberado para o seu usuário. Vazio/nulo recebe o centro de custo padrão da conta |
| `payment_notify_shipping_billing` | boolean | não | - | - | preset da conta | Com `payment.type: afterward`, envia o e-mail de cobrança da remessa |
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
| `address_code` | integer | **sim** nesta forma | Código do endereço cadastrado. **Precisa ter o mesmo país, estado e cidade da cotação** |

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `street` | string | **sim** | - | 70 | - | Logradouro |
| `name` | string | **sim** | 5 | 35 | - | Nome do contato/razão social |
| `email` | string | **sim** | - | 60 | - | E-mail válido (recebe notificações do transportador) |
| `phone` | string | **sim** | - | 25 | - | Telefone |
| `type` | string | **sim** | - | - | - | `pf` (pessoa física) ou `pj` (pessoa jurídica) |
| `number` | string | não | - | 6 | `null` | Número do imóvel |
| `district` | string | não | - | 30 | `null` | Bairro |
| `complement` | string | não | - | 20 | `null` | Complemento |
| `description` | string | não | - | 155 | `null` | Rótulo interno do endereço |
| `phone_extension` | string | não | - | 5 | `null` | Ramal |
| `federal_tax_id` | string | condicional | - | 20 | `null` | CPF/CNPJ ou documento federal. Obrigatório conforme o país (ver [Documentos fiscais](#documentos-fiscais-obrigatórios)) |
| `state_tax_id` | string | condicional | - | 20 | `null` | Inscrição estadual. Obrigatória para brasileiros (`BR` com `foreign: false`) |
| `foreign` | boolean | não | - | - | `false` se o país é `BR`; `true` nos demais | Indica se a pessoa é estrangeira. Define como o `federal_tax_id` é validado no Brasil |

> O país, o estado, a cidade e o CEP **não** são informados aqui: eles vêm da cotação.
> Se você precisa informá-los na remessa, use a [Remessa V2](api-remessa-v2.md).

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|
| `payment.type` | string | **sim** | `billed`, `gateway`, `afterward` | Modelo financeiro do envio |

| Valor | Significado | Requisito da conta |
|---|---|---|
| `billed` | Faturado: o valor entra na sua fatura periódica | Opção de pagamento "faturado" habilitada |
| `gateway` | Pagamento imediato via gateway, na moeda definida em `currency_payment` | Opção de pagamento "imediato" habilitada |
| `afterward` | Pagamento posterior, acertado fora do fluxo automático | - |

> Valor fora da lista retorna `422`. Tipo não habilitado para a sua conta retorna
> `401` com `Opção de pagamento não autorizado!`.

### Parâmetros — `attachments[]`

| Campo | Tipo | Obrigatório | Máx. | Descrição |
|---|---|---|---|---|
| `description` | string | **sim** | 100 | Nome/descrição do documento (usado depois para localizar ou remover o anexo) |
| `base64` | string | **sim** | - | Conteúdo do arquivo codificado em Base64 |

| Limite | Valor |
|---|---|
| Formatos aceitos | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamanho total da requisição | **90 MB** |

> ⚠️ **Na criação, anexos inválidos são descartados em silêncio:** arquivo em formato não reconhecido ou maior
> que o limite não gera erro — a remessa é criada sem ele. Confira os anexos gravados em
> [Documentos](api-documentos.md). Para anexar depois, com validação e mensagens de erro, use
> [atualizar remessa](#endpoint-atualizar-remessa) (limite de **5 MB** por arquivo).

### Parâmetros — `biller` (faturante)

Quando informado, este endereço aparece na invoice em vez do endereço do remetente.
Envie `address_code` **ou** os campos manuais. **Se não for usar faturante, não envie o campo** (ou envie-o vazio:
nesse caso vale o faturante pré-configurado na conta, se houver).

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
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

> `federal_tax_id` (CNPJ válido) e `state_tax_id` aparecem no payload mínimo porque o remetente é brasileiro
> pessoa jurídica. Para remetente de outro país (exceto `MX` e `GR`), esses campos podem ser omitidos.

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
    "phone": "13055550100",
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
    "type": "afterward"
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
    "address_sender": {"street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` quando a remessa foi registrada |
| `message` | string | Confirmação de recebimento |

> A resposta **não** traz AWB nem documentos: a remessa foi apenas recebida. Use o
> [rastreio](api-rastreio.md) para acompanhar e os [documentos](api-documentos.md) para baixar os arquivos.
> O identificador da remessa é o próprio `quotation`.

### Erros específicos da criação

| Status | Mensagem | Causa / correção |
|---|---|---|
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Documento federal ausente onde é obrigatório (ver [Documentos fiscais](#documentos-fiscais-obrigatórios)) |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brasileiro sem `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ com dígito verificador incorreto |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF com dígito verificador incorreto |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` no Brasil com `state_tax_id` isento |
| `400` | `Numero da nota fiscal é requerido` | O contrato exige `nfkey` para este cenário |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` de contrato removido/inativo |
| `401` | `Opção de pagamento não autorizado!` | Tipo de pagamento não habilitado na sua conta |
| `403` | `Cotação Simples é inelegível para remessa!` | Use `type: advanced` ou `items` na cotação |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` ultrapassado — refaça a cotação |
| `403` | `Has no permission!` | Seu perfil não tem permissão de criar remessa |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | `quotation` de outra conta |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fora da lista da cotação |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` inválido ou de outra conta |
| `406` | `Cotação já foi utilizada em outra remessa!` | Uma cotação gera **uma** remessa |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` com país diferente do cotado |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` com estado diferente do cotado |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` com cidade diferente da cotada |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inexistente ou criado na v2 (vem em `data.quotation`) |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente)...` | Conta travada em DDP com cotação DDU não paga |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` não liberado para você |

> Quando há mais de um erro de documento fiscal, as mensagens vêm juntas em `message`, separadas por ` | `.

## Endpoint: consultar status da remessa

- **Método e rota:** `GET /v1/shipping/{shipping}/status`
- **Por canal:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `shipping` | path | string | **sim** | UUID da cotação ou AWB |
| `channel_identifier_number` | path | string | **sim** (variante por canal) | Identificador comercial informado na cotação |
| `channel_code` | path | integer | não | Código do canal de venda |

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

> Formato da resposta, grupos de status e códigos em [Rastreio de Remessa](api-rastreio.md).

## Endpoint: consultar documentos

- **Método e rota:** `GET /v1/shipping/{shipping}/docs`
- **Por canal:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | UUID, chave NF-e ou AWB |
| `attachments` | query | boolean | não | `true` | Incluir os anexos enviados por você (exige `base64=true`) |
| `embedded` | query | boolean | não | `true` | Incluir documentos incorporados (requer permissão específica e `base64=true`) |
| `base64` | query | boolean | não | `true` | Incluir o conteúdo em Base64 além da `url` |
| `documents` | query | string | não | todos | Lista separada por vírgula com os tipos desejados |

Tipos aceitos em `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

### Exemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

> Formato da resposta e detalhes de cada documento em [Documentos de Remessa](api-documentos.md).

## Endpoint: atualizar remessa

- **Método e rota:** `PUT /v1/shipping/{shipping}`
- **Quando usar:** apenas em casos excepcionais — vincular a NF-e, anexar documentos faltantes
  ou informar/trocar/remover o centro de custo.
- **Regra:** envie `nfkey` e/ou `attachments` e/ou `cost_center_code`. **Pelo menos um** precisa estar presente.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | - | UUID, chave NF-e ou AWB |
| `nfkey` | body | string | condicional | 44 | 44 | Chave da NF-e. **Só é gravada se a remessa ainda não tiver NF-e**; caso já tenha, é ignorada |
| `cost_center_code` | body | integer | condicional | - | - | Código do centro de custo liberado para o seu usuário. Enviar `null` **remove** o centro de custo |
| `attachments` | body | array | condicional | - | - | Arquivos a anexar |
| `attachments[].description` | body | string | **sim** se `attachments` for enviado | - | 100 | Descrição do documento |
| `attachments[].base64` | body | string | **sim** se `attachments` for enviado | - | - | Conteúdo em Base64. Formatos: `PDF`, `XML`, `JPEG`, `PNG`. Máx. **5 MB** por arquivo |

### Payload mínimo

```json
{
  "cost_center_code": 1
}
```

### Payload máximo

```json
{
  "nfkey": "12345678901234567890123456789012345678901234",
  "cost_center_code": 1,
  "attachments": [
    { "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
    { "description": "Foto do produto", "base64": "iVBORw0KGgoAAAANSUhEUg..." }
  ]
}
```

### Exemplo de request

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` quando a atualização foi aplicada |
| `message` | string | Confirmação |

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | Nenhum dos três campos foi enviado |
| `400` | `Status da remessa não permite atualização na NFE.` | A remessa já passou de `shipped` (ou está cancelada/com erro) |
| `400` | `Erro ao salvar arquivo!` | `base64` inválido |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Anexo acima de 5 MB |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` inválido ou de outra conta |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Remessa cancelada ou com ciclo encerrado |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Formato do anexo não aceito |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML malformado |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Centro de custo não liberado |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Erro inesperado |

> As alterações são aplicadas na ordem NF-e → centro de custo → anexos. Se um anexo falhar, a NF-e e o centro
> de custo enviados na mesma chamada **já foram gravados**.

## Endpoint: remover anexos

- **Método e rota:** `DELETE /v1/shipping/{shipping}/docs`
- **Quando usar:** para apagar um anexo enviado por engano.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | - | UUID, chave NF-e ou AWB |
| `attachments` | body | array | **sim** | 1 item | - | Lista dos anexos a remover |
| `attachments[].id` | body | integer | **sim** se `name` não for enviado | 1 | - | ID do anexo (retornado em `GET .../docs`) |
| `attachments[].name` | body | string | **sim** se `id` não for enviado | - | 100 | Descrição exata do anexo |

> Envie **`id` OU `name`** por item, nunca os dois.

### Payload mínimo

```json
{
  "attachments": [
    { "id": 123456 }
  ]
}
```

### Payload máximo

```json
{
  "attachments": [
    { "id": 123456 },
    { "name": "Nota Fiscal" }
  ]
}
```

### Exemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

### Exemplo de response

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

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.deleted[]` | array | Anexos removidos (`id`, `name`) |
| `data.failed[]` | array | Anexos não removidos, com o motivo em `reason` |

> A operação é **parcial**: o que foi removido aparece em `deleted` e o que falhou em `failed`,
> com o motivo. Verifique sempre o array `failed`.

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Remessa não encontrada!` | UUID sem remessa |
| `404` | `Esta remessa não foi encontrada!` | AWB/NF-e não localizado ou de outra conta |
| `422` | Validação | `attachments` vazio, ou item sem `id`/`name` |

## Endpoint: cancelar remessa

- **Método e rota:** `DELETE /v1/shipping/{shipping}`
- **Por canal:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Quando usar:** para solicitar o cancelamento. A resposta confirma o pedido; a conclusão pode ser assíncrona.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | - | UUID, chave NF-e, AWB **ou** `channel_identifier_number` |
| `channel_identifier_number` | path | string | **sim** (variante por canal) | - | - | Identificador comercial |
| `channel_code` | path | integer | não | - | - | Código do canal. Só valores **numéricos** são interpretados como canal |
| `reason` | body | string | não | 10 | 255 | Motivo do cancelamento, em texto livre |

> **Como a rota é interpretada:**
> - `DELETE /v1/shipping/{valor}/{numero}` é sempre **cancelamento por canal** (`channel_identifier_number` + `channel_code`).
> - `DELETE /v1/shipping/{valor}` procura primeiro uma remessa cujo `channel_identifier_number` seja `{valor}`;
>   se não encontrar, trata `{valor}` como UUID, chave NF-e ou AWB.

### Payload mínimo

Sem corpo:

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN'
```

### Payload máximo

```json
{
  "reason": "Cliente solicitou o cancelamento do pedido"
}
```

### Exemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/PED-2026-0001/1 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"reason":"Cliente solicitou o cancelamento do pedido"}'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Solicitação de cancelamento da remessa 9c06fd75-d5f4-40eb-a1a8-f12086f49944 recebida. Em breve será processada; consulte o status para acompanhar."
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` quando o pedido de cancelamento foi aceito |
| `message` | string | Confirmação, com o UUID da remessa |

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Esta remessa não foi encontrada!` | Identificador inválido ou remessa de outra conta |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | O status atual não permite cancelamento (ver tabela abaixo) |
| `422` | Validação | `reason` com menos de 10 ou mais de 255 caracteres; `channel_code` inexistente |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Erro inesperado |

**Quando o cancelamento é aceito:**

| Status atual da remessa | Resultado |
|---|---|
| `received`, `processing_payment`, `paid_out` | Cancelada na hora (`cancelled`) |
| `shipped` | Vai para `cancelling` e é cancelada no transportador; alguns contratos cancelam direto (`cancelled`) |
| `error` sem envio registrado no transportador | Cancelada na hora (`cancelled`) |
| `processing_shipment` | Aceito apenas em contratos específicos; nos demais, recusado |
| Demais (`collecting`, `in_transit`, `delivered`, `cancelled`...) | Recusado com `Remessa não atende requisitos para cancelamento :id !` |

> Após solicitar o cancelamento, consulte o [status](api-rastreio.md): ele passa por `cancelling`
> e, quando confirmado pelo transportador, vai para `cancelled`.

## Cotações inelegíveis para remessa

Antes de registrar a remessa, a API confere a cotação nesta ordem. A primeira condição que falhar interrompe
a criação:

| # | Verificação | Status | Mensagem |
|---|---|---|---|
| 1 | `quotation` existe e foi criado na **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | A cotação pertence à sua conta | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Conta travada em DDP: cotação DDU precisa estar paga | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | Documentos fiscais de remetente e destinatário | `400` | Ver [Documentos fiscais obrigatórios](#documentos-fiscais-obrigatórios) |
| 5 | Cotação ainda não usada | `406` | `Cotação já foi utilizada em outra remessa!` |
| 6 | Não é `simple` de `package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 7 | Cotação dentro da validade | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 8 | Contrato está na lista da cotação | `404` | `Contrato especificado não foi listado para esta cotação!` |
| 9 | Tipo de pagamento habilitado para a conta | `401` | `Opção de pagamento não autorizado!` |
| 10 | NF-e informada quando o contrato exige | `400` | `Numero da nota fiscal é requerido` |

> Para saber, **antes** de remeter, se uma cotação já foi usada, consulte
> `GET /v1/quotation/{quotation}` e veja `data.finished`. Detalhes dos cenários elegíveis em
> [Cotação (v1)](api-cotacao-v1.md#cotações-inelegíveis-para-remessa).

## Regras de negócio

### Coerência entre cotação e remessa
- Com endereço manual (Forma B), a localidade é sempre a da cotação.
- Com `address_code` (Forma A), o endereço cadastrado precisa ter **o mesmo país, estado e cidade** da cotação.
  Divergências retornam `406`:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documentos fiscais obrigatórios
| País do endereço | `type` | `foreign` | `federal_tax_id` | `state_tax_id` |
|---|---|---|---|---|
| `BR` | `pf` | `false` (padrão) | **Obrigatório** — CPF válido | **Obrigatório** |
| `BR` | `pj` | `false` (padrão) | **Obrigatório** — CNPJ válido | **Obrigatório** |
| `BR` | qualquer | `true` | **Obrigatório** (documento estrangeiro, sem validação de CPF/CNPJ) | Opcional |
| `MX`, `GR` | qualquer | - | **Obrigatório** | Opcional |
| Outros | qualquer | - | Opcional | Opcional |

- Para `type: pf` no Brasil, a inscrição estadual não pode ser `ISENTO`/`ISENTA`.
- Para `type: pj` em qualquer país, um `federal_tax_id` no formato de CNPJ é validado como CNPJ.
- Com `address_code`, os documentos fiscais são lidos do cadastro do endereço e passam pelas mesmas regras.

### Restrições de transportador
- Para contratos DHL com remetente **pessoa jurídica no Brasil** e objeto diferente de documento, a `nfkey` é
  obrigatória. A ausência retorna `400` com `Numero da nota fiscal é requerido`.

### Predefinições aplicadas na remessa
| Predefinição | Efeito |
|---|---|
| Centro de custo | Preenche `cost_center_code` quando ele não é enviado, ou é enviado vazio/nulo |
| Finalidade do envio | Preenche `type` quando ele não é enviado |
| Faturante | Preenche `biller` quando ele não é enviado (ou é enviado vazio) e a conta exibe faturante |
| Notificação de cobrança | Preenche `payment_notify_shipping_billing` quando ele não é enviado |

### Cancelamento automático
Remessas que ficam **29 dias** sem número de rastreio, em status anterior ao despacho, podem ser canceladas
automaticamente pela plataforma. Detalhes em [Rastreio de Remessa](api-rastreio.md).

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Documento fiscal, NF-e obrigatória, contrato por `slug` inválido, cancelamento não permitido | Ler `message` e corrigir o cenário |
| `401` | Pagamento não autorizado | Habilitar o tipo de pagamento na conta |
| `403` | Cotação simples/expirada, ou sem permissão para a ação | Refazer a cotação / solicitar a permissão ao suporte |
| `404` | Cotação de outra conta, remessa/contrato/endereço não encontrado | Conferir o identificador e o `carrier_code`/`slug` |
| `406` | Cotação já usada ou `address_code` incompatível com a cotação | Fazer nova cotação / usar o endereço cotado |
| `422` | Validação de campos, cotação de outra versão, conta só DDP, anexo inválido | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Guarde sempre o `quotation`: ele é o identificador mais estável do envio.
- Informar `channel_identifier_number` na cotação é a forma mais prática de consultar e cancelar
  usando o número do seu próprio pedido.
- Para informar cidade, estado e CEP **na remessa** (e não na cotação), use a
  [Remessa V2](api-remessa-v2.md). Diferenças em [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Para cotação + remessa + documentos em **uma única chamada síncrona**, veja [Manifesto](api-manifesto.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
