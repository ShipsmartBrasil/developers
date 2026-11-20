# Manifest

- [Back to the language README](README.md)
- Prerequisite: [Getting Started](api-primeiros-passos.md)

> ⚠️ **Examples are for illustration only:** all payloads, responses, identifiers, codes, documents and tokens
> on this page are fictitious and **are not valid data**. Replace them with the real data of your account and your operation.

## Overview
- Runs **quotation + shipment + submission to the carrier + label generation in a single synchronous call**.
- The response already contains the shipment identifier, the tracking number and the label (AWB) in PDF and/or ZPL.
- It is the recommended flow for operations that need the label **immediately**, without waiting for
  asynchronous processing.
- It only considers carrier contracts that support synchronous dispatch (currently **FedEx, UPS and DHL**).
- It only accepts **billed payment** (`payment.type: billed`), and the account must have this payment method enabled.

> ⚠️ **The manifest is synchronous and irreversible.** Since it quotes and dispatches in the same request, there is no
> intermediate review stage. Validate the payload thoroughly before sending it.

## Base URL
- Production: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Interactive documentation: `https://api.shipsmart.global/doc` and `https://hmlapi.shipsmart.com.br/doc`

## Authentication
```
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json
Accept: application/json
lang: pt
```

> The `lang` header accepts `pt`, `en` and `es` (default `pt`). The token profile must have the
> **create manifest** permission; without it the response is `403` with `Has no permission!`.

