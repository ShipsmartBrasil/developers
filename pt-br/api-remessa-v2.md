# Remessa (v2)

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md) e [Cotação (v2)](api-cotacao-v2.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Transforma uma **cotação v2** em um envio real.
- A diferença central em relação à [Remessa v1](api-remessa-v1.md): aqui você informa o **endereço completo**
  (cidade, estado e CEP), porque a cotação v2 recebeu apenas o país (e talvez o estado).
- O endereço completo é **validado na remessa** (CEP no Brasil, endereço internacional nos demais países).
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
1. POST /v2/quotation                  → quotation + carriers[]   (só país/estado)
2. POST /v2/shipping                   → remessa recebida          (endereço completo)
3. GET  /v1/shipping/{shipping}/status → acompanhar até "shipped"
4. GET  /v1/shipping/{shipping}/docs   → baixar AWB, invoice etc.
5. POST /v1/schedule                   → agendar coleta (opcional)
```

1. [Crie a cotação v2](api-cotacao-v2.md) e verifique se ela é [elegível](#cotações-inelegíveis-para-remessa).
2. Envie a remessa com o `quotation`, o transportador escolhido e o endereço completo.
3. A criação é **assíncrona**: `200` significa "recebemos", não "despachamos".
4. Acompanhe pelo [rastreio](api-rastreio.md) e baixe os [documentos](api-documentos.md).

## Conceitos essenciais

### Remessa a partir de uma cotação v2
- O **país** de origem e destino vem sempre da cotação (não é informado na remessa).
- O **estado** informado na cotação (`state_code`) é reaproveitado; se a cotação não teve estado, ele é
  descoberto a partir do CEP/endereço enviado na remessa.
- **Cidade e CEP** são informados aqui e passam a valer para a remessa.
- Uma cotação gera **uma única** remessa.

### Mapa de endpoints por versão

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

> Não existe `GET /v2/shipping/...` nem `DELETE /v2/shipping/...`. Use as rotas `/v1`, documentadas em
> [Remessa (v1)](api-remessa-v1.md).

## Endpoint: criar remessa

- **Método e rota:** `POST /v2/shipping`
- **Quando usar:** após obter uma cotação **v2** elegível, dentro do prazo de validade.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `quotation` | string | **sim** | - | - | - | UUID da cotação. Precisa ter sido criada na **v2** |
| `carrier_code` | integer | **sim** se `slug` não for enviado | - | - | - | Código do contrato (`carriers[].code` da cotação) |
| `slug` | string | **sim** se `carrier_code` não for enviado | - | 20 | - | Identificador textual do contrato (`carriers[].slug` da cotação). Quando enviado, prevalece sobre `carrier_code` |
| `address_sender` | object | **sim** | - | - | - | Remetente (ver tabela abaixo) |
| `address_receiver` | object | **sim** | - | - | - | Destinatário (ver tabela abaixo) |
| `payment` | object | **sim** | - | - | - | Forma de pagamento |
| `type` | string | não | - | - | preset da conta ou `Commercial` | Finalidade do envio. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | condicional | 44 | 44 | `null` | Chave da NF-e. **Exatamente 44 caracteres**. Obrigatória em alguns contratos |
| `invoice_number` | string | não | 1 | 50 | código interno | Número da fatura/invoice |
| `shipment_reference` | string | não | 1 | 50 | ID interno | Referência do envio |
| `cost_center_code` | integer | não | - | - | preset da conta | Código do centro de custo liberado para o seu usuário. Vazio/nulo recebe o padrão da conta |
| `payment_notify_shipping_billing` | boolean | não | - | - | preset da conta | Com `payment.type: afterward`, envia o e-mail de cobrança |
| `attachments` | array | não | - | - | `null` | Documentos anexados ao envio |
| `biller` | object | não | - | - | preset da conta | Faturante exibido na invoice em vez do remetente |

### Parâmetros — `address_sender` e `address_receiver`

**Esta é a principal diferença da v2:** cidade e CEP (ou estado) são obrigatórios aqui.

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código do endereço cadastrado. Quando usado, não envie os demais campos |

> ⚠️ Na v2, use `address_code` na remessa **somente se o mesmo `address_code` foi usado na cotação v2**.
> Uma cotação feita só com país não tem cidade, e o endereço cadastrado será recusado com
> `406 A cidade :city é diferente do enviado na cotação!`.

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `city` | string | **sim** | 3 | 100 | - | Cidade |
| `postal_code` | string | condicional | - | 12 | - | CEP/ZIP. Obrigatório se `state` não for informado. **Recomendado sempre; no Brasil é o que permite validar o endereço** |
| `state` | string | condicional | - | - | - | Nome do estado/província. Obrigatório se `postal_code` não for informado |
| `state_code` | string | não | - | - | estado da cotação | Sigla do estado/província. Quando a cotação teve `state_code`, ele é usado no lugar deste |
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
| `federal_tax_id` | string | condicional | - | 20 | `null` | CPF/CNPJ ou documento federal. Obrigatório conforme o país (ver [Documentos fiscais](api-remessa-v1.md#documentos-fiscais-obrigatórios)) |
| `state_tax_id` | string | condicional | - | 20 | `null` | Inscrição estadual. Obrigatória para brasileiros (`BR` com `foreign: false`) |
| `foreign` | boolean | não | - | - | `false` se o país é `BR`; `true` nos demais | Indica se a pessoa é estrangeira |

> **Regra prática:** informe `city` + `postal_code` + `street` + `name` + `email` + `phone` + `type`
> (+ documentos fiscais quando exigidos). O país vem da cotação.
> Sem `postal_code`, o campo `state` (nome do estado) passa a ser obrigatório — `state_code` sozinho não basta.

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|
| `payment.type` | string | **sim** | `billed`, `gateway`, `afterward` | Modelo financeiro do envio |

| Valor | Significado | Requisito da conta |
|---|---|---|
| `billed` | Faturado: o valor entra na sua fatura periódica | Opção de pagamento "faturado" habilitada |
| `gateway` | Pagamento imediato via gateway, na moeda definida em `currency_payment` | Opção de pagamento "imediato" habilitada |
| `afterward` | Pagamento posterior, acertado fora do fluxo automático | - |

### Parâmetros — `attachments[]`

| Campo | Tipo | Obrigatório | Máx. | Descrição |
|---|---|---|---|---|
| `description` | string | **sim** | 100 | Nome/descrição do documento (usado depois para localizar ou remover o anexo) |
| `base64` | string | **sim** | - | Conteúdo do arquivo codificado em Base64 |

| Limite | Valor |
|---|---|
| Formatos aceitos | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamanho total da requisição | **90 MB** |

> Como na v1, anexos em formato não reconhecido são **descartados em silêncio** na criação. Para anexar
> com validação, use `PUT /v1/shipping/{shipping}` (limite de 5 MB por arquivo).

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
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
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
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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
| `message` | string | Confirmação de recebimento. O identificador da remessa é o próprio `quotation` |

## Diferenças em relação à v1

| Tema | v1 | v2 |
|---|---|---|
| Endereço na remessa | Logradouro, nome, e‑mail, telefone, tipo (+ docs fiscais) | Os mesmos campos **+ `city`, `postal_code` (ou `state`)** |
| `city` | Ignorado (vem da cotação) | **Obrigatório**, mín. 3 e máx. 100 caracteres |
| Validação de endereço | Feita na cotação | **Feita na remessa** |
| `address_code` | Precisa bater com o endereço da cotação | Só funciona se a cotação v2 também usou `address_code` |
| `biller` | Campos de localidade aceitos | Campos de localidade **validados** (`city` mín. 3) |
| Endpoints do ciclo | Criação, status, documentos, atualização, anexos, cancelamento | **Somente criação.** O restante usa `/v1` |
| Origem da cotação | `POST /v1/quotation` | `POST /v2/quotation` |

## Cotações inelegíveis para remessa

Valem as mesmas verificações da v1, na mesma ordem — veja a tabela completa em
[Remessa (v1)](api-remessa-v1.md#cotações-inelegíveis-para-remessa). A única diferença é a versão:

| Verificação | Status | Mensagem |
|---|---|---|
| `quotation` existe e foi criado na **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Cotação simples de pacote/pallet | `403` | `Cotação Simples é inelegível para remessa!` |
| Cotação já usada | `406` | `Cotação já foi utilizada em outra remessa!` |
| Cotação expirada | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| Contrato fora da lista da cotação | `404` | `Contrato especificado não foi listado para esta cotação!` |

## Regras de negócio

### Validação do endereço completo
Depois das verificações da cotação, o endereço manual (Forma B) é validado:

| Caso | Status | Mensagem |
|---|---|---|
| Brasil, sem estado na cotação: CEP inexistente ou de outro estado | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Fora do Brasil: endereço não localizado | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Fora do Brasil: estado/província não reconhecido | `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |

> Envie sempre `postal_code` e, quando souber, `state_code` (de preferência já na cotação): é o que dá a maior
> taxa de acerto na validação.

### Coerência entre cotação e remessa
- O país é sempre o da cotação.
- O `state_code` da cotação, quando existe, prevalece sobre o enviado na remessa.
- Com `address_code`, o endereço cadastrado precisa ter o mesmo país, estado e cidade registrados na cotação:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documentos fiscais, transportador e predefinições
Idênticos à v1:
[Documentos fiscais obrigatórios](api-remessa-v1.md#documentos-fiscais-obrigatórios),
[Restrições de transportador](api-remessa-v1.md#restrições-de-transportador) e
[Predefinições aplicadas na remessa](api-remessa-v1.md#predefinições-aplicadas-na-remessa).

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Documento fiscal, CEP brasileiro inválido, NF-e obrigatória | Ler `message` e corrigir o cenário |
| `401` | Pagamento não autorizado | Habilitar o tipo de pagamento na conta |
| `403` | Cotação simples/expirada, ou sem permissão | Refazer a cotação / solicitar a permissão ao suporte |
| `404` | Contrato fora da cotação, endereço internacional não localizado | Conferir `carrier_code`/`slug` e o endereço |
| `406` | Cotação já usada ou `address_code` incompatível | Fazer nova cotação / usar endereço manual |
| `422` | Validação de campos, cotação de outra versão, conta só DDP | Corrigir os campos listados em `data` |
| `500` | Erro inesperado ou estado/província não reconhecido | Revisar o endereço; repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Use a v2 quando o endereço completo só existir no momento do envio.
- Todo o ciclo operacional pós-criação é feito pelos endpoints `/v1` — eles não distinguem a versão
  em que a remessa foi criada.
- Para o detalhamento de cada operação pós-criação, veja [Remessa (v1)](api-remessa-v1.md),
  [Rastreio](api-rastreio.md), [Documentos](api-documentos.md) e [Agendamento](api-agendamento.md).
- Diferenças entre as versões em [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
