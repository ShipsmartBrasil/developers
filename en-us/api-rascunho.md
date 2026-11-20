# Draft

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Lets you register an **incomplete** shipment: you send what you already have, and the platform validates it later.
- On input, **all fields are optional** (only the format of each field is checked). The strict
  validation happens later, at **dispatch**, which runs automatically in cycles of a few minutes.
- When the draft passes dispatch, the platform generates the **quotation + shipment** by itself and the draft
  **ceases to exist** (it is removed). While data is missing, it stays `incomplete` and can be corrected.
- It is the recommended flow when the integration receives the order data in stages (e.g.: the order arrives before
  the final address or the invoice).
- The **account presets** are applied to the draft, including the party responsible for
  the tax, which **replaces** the sent value when it is configured.

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

The `lang` header accepts `pt`, `en` and `es` (default `pt`). Each operation requires its own permission
in the profile (create, edit, list and remove drafts). Without it, the response is `403 Has no permission!`.

## Usage flow
1. `POST /v1/draft` with the data you already have. The response only confirms receipt
   (**it does not return the draft code**).
2. Wait a few minutes: the draft is stored and then goes through the automatic dispatch.
3. `GET /v1/draft` to locate the draft (by your `channel_identifier_number`), get the `codigo`
   and see the `status`. If it is `incomplete`, read the `status_descricao`.
4. `PUT /v1/draft/{code}` sending **the complete and corrected draft** (the `PUT` replaces the entire content).
5. Repeat 2 to 4 until the draft disappears from the listing: this means the quotation and the shipment were created.
6. Follow the shipment through the [tracking](api-rastreio.md) and download the [documents](api-documentos.md),
   using the `channel_identifier_number` (and the `channel_code`, if any).
7. `DELETE /v1/draft` to discard drafts that will not be used (or that ended up with `error` status).

## Key concepts

### Draft vs. other flows

| Topic | Quotation + Shipment | Manifest | Draft |
|---|---|---|---|
| Payload | Complete in 2 steps | Complete in 1 call | **Partial, in as many steps as needed** |
| Validation | Synchronous, per step | Synchronous, full | **Format on input; complete rules at dispatch (asynchronous)** |
| Response | Quotation with prices | Documents ready | Receipt confirmation (without code) |
| Payment | `billed`, `gateway`, `afterward` | `billed` | **`billed` or `afterward`** |
| Contract choice | You choose | You choose | **Automatic** (preferred `carrier_code` or account preference) |
| Later correction | New quotation | No | **Yes, via `PUT` (only in `incomplete`)** |

### Draft lifecycle

```
POST /v1/draft                      PUT /v1/draft/{code}
     │                                       │
     ▼                                       ▼
  received ──► validating ──► incomplete ──► edited ──► validating ──► ...
                    │
                    ▼
                 complete ──► processing ──► quotation + shipment created (draft removed)
                                  │
                                  ├──► incomplete  (business rule refused: reason in status_descricao)
                                  └──► error       (unexpected failure: reason in status_descricao)
```

| Status | Meaning | Can it be edited (`PUT`)? |
|---|---|---|
| `received` | Draft received, waiting for the next dispatch cycle | No |
| `edited` | Draft updated via `PUT`, waiting for the next dispatch cycle | No |
| `validating` | Under validation | No |
| `incomplete` | **Data is missing or some rule refused the shipment.** The reason is in `status_descricao` | **Yes** |
| `complete` | Passed validation, ready to generate quotation and shipment | No |
| `processing` | Generating quotation and shipment | No |
| `error` | Unexpected failure in processing | No (remove it and create it again) |

| Detail | Behavior |
|---|---|
| Dispatch frequency | `received` and `edited` drafts are validated automatically every ~3 minutes |
| Success | The quotation and the shipment are created and **the draft is removed** (it no longer appears in the `GET`) |
| Draft stuck in `processing` | After ~15 minutes without completion, it goes back to `incomplete` with an empty `status_descricao` |
| Size of `status_descricao` | Limited to **254 characters**: the list of pending issues may come truncated |
| Format of `status_descricao` | In field validation, it brings the first message of each field, concatenated without a separator |

> **Only drafts with `incomplete` status can be edited.** In any other status the response is
> `400 Apenas rascunhos com o status incomplete podem ser editados`.