## Usage flow
1. Assemble the complete payload (addresses, boxes, items, payment). Check the section
   [Manual address: send all keys](#manual-address-send-all-keys).
2. `POST /v1/manifest?format=pdf` (or `zpl`, or without `format` to receive both).
3. Print the label received in `data.docs` (use the `base64` or download it from the `url`).
4. Store the `data.shipping` and use it for [tracking](api-rastreio.md), [documents](api-documentos.md) and
   [pickup scheduling](api-agendamento.md).
5. A few moments later, query [Shipment Documents](api-documentos.md) to obtain the complementary
   documents (invoice, packing list etc.), which are generated in the background.

## Key concepts

### Manifest vs. quotation + shipment

| Topic | Quotation + Shipment | Manifest |
|---|---|---|
| Calls | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Carrier choice | You choose from the returned list | The platform chooses (or you indicate it with `carrier_code`/`slug`) |
| Carriers | All the account contracts | **Only FedEx, UPS and DHL** |
| Label (AWB) | Later query at `/docs` | **Comes in the response** |
| Payment | `billed`, `gateway`, `afterward` | **Only `billed`** |
| Response type | Asynchronous | **Synchronous** |
| Review before dispatch | Yes | No |
| Required fields | Fewer (split into 2 stages) | **More** (everything at once) |

Use the manifest when: the operation is high volume, the carrier is already defined by a business
rule and you need to print the label on the spot.

### How the carrier is chosen

| What you send | Behavior |
|---|---|
| Neither `slug` nor `carrier_code` | The platform chooses based on the preference configured in the account: **fastest** (shortest transit time, default) or **cheapest** (lowest amount in the payment currency) |
| `carrier_code` | It is a **preference**. If the contract appears in the quotation result, it is used. If it does not: nonexistent contract or contract of a carrier compatible with the manifest → the platform falls back to the account preference; existing contract of a carrier **not** compatible with the manifest → `404` |
| `slug` | **Pins** the contract: the quotation evaluates only that contract and the `slug` **overrides** any `carrier_code` sent. If the contract is not among the eligible contracts of your account for that shipment, the quotation goes back to evaluating all of them and the choice follows the `carrier_code` rule |

> Do not mix `slug` and `carrier_code`: use one or the other. If you send both, the `slug` prevails.
> When the account has a preset preferred contract and you do not send `carrier_code`, the preset is used as `carrier_code`.

### Account presets in the manifest

The manifest applies the [account presets](api-primeiros-passos.md#account-presets)
**after** input validation. For this reason, all required fields must be in the payload,
even if the account has a preset for them.

| Preset | When it is applied in the manifest |
|---|---|
| Tax payer (`tax`) | **Whenever it is configured: it replaces the `tax` sent** (see the table below) |
| Cost center (`cost_center_code`) | When the payload does not send the field or sends it empty (`null`, `""` or `0`) |
| Sales channel (`channel_code`) | When the `channel_code` key is not sent. Sending `"channel_code": null` disables the preset |
| Preferred contract (`carrier_code`) | When the `carrier_code` key is not sent |
| Declared freight (`declared_freight`) | When the `declared_freight` key is not sent. Sending `declared_freight` (even `null` or `{}`) discards the preset |
| Biller (`biller`) | When biller display is enabled in the account and the payload does not send `biller`, or sends it empty (`null`, `{}` or only with empty values). A filled-in `biller` in the payload always prevails |
| Other presets (`insurance`, `measurement`, `type`, currencies, residential delivery, non-stackable, sender address) | **Have no effect**: these fields are required in the manifest and the payload value always prevails |

**Resolution of the tax payer (`tax`):**

| Account configuration | `tax` actually used |
|---|---|
| No tax preset | The value sent in the payload |
| Default `sender` or `receiver` | The account default, **ignoring** the value sent |
| Default + destination country in the account exception list | The **opposite** of the default (`sender` ↔ `receiver`) |
| Locked mode | Always `sender`, regardless of the payload and the exceptions |

> The destination country used in the exceptions is the payload `address_receiver.country_code`. If the receiver
> is informed only by `address_code`, the per-country exceptions are not evaluated and the account default applies.
> Even when the preset replaces the value, `tax` remains required in the payload and must be `sender` or `receiver`.

### Processing stages and what happens in case of failure

| Stage | If it fails |
|---|---|
| 1. Input validation | `422`. Nothing is created |
| 2. Account rules and completion (payment, channel, `slug`, addresses, registered boxes, items and kits, state) | `400`/`404`. Nothing is created |
| 3. Quotation, shipment registration and submission to the carrier | Everything is rolled back: no quotation or shipment is recorded in the account |
| 4. Label and document generation (after the carrier accepts the shipment) | The shipment **remains created**, with a tracking number, and receives a processing error event in the tracking. Retrieve the documents in [Shipment Documents](api-documentos.md) and do not repeat the manifest |

## Endpoint: create manifest

- **Method and route:** `POST /v1/manifest`
- **When to use:** to create, in a single call, a shipment already dispatched to the carrier and
  receive the label in the response.

### Parameters — query

| Name | Location | Type | Required | Default | Values | Description |
|---|---|---|---|---|---|---|
| `format` | query | string | no | empty (all) | `pdf`, `zpl` | Label format in the response. Empty (or an unrecognized value) returns PDF **and** ZPL |

> `format` can also be sent **in the body** of the payload. When sent in both places,
> **the body value prevails**. In the body, values other than `pdf`/`zpl` are rejected with `422`.

### Parameters — payload root

| Field | Type | Required | Min. | Max. | Values / Default | Description |
|---|---|---|---|---|---|---|
| `object` | string | **yes** | - | - | `doc`, `package`, `pallet`, `not_doc` | What is being shipped. `not_doc` is accepted for compatibility and treated as `package` |
| `type` | string | **yes** | - | - | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Purpose of the shipment (exact value, with uppercase letters and spaces) |
| `tax` | string | **yes** | - | - | `sender`, `receiver` | Who pays the taxes. It can be replaced by the account preset (see [Presets](#account-presets-in-the-manifest)) |
| `insurance` | boolean | **yes** | - | - | - | Purchase insurance. It can be enabled automatically if the account has automatic insurance purchase and the declared value reaches the configured minimum |
| `residential_delivery` | boolean | **yes** | - | - | - | Delivery to a residential address |
| `non_stackable` | boolean | **yes** | - | - | - | Non-stackable packages |
| `measurement` | string | **yes** | - | - | `metric`, `imperial` | Measurement system for weights and dimensions |
| `ior` | boolean | **yes** | - | - | - | Importer of Record |
| `currency_quote` | string | **yes** | 3 | 3 | see [currencies](api-primeiros-passos.md#accepted-currencies) | Quotation currency (and currency of the item values) |
| `currency_payment` | string | **yes** | 3 | 3 | see [currencies](api-primeiros-passos.md#accepted-currencies) | Payment currency |
| `address_sender` | object | **yes** | - | - | - | Sender (see table) |
| `address_receiver` | object | **yes** | - | - | - | Receiver (see table) |
| `boxes` | array | **yes** | 1 item | - | - | Packages (see table) |
| `payment` | object | **yes** | - | - | - | Payment (see table) |
| `items` | array | conditional | - | - | - | Required when `object` is **not** `doc` and no box uses `kit`. **Forbidden (the key must not exist) when `object` is `doc`** |
| `carrier_code` | integer | no | - | - | - | Preferred contract (see [How the carrier is chosen](#how-the-carrier-is-chosen)) |
| `slug` | string | no | - | 20 | - | Text identifier that pins the contract. It must exist; a removed or expired contract returns `400` |
| `channel_code` | integer | no | - | - | - | Code of the sales channel registered **in your account** |
| `channel_identifier_number` | string | no | - | - | - | Your business identifier (e.g., order number). **Recommended** |
| `cost_center_code` | integer | no | - | - | Account preset | Cost center registered for your account or for authorized associated accounts. Empty/absent → uses the account preset |
| `nfkey` | string | no | 44 | 44 | - | NF-e key (exactly 44 characters) |
| `description` | string | no | 5 | 155 | - | Free-text description of the manifest |
| `declared_freight` | object | no | - | - | Account preset | Declared freight (see table) |
| `biller` | object | no | - | - | Account preset | Biller displayed on the invoice (see section) |
| `manufacturer` | object | no | - | - | - | Manufacturer displayed on the invoice (see section) |
| `extra` | object | no | - | - | - | Auxiliary data in key/value pairs (see table) |
| `format` | string | no | - | - | `pdf`, `zpl` | Alternative to the query string (prevails over it) |
| `size` | string | no | - | - | `label`, `letter` | Accepted by validation, but **currently does not change** the generated label |

### Parameters — `address_sender` and `address_receiver`

Unlike shipment v1/v2, here the address is informed **in full**, because the quotation and the dispatch
happen in the same call. There are two ways to inform each address — **do not mix both in the same object**.

**Form A — already registered address:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address_code` | integer | **yes** in this form | Code of an address registered and visible to your account |

> With `address_code`, **all** address data comes from the registration; any other field sent in the same
> object is discarded. If you send `address_code` together with `country_code`, `street`, `name`, `email`,
> `phone`, `type` and `foreign`, the request is rejected with `422`. Sender and receiver cannot use the
> same `address_code`.

**Form B — manual address:**

| Field | Type | Required | Min. | Max. | Default | Description |
|---|---|---|---|---|---|---|
| `country_code` | string | **yes** | 2 | 2 | - | Country in ISO 3166-1 alpha-2 |
| `postal_code` | string | conditional | - | 12 | - | Postal code/ZIP. Required if `state` and `state_code` are not informed. In Brazil, it is used to discover the state when `state_code` is not sent |
| `state` | string | conditional | - | 100 | - | State name. Required if `state_code` and `postal_code` are not informed |
| `state_code` | string | conditional | 1 | 10 | - | State abbreviation. Required if `state` and `postal_code` are not informed. **Always recommended** (see the business rules) |
| `city` | string | conditional | 1 | 100 | - | City. Required if `postal_code` is not informed. **Always recommended**: carriers use the city on the label |
| `street` | string | **yes** | 5 | 70 | - | Street |
| `number` | string | no | 1 | 6 | - | Building number |
| `district` | string | no | 1 | 30 | - | District/neighborhood |
| `complement` | string | no | 1 | 20 | - | Address complement |
| `name` | string | **yes** | 1 | 70 | - | Contact name/company name |
| `email` | string | **yes** | - | 60 | - | Valid e-mail |
| `phone` | string | no | 1 | 25 | - | Phone. **Always recommended**: carriers require a contact phone number |
| `phone_extension` | string | no | - | 5 | - | Phone extension |
| `type` | string | **yes** | - | - | - | `pf` (individual) or `pj` (legal entity) |
| `foreign` | boolean | **yes** | - | - | - | Indicates whether the person is a foreigner (used to validate the `federal_tax_id`) |
| `federal_tax_id` | string | conditional | - | 20 | - | Federal tax document (CPF/CNPJ or equivalent). **Required when `country_code` is `BR` or `AR`** |
| `state_tax_id` | string | no | - | 20 | - | State tax registration |
| `description` | string | no | 1 | 155 | - | Internal label of the address |
| `company_name` | string | no | 5 | 50 | - | **Only in `address_sender`.** Company name used in FedEx shipments (the carrier considers only the first 35 characters) |

#### Manual address: send all keys

In the manifest, the keys below must **exist** in the manual address object, even when you have no
value for them — in that case, send `null`. The absence of any of them interrupts processing with an error
(`500`, or `400` at the carrier stage).

| Keys that must be present (value can be `null`) |
|---|
| `postal_code`, `city`, `number`, `district`, `complement`, `phone`, `phone_extension`, `federal_tax_id`, `state_tax_id` |

> Also do not send `"address_code": null` in form B: the `address_code` key must be omitted.

### Parameters — `boxes[]`

Each box can be informed in three ways — **do not mix the forms in the same box**:

| Form | Fields | Behavior |
|---|---|---|
| A — registered box | `box_code` **or** `box_identifier` | Name, dimensions and weight come from the [box registry](api-caixa-v1.md) |
| B — manual box | `name`, `height`, `width`, `depth` (+ `weight`/`price` in `doc`) | Data informed in the payload |
| C — kit | `kit` | The box **and** the items of that position come from the registered kit |

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `kit` | string | no | - | 50 | SKU of the registered kit. Fills in the box and the items of that position; the box `weight` and `price` are ignored |
| `box_code` | integer | conditional | - | - | Code of the registered box. Required when the box does not inform any other field |
| `box_identifier` | string | conditional | - | 50 | Identifier of the registered box. Alternative to `box_code` |
| `name` | string | **yes** without `kit`/`box_code`/`box_identifier` | 5 | 155 | Box name |
| `height` | number | **yes** without `kit`/`box_code`/`box_identifier` | 1 | - | Height (cm in `metric`, in in `imperial`) |
| `width` | number | **yes** without `kit`/`box_code`/`box_identifier` | 1 | - | Width |
| `depth` | number | **yes** without `kit`/`box_code`/`box_identifier` | 1 | - | Depth |
| `weight` | number | **yes** when `object` is `doc` (including with a registered box) | 0.001 | - | Package weight (kg in `metric`, lb in `imperial`) |
| `price` | number | **yes** when `object` is `doc` (including with a registered box) | 0 | - | Declared value of the package. In `doc`, the declared value of the shipment is the sum of the `price` values |

> With a registered box, the fields sent in the payload complement (and override) those of the registration.
> The box must belong to your account.

### Parameters — `items[]`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `box_id` | integer | **yes** | - | - | **Position** of the box in the `boxes` array (1 = first) |
| `quantity` | integer | **yes** | 1 | 99 | Item quantity |
| `sku` | string | conditional | - | 50 | SKU of the registered item ([Items](api-itens-v1.md)). Required when none of `name`, `country_code`, `hscode`, `weight` and `unit_value` is informed |
| `name` | string | **yes** without `sku` | 6 | 50 | Item name |
| `country_code` | string | **yes** without `sku` | 2 | 2 | Country of origin of the item (ISO 3166-1 alpha-2) |
| `hscode` | string | **yes** without `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **yes** without `sku` | 0.001 | 99 | Unit weight (kg in `metric`, lb in `imperial`) |
| `unit_value` | number | **yes** without `sku` | greater than 0 | - | Unit value, in the `currency_quote` currency. Validation accepts `0`, but the quotation rejects an item with zero value |
| `selling_unit_value` | number | no | 0 | - | Unit selling value, used when `ior` is `true`. If omitted, it comes from the item registration (with `sku`) or, failing that, takes the `unit_value` |
| `description` | string | no | 5 | 450 | Detailed description of the item |
| `ean` | string | no | - | 50 | EAN code (barcode) of the item. Used as the standardized manufacturer identifier in the regulatory information for destinations in the European Union (and as the product identifier when there is no `sku`) |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: non-standardized manufacturer product identifier. It is printed on the invoice and used in the regulatory information required by some carriers for destinations in the European Union |

> **Items with `sku`:** when the item informs `sku` and any of the fields `name`, `country_code`, `hscode`,
> `weight`, `unit_value`, `nsmp` (or `selling_unit_value` with `ior: true`) is missing, the missing data is completed
> from the [item registration](api-itens-v1.md) (including `ean` and `nsmp`). The values sent in the payload
> always prevail over the registration. If the SKU is not registered, the item is only accepted if it brings
> `name`, `country_code`, `hscode` and `weight` in the payload itself.
>
> The regulatory information for destinations in the European Union is only built when the item has `sku` or `ean`.

> **Pay attention to the differences in relation to the quotation:** here `quantity` has a maximum of **99** and
> `description` accepts up to **450** characters.

### Parameters — `payment`

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `payment` | object | **yes** | - | Envelope |
| `payment.type` | string | **yes** | **`billed`** | The manifest accepts **only** billed payment. The account must also have billed payment enabled; otherwise, the response is `400` |

### Parameters — `declared_freight`

| Field | Type | Required | Min. | Max. | Description |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Declared freight value. `0` is accepted |
| `currency` | string | **yes** when `value` is informed | 3 | 3 | Declared freight currency |
| `declared` | boolean | no | - | - | Accepted, but **has no effect** in the manifest: the declared freight is only considered when `value` **and** `currency` are informed |

> To declare zero freight, send `"value": 0` with the `currency`. Without `value` and `currency`, the declared freight
> is not applied (nor the preset one, if you sent the `declared_freight` key).

### Parameters — `extra`

Auxiliary key/value pairs, stored in the shipment.

| Rule | Detail |
|---|---|
| Structure | **Flat** object (one level, no nested objects or lists) |
| Keys | Only lowercase letters, digits and `_` |
| Values | Text, integer or decimal (boolean and `null` are rejected) |

| Known key | Type | Rule | Description |
|---|---|---|---|
| `federal_tax_provision` | number | - | Federal tax provision |
| `state_tax_provision` | number | - | State tax provision |
| `usd_rate_provision` | number | greater than 0 | Dollar exchange rate provision. **Only kept** for accounts in the Remessa Conforme program with destination `BR`; in other cases it is silently discarded |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parameters — `biller` and `manufacturer`

- `biller`: same format described in [Shipment (v1)](api-remessa-v1.md#parameters--biller)
  (`address_code` **or** manual fields). In the manifest, `biller.name` has from 5 to **355** characters.
  Fields not sent take `type: pj` and `foreign: false`. Without `biller` in the payload, the account
  biller preset applies (when enabled).
- `manufacturer`: same format described in [Quotation (v1)](api-cotacao-v1.md#parameters--manufacturer)
  (`address_code` **or** manual fields). With the `address_code` of an existing address, the data comes from the registration.

### Minimum payload

Document manifest (`object: doc`) with manual addresses — `items` is forbidden in this case:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": null,
    "district": null,
    "complement": null,
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11999990000",
    "phone_extension": null,
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": null,
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "number": null,
    "district": null,
    "complement": null,
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550000",
    "phone_extension": null,
    "type": "pf",
    "federal_tax_id": null,
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Documents envelope",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

**Minimum variation with registered addresses and kit** (`object: package`; `items` can be omitted because the box uses `kit`):

```json
{
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": { "address_code": 1234 },
  "address_receiver": { "address_code": 5678 },
  "boxes": [
    { "kit": "KIT-001" }
  ],
  "payment": { "type": "billed" }
}
```

### Maximum payload

All accepted fields. `slug` and `carrier_code` appear together only for illustration: in practice use one
or the other (the `slug` prevails). The boxes show the manual form and the registered form.

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 12,
  "carrier_code": 45,
  "slug": "fedex-br",
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
  "description": "Manifest for the march batch",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Distribution center SP",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Warehouse 2",
    "company_name": "Loja Exemplo Comercio LTDA",
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
    "phone": "13055550000",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Master box 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 1.5,
      "price": 90
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7891234567895",
      "nsmp": "MFR-TSH-000-S",
      "name": "Cotton t-shirt S",
      "description": "100% cotton t-shirt, short sleeve, white, size S",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Sport sneaker",
      "description": "Mens running sneaker size 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
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
    "federal_tax_id": "11222333000181",
    "state_tax_id": "111222333444"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Request example

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": false,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "number": null, "district": null, "complement": null, "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11999990000", "phone_extension": null, "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": null, "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "number": null, "district": null, "complement": null, "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550000", "phone_extension": null, "type": "pf", "federal_tax_id": null, "state_tax_id": null, "foreign": true},
    "boxes": [{"name": "Documents envelope", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Response example

`200 OK` (without `format`, the response contains `AWB` and `AWB_ZPL`; with `format=pdf`, only `AWB`; with `format=zpl`, only `AWB_ZPL`):

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "271249639538",
    "tracking_packages_ids": [
      "271249639538"
    ],
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Response fields

| Field | Type | Presence | Description |
|---|---|---|---|
| `data.shipping` | string (UUID) | always | **Shipment identifier.** Use it for tracking, documents and scheduling |
| `data.carrier` | string | always | Name of the contract/carrier that processed the shipment |
| `data.tracking_id` | string | always | Main tracking number (AWB) |
| `data.tracking_packages_ids` | array of string | when the carrier provides it | Tracking number of each package, in the order of the boxes |
| `data.docs` | object | always | Generated labels, in the format `name → { url, base64 }` |
| `data.docs.AWB` | object | `format` empty or `pdf` | Label in PDF |
| `data.docs.AWB_ZPL` | object | `format` empty or `zpl` | Label in ZPL (thermal printers) |
| `data.docs.AWBPRC` / `data.docs.AWBPRCZPL` | object | accounts in the Remessa Conforme program, for imports (origin outside Brazil and destination `BR`) | Program-specific label, in PDF / ZPL |
| `data.docs.*.url` | string | always | File link. If the account protects document links, the link is **temporary** (valid for about 5 minutes) |
| `data.docs.*.base64` | string | always | File content in Base64 |
| `data.valorProvCBS`, `data.valorProvIBSEstadual`, `data.valorProvIBSMunicipal` | number | when the tax calculation includes these provisions | CBS and IBS provisions (state and municipal) |
| `data.serviceType` | string | FedEx shipments | FedEx service used |
| `data.operationalDetail` | object | FedEx shipments | Carrier operational data (`ursaPrefixCode`, `ursaSuffixCode`, `originLocationId`, `originLocationNumber`, `originServiceArea`, `destinationLocationId`, `destinationLocationNumber`, `destinationServiceArea`, `destinationLocationStateOrProvinceCode`, `deliveryDate`, `deliveryDay`, `commitDate`, `commitDay`, `ineligibleForMoneyBackGuarantee`, `astraPlannedServiceLevel`, `astraDescription`, `postalCode`, `stateOrProvinceCode`, `countryCode`, `airportId`, `serviceCode`, `packagingCode`, `publishedDeliveryTime`, `scac`). Fields without a value come as `null` |

> Complementary documents — invoice (shipments that are not `doc`), content declaration (when the
> sender is `pf`), packing list and non-hazardous goods declaration — are generated in the background.
> Query [Shipment Documents](api-documentos.md) a few moments later to obtain the complete list.

### Carrier error response

When the carrier rejects the shipment, the response is `400` with `status: "error"` and contains its raw response:

```json
{
  "status": "error",
  "message": "[1234] Fedex - Teste - Mensagem de erro do transportador",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Field | Description |
|---|---|
| `message` | `[código do contrato] nome do transportador - mensagem` (`[contract code] carrier name - message`) returned by the carrier. If the carrier responds without a recognizable message, it comes as `CARRIER SYNC - SHIPPING RETURN UNEXPECTED RESPONSE!` |
| `data` | Raw carrier payload. **The structure varies by carrier** and it can come as `null` |

> Record `message` and `data` in your logs: they are the only source of the technical reason for the rejection. A carrier
> rejection rolls back the quotation and the shipment; fix the payload and send a new manifest.

## Business rules
- **Payment:** only `billed`, and the account must have billed payment enabled (alone or together
  with immediate payment). Another value in `payment.type` → `422`; account without billing → `400`.
- **Carriers:** the manifest quotes only the FedEx, UPS and DHL contracts of your account. The choice follows
  [How the carrier is chosen](#how-the-carrier-is-chosen).
- **Presets:** applied after validation, as described in
  [Account presets in the manifest](#account-presets-in-the-manifest). Highlights: the preset `tax`
  **replaces** the one sent; the preset `cost_center_code` fills in the field when it comes empty.
- **`items` and `object: doc`:** with `object: doc`, the `items` key **cannot** be sent (not even empty).
  With any other `object`, `items` is required — except when some box uses `kit`.
- **Kits:** when a box informs `kit`, the box and the items of that position come from the registration,
  and `boxes[].weight`/`boxes[].price` are ignored.
- **Declared value:** in `doc`, it is the sum of `boxes[].price`; in the others, the sum of `unit_value × quantity`
  of the items. An item with an empty or zero unit value is rejected in the quotation (`400`).
- **`imperial`:** weights and dimensions informed in pounds/inches are converted to kg/cm before the quotation.
- **State (`state_code`):** it must be a state abbreviation recognized by the platform; for countries without
  division into states, send the `country_code` itself. Without `state_code`, the platform tries to discover it
  (by postal code in Brazil, by the name in `state` or by the postal code abroad). If it cannot do so for the
  sender **or** for the receiver, the manifest is rejected with `400` before any record is created.
- **`federal_tax_id`:** required when the address country is `BR` or `AR`.
- **`address_code`:** the address must be visible to your account; with it, the other fields of the
  object are ignored. Sender and receiver cannot have the same `address_code`.
- **Manual address:** send all the keys listed in
  [Manual address: send all keys](#manual-address-send-all-keys) (use `null` when there is no value).
- **`channel_code`:** it must exist and belong to your account (including when it comes from the preset).
- **`cost_center_code`:** it must exist and belong to your account, to the main account or to authorized
  associated accounts.
- **`slug`:** it must exist (`422`) and the contract must be active and within its validity period (`400`).
- **Automatic insurance:** if the account purchases insurance automatically from a minimum value and the declared
  value (converted to USD) reaches that minimum, the insurance is purchased even with `insurance: false`.
- **`extra.usd_rate_provision`:** kept only for accounts in the Remessa Conforme program with destination `BR`.
- **`not_doc`** is accepted for compatibility and treated as `package`.

## Common errors

Validation errors (`422`) have `status: "warning"`, `message` equal to
`Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!`
and `data` with the `field → messages` map. Business rule errors have `status: "warning"` and do not contain `data`.

| Status | Meaning / Message | How to fix |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Check the token sent in `Authorization` |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contact support |
| `403` | `Has no permission!` | Ask support for the create manifest permission |
| `422` | Field validation (messages in `data`, e.g., `O campo :attribute é obrigatório.`) | Fix the fields listed in `data` |
| `422` | `The :attribute field must be missing when :other is :value.` (in `items`) | Remove the `items` key when `object` is `doc` |
| `422` | `The :attribute field must be missing when :values are present.` (in `address_code`) | Use only `address_code` **or** only the manual fields |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Use a `cost_center_code` of your account |
| `400` | `Tipo de pagamento não autorizado!` | Ask support to enable billed payment |
| `400` | `O canal especificado não foi encontrado!` | Use a `channel_code` registered in your account (also check the preset) |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | Use the `slug` of an active contract |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Use different `address_code` values |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | Check the `address_code` |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Fix the `postal_code` or send `state_code` |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Fix `state` or send `state_code`/`postal_code` |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Send `state_code` in both addresses |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | Check `box_code`/`box_identifier` |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | Register the item or send the manual fields |
| `400` | `O kit :sku é inválido!` / `A caixa do kit :sku é inválida!` / `Um ou mais itens do kit :sku são inválidos!` | Review the kit registration |
| `400` | Quotation message (e.g., `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. ...`, `Nenhum transportador encontrado para este usuário!`) | In the manifest, every error from the quotation stage comes back as `400`. Read `message` and adjust the payload |
| `400` | Shipment registration message | Read `message` and adjust the payload |
| `404` | `A transportadora informada não está disponível para esta remessa!` | Use the `carrier_code` of a FedEx, UPS or DHL contract, or omit the field |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | The chosen contract does not have synchronous dispatch; contact support |
| `400` | `[código] transportador - mensagem` (`status: "error"`, `data` with the carrier response) | See [Carrier error response](#carrier-error-response) |
| `500` | Unexpected error (`status: "error"`) | **Before retrying**, check in [Tracking](api-rastreio.md) whether the shipment was created |

> ⚠️ **Never retry a manifest automatically after a `500` or after a network failure/timeout.** If the failure
> happens after the carrier accepts the shipment, the shipment really exists and a retry generates a
> duplicate shipment. First check by `channel_identifier_number` + `channel_code` in [Tracking](api-rastreio.md).

## Final notes
- Always inform `channel_identifier_number` (with `channel_code`): it is your protection against duplicates and the
  way to locate the shipment if the response is lost.
- The manifest concentrates a lot of responsibility in one call. For integrations that can wait,
  the [quotation](api-cotacao-v1.md) + [shipment](api-remessa-v1.md) flow is safer.
- For the opposite flow (partial payload, validated later), see [Draft](api-rascunho.md).
- Registries used by the manifest: [Box](api-caixa-v1.md) and [Items](api-itens-v1.md).
- Consolidated list of error messages: [Error Codes](api-codigos-de-erro.md).
