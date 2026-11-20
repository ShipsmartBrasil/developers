# Error Codes

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Gathers, in a single place, the error and warning messages returned by the Public API, grouped by module.
- For each message: the HTTP code, the literal text, the cause and how to fix it.
- The messages are in the API's default language (`pt`). With `lang: en` or `lang: es`, the text comes translated,
  but the HTTP code and the `status` are the same.
- Placeholders such as `:name`, `:sku` and `:address` are replaced by the actual value in the response.
- The details of each rule are on the module page, linked in each section.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json
lang: pt
```

## Usage flow

How to handle a response that is not `200`:

```
HTTP 422 ──► read data (field → messages) ──► fix the payload  ──► resend
HTTP 4xx ──► read message                 ──► fix the scenario ──► resend
HTTP 500 ──► wait (exponential backoff)   ──► retry            ──► persisted? contact support
```

1. Check the **HTTP code** first: it defines the family of the problem.
2. In `422`, the top-level `message` is generic — the actual errors are in `data`.
3. In `400`, `401`, `403`, `404` and `406`, the `message` describes the business rule that blocked the operation.
4. Only retry `500` errors automatically. `4xx` errors repeat as long as the payload is not fixed.

## Key concepts

### Error envelope

```json
{
  "status": "warning",
  "message": "Cotação já foi utilizada em outra remessa!"
}
```

| Field | Description |
|---|---|
| `status` | `warning` for business rule/validation; `error` for unexpected failure or carrier refusal |
| `message` | Error text, translated according to the `lang` header |
| `data` | Present in `422` (field map) and in some carrier errors (raw return) |

### Validation error (`422`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "items.0.hscode": ["O campo items.0.hscode deve conter no mínimo 6 caracteres."],
    "address_sender.city": ["O campo address sender.city é obrigatório quando address sender.address code não está presente."]
  }
}
```

> The `422` `message` is always the same generic message. **Use `data`** to show the error to the user.

### Most frequent validation messages in `data`

| Message (default) | Meaning |
|---|---|
| `O campo :attribute é obrigatório.` | Required field missing |
| `O campo :attribute é obrigatório quando :other não está presente.` | Conditional requirement (e.g.: manual address without `address_code`) |
| `O campo :attribute não contém um valor válido.` | Value outside the accepted list (enum) |
| `O valor selecionado para o campo :attribute é inválido.` | Nonexistent registration code (`channel_code`, `box_code`, `address_code`, `quotation`...) |
| `O campo :attribute deve conter no mínimo :min caracteres.` / `não pode conter mais de :max caracteres.` | Text length outside the limit |
| `O campo :attribute deve conter um número inteiro.` / `deve conter um valor numérico.` | Incorrect numeric type |
| `O campo :attribute não contém um endereço de email válido.` | Invalid e-mail |
| `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` not authorized |

## Errors by HTTP status

| Status | Typical `status` | Family | What to do |
|---|---|---|---|
| `308` | - | Call without version prefix | Use `/v1/...` or `/v2/...` |
| `400` | `warning` / `error` | Business rule | Read `message` and fix the scenario |
| `401` | `warning` | Payment type not enabled | Enable the payment option in the account |
| `403` | `warning` | No permission, blocked account, ineligible quotation | Request permission / redo the quotation |
| `404` | `warning` / `error` | Resource not found | Check identifiers and registrations |
| `406` | `warning` | Business conflict | Read `message` and adjust the flow |
| `422` | `warning` | Field validation (or rule about data) | Fix the fields listed in `data` |
| `500` | `error` | Unexpected failure | Retry with backoff; if it persists, contact support |

## Authentication and permission errors

They apply to **all** endpoints.

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | `Authorization` header missing or invalid token | Send `Authorization: Bearer YOUR_TOKEN` with a valid token |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Blocked account | Contact support |
| `403` | `Has no permission!` | The profile does not have the permission for the action (quote, ship, query documents etc.) | Request the permission from support |
| `400` | Variable message (`status: "error"`) | Unexpected failure when validating the token | Retry; if it persists, contact support |