### The `PUT` replaces the entire draft

The `PUT /v1/draft/{code}` does **not** merge with the previous content: the saved content is **replaced** by the
sent payload (already merged with the account presets). Fields that you do not resend **are lost**.

- Always send the complete draft, with the corrections.
- Be careful when building the `PUT` from the `GET`: the listing **does not return** `measurement`,
  `residential_delivery`, `non_stackable`, `cost_center_code`, `declared_freight`, `biller`,
  `manufacturer` and `address_sender.address_code`. Resend them from your own data.

### Account presets in the draft

The presets configured in the dashboard are merged into the payload in the `POST` **and** in the `PUT`
(see [Account presets](api-primeiros-passos.md#account-presets)).

| Field | Preset used when | Note |
|---|---|---|
| `channel_code` | The field is **not** sent | Sending the field (even `null`) prevents the preset |
| `cost_center_code` | The field is not sent **or** comes empty/`null` | Only field in which an empty value also receives the preset |
| `carrier_code` | The field is not sent | - |
| `tax` | **Whenever there is a tax preset** | The preset **replaces** the sent value (see below) |
| `insurance` | The field is not sent | - |
| `measurement` | The field is not sent or comes `null` | Without a preset, it assumes `metric` |
| `currency_quote`, `currency_payment` | The field is not sent | - |
| `residential_delivery`, `non_stackable` | The field is not sent | - |
| `type` (shipment purpose) | The field is not sent | - |
| `address_sender` | The `address_sender` object is **not** sent | The preset comes in as `address_code`. Sending `address_sender` with any field discards the preset |
| `biller` | The object is not sent **or** comes empty, and the account is configured to display the biller | If you send a filled `biller`, it always applies |
| `declared_freight` | The object is not sent | Declared freight passed on to the quotation |

> The merge is done **at the first level**: a sent object (`address_sender`, `biller`,
> `declared_freight`) replaces the preset object entirely, without combining field by field.

### Party responsible for the tax (`tax`)

When the account has a tax preset, **the value sent in `tax` is ignored** and the platform resolves
the responsible party as follows:

| Tax preset configuration | Value stored in the draft |
|---|---|
| No tax preset | The sent `tax`. If not sent, the dispatch reports `tax` as required |
| Default `sender` | `sender`, even if you send `receiver` |
| Default `receiver` | `receiver`, even if you send `sender` |
| Default defined and destination country in the exception list | The **opposite** of the default (`sender` ↔ `receiver`) |
| Locked mode | Always `sender` |

- The destination country considered is the `address_receiver.country_code` **of the same payload**. If it is not
  sent, the per-country exception is not evaluated and the default applies.
- Since the `PUT` reapplies the presets, the tax is recalculated on each update.

## Endpoint: create draft

- **Method and route:** `POST /v1/draft`
- **When to use:** to register a shipment, even with missing data.
- **Processing:** asynchronous. The `200` response only confirms receipt; the draft appears in the
  `GET` a few moments later.

### Parameters — payload root

"Input" = rule checked in the call (`422` error). "Dispatch" = rule checked later, which leaves
the draft `incomplete`.

| Field | Type | Input | Dispatch | Min. | Max. | Default | Values / description |
|---|---|---|---|---|---|---|---|
| `channel_identifier_number` | string | optional | optional | - | 100 | - | Your order identifier. **Recommended**: it is how you locate the draft and the shipment |
| `channel_code` | integer | optional | optional | - | - | account preset | Registered sales channel. At dispatch it must belong to your account |
| `cost_center_code` | integer | optional | optional | - | - | account preset | Registered cost center of your account (or of authorized accounts) |
| `carrier_code` | integer | optional | optional | - | - | account preset | Preferred contract. Used if it is available in the quotation result |
| `object` | string | optional | optional | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | string | optional | optional | - | 30 | `Commercial` (or preset) | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `tax` | string | optional | **required** | - | - | account preset | `sender`, `receiver`. See [Party responsible for the tax](#party-responsible-for-the-tax-tax) |
| `insurance` | boolean | optional | **required** | - | - | account preset | Purchase insurance |
| `measurement` | string | optional | optional | - | - | preset or `metric` | `metric`, `imperial` |
| `ior` | boolean | optional | optional | - | - | `false` | Importer of Record |
| `residential_delivery` | boolean | optional | optional | - | - | `false` (or preset) | Residential delivery |
| `non_stackable` | boolean | optional | optional | - | - | `false` (or preset) | Non-stackable packages |
| `currency_quote` | string | optional | **needed** (see note) | 3 | 3 | account preset | Quotation currency. See [currencies](api-primeiros-passos.md#accepted-currencies) |
| `currency_payment` | string | optional | **needed** (see note) | 3 | 3 | account preset | Payment currency. See [currencies](api-primeiros-passos.md#accepted-currencies) |
| `nfkey` | string | optional | optional | 44 | 44 | - | NF-e key |
| `description` | string | optional | optional | - | 155 | - | Free description |
| `address_sender` | object | optional | **required** | - | - | account preset | Sender (see table) |
| `address_receiver` | object | optional | **required** | - | - | - | Receiver (see table) |
| `boxes` | array | optional | see rules | - | - | - | Packages (see table) |
| `items` | array | optional | see rules | - | - | - | Items (see table) |
| `payment` | object | optional | **required** | - | - | - | Payment (see table) |
| `declared_freight` | object | optional | optional | - | - | account preset | Declared freight: `value` (number) and `currency` (3 letters). It is not checked on input; it is passed on to the quotation |
| `biller` | object | optional | optional | - | - | account preset | Biller displayed on the invoice (same structure as the sender) |
| `manufacturer` | object | optional | optional | - | - | - | Manufacturer displayed on the invoice (see table) |

> **Note on currencies:** the dispatch validation does not report the absence of `currency_quote` and
> `currency_payment`, but without them (in the payload or in the preset) the quotation generation fails and the draft
> goes to `error`, which **cannot be edited**. Always provide both currencies or configure the preset.

### Parameters — `address_sender` and `address_receiver`

Two ways to provide the **sender** (do not mix them):
- **Form A — registered address:** send only `address_sender.address_code`. At dispatch, the registered
  address is copied into the draft and validated with the same rules below.
- **Form B — manual fields:** send the fields below and do **not** send `address_code`.

The receiver accepts **only Form B** (`address_receiver.address_code` is not used).

| Field | Type | Min. | Max. (input) | Max. (dispatch) | Required at dispatch | Description |
|---|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | - | Registered address code (only `address_sender`) |
| `country_code` | string | 2 | 2 | 2 | **Yes** | Country in ISO 3166-1 alpha-2 |
| `postal_code` | string | - | 12 | 12 | Yes, if `state` is not provided | Postal code/ZIP |
| `state` | string | - | - | - | Yes, if `postal_code` and `state_code` are not provided | State name |
| `state_code` | string | - | - | - | See note | State abbreviation, checked against the state database. For a country without states, use the country code itself |
| `city` | string | - | 100 | 100 | **Yes** | City |
| `street` | string | - | 70 | 70 | **Yes** | Street |
| `number` | string | - | 100 | **6** | - | Building number |
| `district` | string | - | 30 | 30 | - | District |
| `complement` | string | - | 20 | 20 | - | Address complement |
| `description` | string | - | 155 | 155 | - | Internal label of the address |
| `name` | string | 5 | 35 | 35 | **Yes** | Contact name/company name |
| `email` | string | - | 60 | 60 | **Yes** | Valid e-mail |
| `phone` | string | - | 25 | 25 | **Yes** | Phone |
| `phone_extension` | string | - | 5 | 5 | - | Phone extension |
| `type` | string | - | - | - | **Yes** (no default) | `pf` or `pj` |
| `federal_tax_id` | string | - | 20 | 20 | Yes when the country is `BR` | Federal document. For `BR` without `foreign: true`, it must be a valid CPF or CNPJ |
| `state_tax_id` | string | - | 20 | 20 | Yes when the country is `BR` | State registration |
| `foreign` | boolean | - | - | - | - | Indicates a foreign person (waives the CPF/CNPJ check) |

> **Address location:** provide `postal_code` **or** `state`. `state_code` alone is not enough,
> because, without `state`, the `postal_code` becomes required.
>
> **Watch out for `number`:** the input accepts up to 100 characters, but the dispatch only accepts up to **6**.
> Longer values leave the draft `incomplete`.

### Parameters — `boxes[]`

When you **send `boxes`**, each box must be provided in **one** of the forms below (do not mix them
in the same box):

| Form | Fields | What happens |
|---|---|---|
| A — kit | `kit` | The box and the kit items are filled in from the registered kit |
| B — registered box by code | `box_code` | Uses the [registered box](api-caixa-v1.md) |
| C — registered box by identifier | `box_identifier` | Uses the registered box |
| D — manual box | `name`, `height`, `width`, `depth` | **All four fields are required at dispatch** |

| Field | Type | Min. | Max. | Description |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU of the registered kit |
| `box_code` | integer | - | - | Code of the registered box |
| `box_identifier` | string | - | - | Identifier of the registered box |
| `name` | string | 4 | 155 | Box name |
| `height` | number | 1 | - | Height |
| `width` | number | 1 | - | Width |
| `depth` | number | 1 | - | Depth |
| `volume` | number | 0.001 | - | Volume. Accepted, but it **does not waive** the dimensions at dispatch |
| `weight` | number | 0.1 | - | Package weight, up to 3 decimal places. **Required (already on input) when `object` is `doc`**. Ignored in a box with `kit` |
| `price` | number | 0 | - | Declared value. **Required (already on input) when `object` is `doc`**. Ignored in a box with `kit` |

> With `boxes` and `object` other than `doc`, **every box that is not a kit must have at least one item**
> pointing to it in `items[].box_id`. With `object: doc`, the `items` are discarded.

### Parameters — `items[]`

Two ways to provide each item:
- **Form A — registered item:** `sku` (+ `quantity`, `unit_value` and, with `boxes`, `box_id`). The missing
  data comes from the [item registration](api-itens-v1.md). The SKU must exist.
- **Form B — manual item:** all fields marked as required without `sku`.

| Field | Type | Min. | Max. | Required at dispatch | Description |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU of the registered item |
| `ean` | string | - | 50 | - | EAN code |
| `nsmp` | string | - | 50 | - | *Non Standard Manufacturer Product Id*: non-standardized product identifier of the manufacturer. Printed on the invoice and used in the regulatory information required by some carriers for destinations in the European Union. With `sku`, if it is not sent, it comes from the item registration |
| `box_id` | integer | - | - | Yes, when there are `boxes` | **Position** of the box in the `boxes` array (1 = first). Without `boxes`, it is ignored |
| `name` | string | 6 | 35 | Yes, without `sku` | Item name |
| `description` | string | 4 | 155 | - | Item description (the minimum of 4 is checked at dispatch) |
| `country_code` | string | 2 | 2 | Yes, without `sku` | Item country of origin |
| `hscode` | string | 6 | 10 | Yes, without `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Yes, without `sku` | Unit weight (the maximum of 99 is checked at dispatch) |
| `quantity` | integer | 1 | 1000 | **Yes** | Quantity |
| `unit_value` | number | - | - | **Yes** | Unit value |
| `selling_unit_value` | number | - | - | Required by the quotation when `ior` is `true` | Unit selling value |
| `height`, `width`, `depth` | number | 1 | - | Yes, when there are **no** `boxes` (except a registered SKU with dimensions) | Item dimensions, used to calculate the boxes |
| `is_groupable` | boolean | - | - | - | Default `true`. Without `boxes`: groupable items go together in a calculated box; `false` generates one box per unit |
| `is_foldable` | boolean | - | - | - | Accepted, but with no effect in the draft |

> **When `items` is required:** without `boxes`, `items` is always required (including for `doc`). With
> `boxes`, it is required for `package`, `pallet` and `not_doc`, except when the boxes use `kit`.
>
> **Without `boxes`:** the platform builds the boxes from the item dimensions. For `doc`, always send
> `boxes` with `weight` and `price`.

### Parameters — `payment`

| Field | Type | Required at dispatch | Values | Description |
|---|---|---|---|---|
| `payment.type` | string | **Yes** | `billed`, `afterward` | The draft **does not accept** `gateway` (`422` error on input) |

### Parameters — `biller`

Same structure and limits as `address_sender`, including `address_code` (Form A) or manual fields
(Form B). At dispatch, all `biller` fields are optional; `number` accepts up to 6 characters.
If you do not send `biller` (or send it empty), the biller from the account preset may be applied.

### Parameters — `manufacturer`

Send `address_code` **or** the manual fields. The manufacturer **is not checked on input**: the limits
below are applied at dispatch.

| Field | Type | Min. | Max. | Required at dispatch | Description |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Registered address code. When used, the other fields are optional |
| `name` | string | 5 | 30 | Yes, without `address_code` | Company name/manufacturer name |
| `country_code` | string | 2 | 2 | Yes, without `address_code` | Country |
| `state_code` | string | 1 | 10 | Yes, without `address_code` | State abbreviation |
| `state` | string | - | 155 | - | State name |
| `postal_code` | string | - | 12 | Yes, without `address_code` | Postal code/ZIP |
| `city` | string | - | 100 | Yes, without `address_code` | City |
| `street` | string | - | 70 | Yes, without `address_code` | Street |
| `number` | string | - | 6 | - | Number |
| `district` | string | - | 30 | - | District |
| `complement` | string | - | 20 | - | Address complement |
| `email` | string | - | 60 | Yes, without `address_code` | E-mail |
| `phone` | string | - | 25 | Yes, without `address_code` | Phone |
| `phone_extension` | string | - | 6 | - | Phone extension |
| `type` | string | - | - | Yes, without `address_code` | `pf` or `pj` |
| `federal_tax_id` | string | - | 20 | - | Federal document |
| `state_tax_id` | string | - | 20 | - | State registration |

### Minimum payload

The smallest accepted payload is an empty object: it creates a draft that will stay `incomplete` until it is completed.

```json
{}
```

In practice, send at least your identifier, so you can locate the draft later:

```json
{
  "channel_identifier_number": "PED-2026-0001"
}
```

Minimum payload for the draft to **pass dispatch** without depending on presets (sender and receiver
outside Brazil, manual box):

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "US",
    "postal_code": "10001",
    "city": "New York",
    "street": "Fifth Avenue",
    "name": "Loja Exemplo",
    "email": "contato@example.com",
    "phone": "12125550100",
    "type": "pj"
  },
  "address_receiver": {
    "country_code": "CA",
    "postal_code": "M5V2T6",
    "city": "Toronto",
    "street": "King Street",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "14165550100",
    "type": "pf"
  },
  "boxes": [
    { "name": "Small box", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Cotton t-shirt",
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

### Maximum payload

All accepted fields. The packages show the manual form (box 1), registered box (box 2)
and kit (box 3); in a real integration use the appropriate form for each package. The sender uses Form B
(manual fields); for Form A, send only `"address_sender": {"address_code": 1}`.

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 1,
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
  "description": "Draft of the marketplace order",
  "declared_freight": {
    "value": 50,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "SP distribution center",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
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
    "description": "Miami office",
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
      "name": "Master box",
      "height": 20,
      "width": 20,
      "depth": 20,
      "volume": 0.008,
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
      "nsmp": "NSMP-TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton short sleeve white t-shirt",
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
    },
    {
      "box_id": 2,
      "name": "Twill shorts M",
      "description": "Beige twill shorts",
      "country_code": "BR",
      "hscode": "620342",
      "weight": 0.4,
      "quantity": 1,
      "unit_value": 59.9,
      "selling_unit_value": 89.9
    }
  ],
  "payment": { "type": "billed" },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Headquarters",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Room 1",
    "name": "Holding Exemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "200",
    "district": "Centro",
    "complement": "Block B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000144",
    "state_tax_id": "111222333444"
  }
}
```

### Request example

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer YOUR_TOKEN' \
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
    "boxes": [{"name": "Small box", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Cotton t-shirt", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Response example

`200 OK`

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

`422 Unprocessable Entity` (e.g.: `payment.type` with `gateway`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "payment.type": ["O campo payment.type não contém um valor válido."]
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success`, `warning` or `error` |
| `message` | string | Confirmation or error message |
| `data` | object | Only in a `422` error: map field → list of messages |

> The success response **does not include the draft code**. Use `GET /v1/draft` to get it.

## Endpoint: list drafts

- **Method and route:** `GET /v1/draft`
- **When to use:** to find out the `codigo` of a draft and follow the `status`/`status_descricao`.
- **Scope:** only drafts of your account. **Items per page:** 20.

### Parameters

| Field | Location | Type | Required | Description |
|---|---|---|---|---|
| `page` | query | integer | no | Listing page (default 1) |
| `drafts` | body | object | no | Filter envelope |
| `drafts.drafts_ids` | body | array | no | Draft codes (integers or strings). **Empty or absent returns all** |

> This `GET` accepts a JSON body for the filter. If your HTTP client does not support a body in `GET`,
> list all of them (paginating) and filter on your side.

### Minimum payload

No body and no query: lists the first page of all drafts.

```
GET /v1/draft
```

### Maximum payload

```
GET /v1/draft?page=1
```

```json
{
  "drafts": {
    "drafts_ids": [30, 31]
  }
}
```

### Request example

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

### Response example

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
          "carrier_code": "",
          "nfkey": "",
          "object": "package",
          "type": "Commercial",
          "channel_code": "",
          "channel": "",
          "tax": "sender",
          "insurance": true,
          "ior": false,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "description": "",
          "address_sender": {
            "country_code": "",
            "state": "",
            "state_code": "",
            "postal_code": "",
            "city": "",
            "description": "",
            "street": "",
            "number": "",
            "district": "",
            "complement": "",
            "name": "",
            "email": "",
            "phone": "",
            "type": "",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "address_receiver": {
            "country_code": "US",
            "state": "",
            "state_code": "",
            "postal_code": "33101",
            "city": "Miami",
            "description": "",
            "street": "Main Avenue",
            "number": "",
            "district": "",
            "complement": "",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "items": [
            {
              "box_id": 1,
              "name": "Cotton t-shirt",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9
            }
          ],
          "boxes": [
            { "name": "Small box", "height": 20, "width": 20, "depth": 20 }
          ],
          "payment": { "type": "billed" }
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "links": [
      { "url": null, "label": "« Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/draft?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo »", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.data[].codigo` | integer | **Draft code.** Use it in `PUT /v1/draft/{code}` and in `drafts.drafts_ids` |
| `data.data[].codigo_usuario` | integer | Code of the account that owns the draft |
| `data.data[].data` | object | Draft content (already merged with the presets), in a fixed format: missing fields come as `""` |
| `data.data[].data.channel` | string | Sales channel name (when there is a `channel_code`) |
| `data.data[].status` | string | Current status (see [lifecycle](#draft-lifecycle)) |
| `data.data[].status_descricao` | string/null | **When `incomplete` or `error`, it tells what prevented progress** (up to 254 characters) |
| `data.data[].created_at`, `updated_at` | string | Dates in the `YYYY-MM-DD HH:MM:SS` format |
| `data.current_page`, `last_page`, `per_page`, `total`, `from`, `to` | integer | Pagination data |
| `data.first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url`, `path`, `links` | string/array | Navigation links between pages |

> The listing's `data` object **does not include** `measurement`, `residential_delivery`, `non_stackable`,
> `cost_center_code`, `declared_freight`, `biller`, `manufacturer` or `address_sender.address_code`.
> When the sender was provided by `address_code`, after the first dispatch it appears expanded
> with the data of the registered address.
>
> If there is no draft for the filters (or the requested page is beyond the last one), the response is
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`, and not an empty list.

## Endpoint: update draft

- **Method and route:** `PUT /v1/draft/{code}`
- **When to use:** to correct an `incomplete` draft.
- **Restriction:** only drafts of your account with `incomplete` status.
- **Body:** same fields and rules as the `POST`. **The previous content is fully replaced**
  (see [The PUT replaces the entire draft](#the-put-replaces-the-entire-draft)).
- **Effect:** the status changes to `edited` and the draft goes back to the next dispatch cycle.
  The previous `status_descricao` remains until the new validation.

### Parameters

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `code` | path | integer | **yes** | Draft code (`codigo` in the listing) |
| body | body | object | no | Same structure as the [`POST`](#endpoint-create-draft) |

### Minimum payload

`PUT /v1/draft/30` with body `{}`. It is accepted, but it **erases** the draft content (only the presets remain).
Use it only if you want to restart the draft.

```json
{}
```

### Maximum payload

The same [maximum payload of the `POST`](#maximum-payload). In practice, resend the complete draft with the corrections.

### Request example

Draft that was `incomplete` due to the missing sender: the body brings the original content **plus** the sender.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer YOUR_TOKEN' \
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
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "number": "100", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Small box", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Cotton t-shirt", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Response example

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso.",
  "data": {
    "codigo": 30,
    "codigo_usuario": 1,
    "data": {
      "channel_identifier_number": "PED-2026-0001",
      "object": "package",
      "type": "Commercial",
      "tax": "sender",
      "insurance": true,
      "measurement": "metric",
      "currency_quote": "USD",
      "currency_payment": "BRL",
      "address_sender": {
        "country_code": "BR",
        "postal_code": "09941070",
        "city": "Diadema",
        "street": "Rua Principal",
        "number": "100",
        "name": "Loja Exemplo LTDA",
        "email": "contato@example.com",
        "phone": "11989677465",
        "type": "pj",
        "federal_tax_id": "12345678000199",
        "state_tax_id": "123456789012"
      },
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
        { "name": "Small box", "height": 20, "width": 20, "depth": 20 }
      ],
      "items": [
        {
          "box_id": 1,
          "name": "Cotton t-shirt",
          "country_code": "BR",
          "hscode": "610910",
          "weight": 0.25,
          "quantity": 3,
          "unit_value": 29.9
        }
      ],
      "payment": { "type": "billed" }
    },
    "status": "edited",
    "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
    "created_at": "2026-02-20 19:04:00",
    "updated_at": "2026-02-20 19:20:31"
  }
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `data.codigo` | integer | Draft code |
| `data.codigo_usuario` | integer | Code of the account that owns the draft |
| `data.data` | object | Stored content: the sent payload already merged with the presets. It may contain additional fields coming from the account presets; ignore the ones you do not know |
| `data.status` | string | Always `edited` after the update |
| `data.status_descricao` | string/null | Reason from the previous validation (it will be updated at the next dispatch) |
| `data.created_at`, `data.updated_at` | string | Dates in the `YYYY-MM-DD HH:MM:SS` format |

## Endpoint: remove drafts

- **Method and route:** `DELETE /v1/draft`
- **When to use:** to discard drafts that will not be used, including those that ended up in `error`.
- **Scope:** only the codes that belong to your account are removed; the others are ignored.
  The removal is permanent.

### Parameters

| Field | Location | Type | Required | Description |
|---|---|---|---|---|
| `drafts` | body | object | **yes** | Envelope |
| `drafts.drafts_ids` | body | array | **yes** (not empty) | Draft codes to remove (integers or strings) |

### Minimum payload

```json
{ "drafts": { "drafts_ids": [30] } }
```

### Maximum payload

```json
{ "drafts": { "drafts_ids": [30, 31, 32] } }
```

### Request example

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

### Response example

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Response fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `success` when at least one draft from the list was removed |
| `message` | string | Confirmation message |

> It is enough for **one** code from the list to belong to your account for the response to be `success`; the others are
> ignored without warning. If no code matches, the response is
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Business rules

### Validation at dispatch
At dispatch, the platform applies a **strict** set of rules. What is missing appears in
`status_descricao` and the draft stays `incomplete`.

| Group | Requirements |
|---|---|
| General | `tax` and `insurance` present (in the payload or via preset); `channel_identifier_number` text of up to 100 characters |
| Sender and receiver | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`postal_code` or `state`); `number` with up to 6 characters |
| Addresses in Brazil (`BR`) | `federal_tax_id` and `state_tax_id`; `federal_tax_id` must be a valid CPF or CNPJ, unless `foreign: true` |
| Without `boxes` | `items` required (any `object`), with dimensions to calculate the boxes |
| With `boxes` | Each box: `kit`, `box_code`, `box_identifier` or (`name` + `height` + `width` + `depth`). For `object` ≠ `doc`, each box (except kit) needs at least one item with its `box_id` |
| `object` = `doc` | `boxes[].weight` and `boxes[].price` (already checked on input) |
| Items | `quantity` (1 to 1000) and `unit_value`; without `sku`: `name`, `country_code`, `hscode`, `weight` (up to 99). With `sku`, the SKU must be registered |
| Payment | `payment.type` (`billed` or `afterward`) |
| Manufacturer | With `manufacturer` without `address_code`: `name`, `country_code`, `state_code`, `postal_code`, `city`, `street`, `email`, `phone` and `type` |
| Channel | `channel_code`, when provided (or coming from the preset), must belong to your account |

### Generation of the quotation and the shipment
Once validated, the draft becomes a quotation and a shipment automatically:

| Step | Behavior |
|---|---|
| Kits | Each `boxes[].kit` is replaced by the box and the items of the registered kit |
| Boxes | With `boxes`, it uses the sent boxes. Without `boxes`, it calculates the boxes from the item dimensions (groupable ones together; non-groupable ones, one box per unit) |
| Quotation | Simple type for `doc` and advanced for the others. Uses `declared_freight` from the payload or from the preset |
| Contract choice | If `carrier_code` is among the results, it is used. Otherwise, the platform chooses based on the account preference: the **cheapest** or the **fastest** |
| Shipment | Empty `type` becomes `Commercial`; `nfkey`, `cost_center_code` and `biller` are passed on when provided |
| Refusal by business rule | The draft goes back to `incomplete`, with the quotation/shipment message in `status_descricao` |
| Unexpected failure | The draft goes to `error`. Remove it and create a new one |

### Possible messages in `status_descricao`
In addition to the field validation messages (e.g.: `O campo address sender.country code é obrigatório.`),
the following may appear:

| Message | Cause | How to fix it |
|---|---|---|
| `Nenhum item foi enviado para este rascunho` | No `items` (and no kits) | Send `items` |
| `O item com o sku :sku não foi encontrado!` | SKU not registered and item without complete data | Register the item or send the manual fields |
| `Nenhuma caixa foi enviado para este rascunho` | Empty `boxes`, or `box_code`/`box_identifier` not found | Check the boxes |
| `Parâmetros da caixa não foram prenchidos corretamente por favor certifique-se dos seguintes campos (name,height,width,depth)` | Manual box without `name`, `height`, `width` or `depth` | Complete all four fields |
| `Nenhum item foi associado para a caixa :name !` | Box without an item with its `box_id` | Adjust `items[].box_id` |
| `Não há itens válidos, verifique os dados e/ou o codigo SKU` | Without `boxes`, no usable item | Check the items and SKUs |
| `O canal especificado não foi encontrado!` | `channel_code` does not belong to your account | Fix the `channel_code` |
| `O kit :sku é inválido!` | Nonexistent or inactive kit | Check the kit SKU |
| `A caixa do kit :sku é inválida!` | The kit box was not found | Review the kit registration |
| `Um ou mais itens do kit :sku são inválidos!` | Kit item not found | Review the kit registration |
| Quotation or shipment messages | E.g.: no contracts available, cargo restrictions | See [Quotation](api-cotacao-v1.md) and [Shipment](api-remessa-v1.md) |

## Common errors

| Status | Meaning / Message | How to fix it |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | Only edit `incomplete` drafts; check the status in the `GET` |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Check the `code`/`drafts_ids` (it may belong to another account or may have already been converted into a shipment and removed) |
| `403` | `Has no permission!` | Ask support for the permission for the action (create, edit, list or remove drafts) |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + `data` with the fields | Fix the fields listed in `data` |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` (in `data.cost_center_code`) | Use a cost center of your account or of authorized accounts |
| `422` | `O campo drafts é obrigatório.` / `O campo drafts.drafts ids é obrigatório.` (in the `DELETE`) | Send `drafts.drafts_ids` with at least one code |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Retry with backoff; if it persists, contact support |

Authentication errors (invalid token, blocked account) are in
[Getting Started](api-primeiros-passos.md#authentication-errors).

## Final notes
- The **`status_descricao` field is your guide**: it tells what prevented the draft from moving forward.
- Always provide `channel_identifier_number`: it is what ties the draft to your order, lets you find the
  `codigo` in the listing and query the shipment after the draft is removed.
- In the `PUT`, **resend the complete draft**: the previous content is replaced.
- If the account has a tax preset, the sent `tax` is ignored; check the stored value in the `GET`.
- The draft **does not accept** payment via `gateway`. For that scenario use
  [quotation](api-cotacao-v1.md) + [shipment](api-remessa-v1.md).
- For the opposite flow (everything validated at once, immediate documents), see [Manifest](api-manifesto.md).
- Registrations used by the draft: [Box](api-caixa-v1.md), [Items](api-itens-v1.md), [Contracts](api-contrato-v1.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