Details in [Getting Started](api-primeiros-passos.md#authentication).

## Quotation errors

`POST /v1/quotation`, `POST /v2/quotation` and `GET /v1/quotation/{quotation}/{level}`.
Details in [Quotation (v1)](api-cotacao-v1.md#business-rules).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Box without item in `advanced` or `items` (v2) | Associate at least one item (`box_id`) with each box |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` outside the range of boxes | Use the position (1, 2, 3...) of the box in `boxes[]` |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Manual item without `box_id` | Provide `box_id` or `sku` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | Weight above the maximum of the registered box | Reduce the weight or change the box |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Sum of the items above the maximum of the registered box | Redistribute the items |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` nonexistent or from another account | Check the [box registration](api-caixa-v1.md) |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` not registered and incomplete item | Register the item or send the manual data |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | `unit_value` equal to or less than zero | Provide a positive value |
| `400` | `O kit :sku é inválido!` | Nonexistent/inactive kit | Review the kit in the dashboard |
| `400` | `A caixa do kit :sku é inválida!` | Nonexistent kit box | Review the kit in the dashboard |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Kit item removed | Review the kit in the dashboard |
| `400` | `Para usar SmartBox, informe ao menos uma caixa em boxes[].` | `type: items` (v2) with `smartbox` grouping without `boxes[]` | Send the candidate boxes |
| `400` | `Nenhuma caixa comporta o item :name (Smart Box).` | None of the sent boxes fits the item | Send a larger box |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Identical origin and destination | Review the addresses |
| `400` | `Nenhum endereco encontrado para o usuário especificado!` | `address_code` nonexistent or from another account | Check the address code |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Invalid Brazilian CEP (v1) | Fix the CEP or send `state_code` |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | No carrier returned a result | Review the scenario or talk to the sales team |
| `400` | `Não foi encontrada a cotação especificada!` | `GET` of a nonexistent quotation or one from another account | Check the `quotation` |
| `400` | `Nivel de detalhe da cotação não existe` | Invalid `level` in the `GET` | Use `simple` or `detailed` |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` from another account | Use a channel of your account |
| `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` | Brazilian address without CEP (v1) | Provide `postal_code` |
| `404` | `Endereço internacional enviado está errado ou não existe!` | International address not found (v1) | Review city/postal code/state |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | State/province not recognized | Send a valid `state_code` |
| `404` | `O item com o sku :name não existe!` | `type: items` (v2) with unregistered SKU | Register the item |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | `default` grouping: box linked to the item does not exist | Review the item → box link |
| `404` | `Nenhum transportador encontrado para este usuário!` | Account without a contract for the route/object | Talk to the sales team |
| `404` | `Não há contratos que atendem as especificações do pacote enviado. Se você possui duvidas sobre restrições de envios (como dimensões e pesos), por favor contate o suporte para atendimento!` | No contract covers the weight/dimension/route | Review the cargo or talk to support |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado. Fale com nossa equipe pelo chat para recebermos seu pedido especial.` | Cargo outside the automatic range | Talk to the sales team |
| `406` | `Codigo de endereço do remetente não encontrado!` | Account type requires a registered sender | Send `address_sender.address_code` |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | HS Code without a tax table at the destination | Review the `hscode` |
| `400` | `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³` | Pallet without measurements and without `volume` | Send the measurements or `volume` |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Unexpected failure | Retry with backoff |

> Alerts (non-blocking): `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` and
> `Valores declarados dos itens podem ocasionar problemas aduaneiros`, in `data.warnings`.

## Shipment errors

`POST /v1/shipping` and `POST /v2/shipping`. Details in [Shipment (v1)](api-remessa-v1.md#quotations-ineligible-for-shipment)
and [Shipment (v2)](api-remessa-v2.md#business-rules).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | Nonexistent `quotation` or one from another version (in `data.quotation`) | Use the shipment of the same version as the quotation |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | Quotation from another account | Use the token of the owning account |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` | Account locked to DDP with an unpaid DDU quotation | Redo the quotation with `tax: sender` |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Federal document missing | Provide `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brazilian without state registration | Provide `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | Invalid CNPJ | Fix the CNPJ |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | Invalid CPF | Fix the CPF |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | Individual with exempt state registration | Provide a valid state registration |
| `406` | `Cotação já foi utilizada em outra remessa!` | Quotation reused | Make a new quotation |
| `403` | `Cotação Simples é inelegível para remessa!` | `simple` of `package`/`pallet` | Quote with `advanced` (or `items`, in v2) |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` exceeded | Make a new quotation |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` outside `carriers[]` | Use a contract from the quotation |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` of an inactive contract | Use another contract |
| `401` | `Opção de pagamento não autorizado!` | Payment type not enabled | Enable it in the account or use another type |
| `400` | `Numero da nota fiscal é requerido` | Contract requires `nfkey` | Provide the NF-e key |
| `404` | `O endereço especificado, não foi encontrado!` | Invalid `address_code` | Check the code |
| `406` | `O país :country é diferente do enviado na cotação!` | Incompatible `address_code` | Use the quoted address |
| `406` | `O Estado :state é diferente do enviado na cotação!` | Incompatible `address_code` | Use the quoted address |
| `406` | `A cidade :city é diferente do enviado na cotação!` | Incompatible `address_code` (in v2, quotation made with country only) | Use the quoted address / manual address |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Invalid CEP (v2) | Fix the CEP |
| `404` | `Endereço internacional enviado está errado ou não existe!` | International address not found (v2) | Review the address |
| `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | State/province not recognized (v2) | Send a valid `state_code` |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Unexpected failure | Retry with backoff |

## Update, attachment and cancellation errors

`PUT /v1/shipping/{shipping}`, `DELETE /v1/shipping/{shipping}/docs` and `DELETE /v1/shipping/...`.
Details in [Shipment (v1)](api-remessa-v1.md#endpoint-update-shipment).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | `PUT` without `nfkey`, `attachments` or `cost_center_code` | Send at least one field |
| `400` | `Status da remessa não permite atualização na NFE.` | Shipment beyond `shipped` | It is not possible to link the NF-e |
| `400` | `Erro ao salvar arquivo!` | Invalid `base64` | Re-encode the file |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Attachment above 5 MB | Reduce the file |
| `404` | `Esta remessa não foi encontrada!` | Invalid `{shipping}` or one from another account (`PUT`); AWB/NF-e not found | Check the identifier |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Canceled/closed shipment | It is not possible to attach |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Format not accepted | Convert the file |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | Malformed XML | Fix the XML |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Unexpected failure in the `PUT` | Retry with backoff |
| `400` | `Remessa não encontrada!` | Attachment removal with a UUID without a shipment | Check the UUID |
| `400` | `Esta remessa não foi encontrada!` | Cancellation with an invalid identifier or one from another account | Check the identifier |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | Current status does not allow cancellation | See the cancellation table in [Shipment (v1)](api-remessa-v1.md#endpoint-cancel-shipment) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Unexpected failure in the cancellation | Retry with backoff |

> In attachment removal, individual failures come in `data.failed[]` with `reason` (e.g.: `Nenhum documento encontrado!`),
> even with HTTP `200`.

## Tracking and document errors

`GET /v1/shipping/.../status` and `GET /v1/shipping/.../docs`.
Details in [Shipment Tracking](api-rastreio.md#common-errors) and [Shipment Documents](api-documentos.md#common-errors).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `400` | `remessa.status.status_not_found: ` | Status: no shipment for the provided UUID/AWB (untranslated text) | Use the quotation UUID or the correct AWB |
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Status: shipment from another account or channel without shipment | Check the identifier/channel |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Shipment without a payment record yet | Wait; if it persists, contact support |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Unexpected failure in the status | Retry with backoff |
| `400` | `Remessa não encontrada!` | Documents: UUID without a shipment | Check the UUID |
| `404` | `Esta remessa não foi encontrada!` | Documents: AWB/NF-e not found | Check it or use the UUID |
| `400` | `Nenhum documento encontrado com o código especificado!` | Documents not generated yet, filter without result or channel without shipment | Wait for `shipped` / review `documents` |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Unexpected failure in the documents | Retry with backoff |

## Pickup scheduling errors

`POST /v1/schedule`, `GET /v1/schedule/{shipping}` and `DELETE /v1/schedule/{shipping}/{reason}`.
Details in [Pickup Scheduling](api-agendamento.md#common-errors).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Today's or a past date | Use a date from tomorrow onward |
| `400` | `Coleta não pode ser realizada em um final de semana` | Saturday or Sunday | Choose a business day |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | National holiday | Choose another business day |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Date too far away | Reduce the advance notice |
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | A schedule already exists for the shipment | Query it with `GET` |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Shipment not in `shipped` | Wait for `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Carrier without integrated pickup | Take the package to a drop-off point |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Incomplete pickup address | Complete the address |
| `400` | `O atributo federal_tax_id não é válido` | Invalid CPF/CNPJ | Fix the document |
| `400` | `Endereço enviado não pertence ao pais da remessa` | State from another country | Use an address in the sender's country |
| `400` | `Agendamento não foi encontrado para esta remessa` | AWB not found (`POST`/`DELETE`) | Check the AWB or use the UUID |
| `404` | `Agendamento não foi encontrado para esta remessa` | `GET` without a schedule | Check the identifier |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Cancellation outside the allowed status | Cancel only confirmed schedules |
| `422` | `O campo reason não contém um valor válido.` (in `data`) | `reason` outside `001` to `008` | Use a valid reason |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Unexpected failure (includes a missing key in `address`) | Send all the `address` keys; retry |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Unexpected failure in the cancellation | Retry; contact support |

## Manifest errors

`POST /v1/manifest`. Details in [Manifest](api-manifesto.md#common-errors).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `400` | `Tipo de pagamento não autorizado!` | Account without billed payment | Enable billing |
| `400` | `O canal especificado não foi encontrado!` | `channel_code` from another account | Use a channel of the account |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | Inactive `slug` | Use another contract |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | State not resolved | Send `state_code` |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | `address_code` not visible to the account | Check the code |
| `404` | `A transportadora informada não está disponível para esta remessa!` | `carrier_code` without manifest | Omit or change the `carrier_code` |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Contract without synchronous dispatch | Contact support |
| `400` | Carrier return (`status: "error"`, with `data`) | The carrier refused the shipment | Fix it according to the return |
| `400` | `MANIFEST WARNING - AWB cannot be generated, carrier not mapped` | Label not generated after acceptance; the shipment exists | Fetch it in Documents; do **not** retry |

> Errors of the quotation step within the manifest (boxes, items, kits, contracts) come back as `400 warning`,
> with the same messages as the [quotation](#quotation-errors).

## Draft errors

`POST /v1/draft`, `PUT /v1/draft/{code}`, `GET /v1/draft` and `DELETE /v1/draft`.
Details in [Draft](api-rascunho.md#common-errors).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | `PUT` on a draft not in `incomplete` | Check the status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Code nonexistent, from another account or already converted into a shipment | Check the code |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Unexpected failure | Retry with backoff |

> Errors from the asynchronous processing of the draft do not come back in the HTTP response: they appear in
> the draft's `status_descricao` (e.g.: `Nenhum item foi enviado para este rascunho`).

## Box and item errors

`/v1/box` and `/v1/item`. Details in [Box](api-caixa-v1.md#common-errors) and [Items](api-itens-v1.md#common-errors).

| Status | Message | Cause | How to fix it |
|---|---|---|---|
| `422` | Validation (in `data`) | Invalid box fields; invalid listing filters | Fix the listed fields |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Unexpected failure | Retry with backoff |
| `200` + report | `Box Error: box :id not found!` / `Box Error: box :id cannot be deleted, has related item` | Box removal refused during processing | Use an integer `box_code` / unlink items |
| `200` + report | `sku => :sku, status => invalid, reason => :campo` | Item discarded in the asynchronous processing | Fix the indicated field |

> Box and item registrations are **asynchronous**: the `200` response confirms receipt, and invalid
> records are discarded afterwards, with a report sent to the account's e-mail.

## Business rules

### Retry policy
| Situation | Retry automatically? |
|---|---|
| `500` | Yes, with exponential backoff (e.g.: 2 s, 4 s, 8 s) and a limit of attempts |
| `4xx` | No. Fix the payload/scenario first |
| Manifest with `400` after the carrier's acceptance | **No.** The shipment already exists: check [Tracking](api-rastreio.md) and [Documents](api-documentos.md) |
| Shipment (`POST /v*/shipping`) with network timeout | Before retrying, check the status by the `quotation`: a quotation only generates one shipment |

### Refusals with `status: "error"`
Some business refusals related to **weight** or to the **carrier** come with `status: "error"` and code
`400`/`404`. Always handle them by the HTTP code and the `message`, not only by the `status`.

## Final notes
- General response conventions in [Getting Started](api-primeiros-passos.md#default-response-format).
- Quotations that cannot become a shipment: [Quotation (v1)](api-cotacao-v1.md#quotations-ineligible-for-shipment).
- If in doubt, provide support with the `quotation`, the time of the call and the `message` received.
