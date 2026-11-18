# Cotización (v2)

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- El mismo motor de cálculo de la [Cotización v1](api-cotacao-v1.md), con un **contrato de dirección mucho más reducido**.
- Usa la v2 cuando, en el momento de la cotización, todavía **no tienes** la ciudad ni el código postal del destinatario — solo el país
  (y opcionalmente el estado).
- La ciudad, el código postal y la dirección completa se informan después, en el [Envío V2](api-remessa-v2.md).

> **Importante:** una cotización creada en la v2 solo puede generar un envío en la v2 (`POST /v2/shipping`).
> Consulta el [comparativo](api-comparativo-cotacao-remessa.md) para elegir la versión.

## Base URL
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
Content-Type: application/json
lang: pt
```
Detalles en [Primeros Pasos](api-primeiros-passos.md).

## Flujo de uso
1. `POST /v2/quotation` informando solo el país (y el estado, si lo sabes) en el origen y en el destino.
2. La respuesta ya trae el resultado calculado (`data.carriers`) y el código `data.quotation`.
3. Elige un transportista y crea el [Envío V2](api-remessa-v2.md) con la dirección completa,
   **antes** del `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
   (solo país/estado)          │
                               ▼
                    POST /v2/shipping
             (dirección completa + carrier_code|slug)
```

## Diferencias respecto a la v1

| Tema | v1 | v2 |
|---|---|---|
| Dirección en la cotización | `country_code` + `city` + (`postal_code` o `state_code`/`state`) | Solo `country_code` (+ `state_code` opcional) |
| Dirección en el envío | Calle, nombre, e‑mail, teléfono, tipo | Calle, nombre, e‑mail, teléfono, tipo **+ ciudad, estado y código postal** |
| Consulta del resultado | `GET /v1/quotation/{quotation}/{level}` | **No existe.** El resultado viene en el propio `POST` |
| `ior` (Importer of Record) | Soportado | **No forma parte del contrato de la v2** |
| `items[].selling_unit_value` | Soportado | **No forma parte del contrato de la v2** |
| Precisión del cálculo | Mayor: ciudad/código postal permiten aplicar tasas regionales ya en la cotización | El valor puede refinarse en el envío, cuando se informa la dirección completa |

> Todo el resto (objeto, tipo, cajas, artículos, kits, monedas, flete declarado, fabricante, warnings)
> es idéntico a la v1.

## Conceptos esenciales

### `object` — qué se está enviando
| Valor | Significado | Observaciones |
|---|---|---|
| `doc` | Documento | Cotización por caja, informando `weight` y `price` |
| `package` | Paquete/caja | Flujo más común |
| `pallet` | Palé | Permite informar `volume` en lugar de altura/anchura/profundidad |
| `not_doc` | Legado | Convertido internamente a `package`. **No lo uses en integraciones nuevas** |

### `type` — cómo se monta el cálculo
| Valor | Lo que envías | Cuándo usarlo |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` y `price` | Simulación rápida. **No es elegible para envío cuando `object` es `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensiones) + `items[]` con `box_id` | Flujo estándar para envío |
| `items` | Solo `items[]` con `sku` | Cuando los artículos y las cajas ya están registrados y la plataforma hace el encaje |

### `measurement` — sistema de medida
| Valor | Dimensiones | Peso | Volumen |
|---|---|---|---|
| `metric` (predeterminado) | centímetros | kilogramos | m³ |
| `imperial` | pulgadas | libras | ft³ |

### `level` — nivel de detalle del resultado
| Valor | Retorno |
|---|---|
| `simple` (predeterminado) | Lista reducida de transportistas con totales |
| `detailed` | Incluye la composición detallada (flete, seguro, impuesto, antes y después de las tasas administrativas) |

Enviado como **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: crear cotización

- **Método y ruta:** `POST /v2/quotation`
- **Query string opcional:** `level=simple|detailed`

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Valores aceptados | Descripción |
|---|---|---|---|---|---|---|---|
| `object` | string | **sí** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Qué se está enviando |
| `type` | string | **sí** | - | - | - | `simple`, `advanced`, `items` | Cómo se monta el cálculo |
| `tax` | string | **sí** | - | - | - | `sender`, `receiver` | Quién paga los impuestos |
| `insurance` | boolean | **sí** | - | - | - | `true`, `false` | Contratar seguro |
| `currency_quote` | string | **sí** | 3 | 3 | - | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda en la que se presenta la cotización |
| `currency_payment` | string | **sí** | 3 | 3 | - | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda en la que se hará el pago |
| `address_sender` | object | **sí** | - | - | - | - | Origen (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | - | Destino (ver tabla abajo) |
| `boxes` | array | condicional | - | - | - | - | Obligatorio cuando `type` **no** es `items` |
| `items` | array | condicional | - | - | - | - | Obligatorio cuando `type` **no** es `simple` y ninguna caja usa `kit` |
| `measurement` | string | no | - | - | `metric` | `metric`, `imperial` | Sistema de medida |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Entrega en dirección residencial |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Bultos no apilables |
| `description` | string | no | - | 155 | `null` | - | Descripción libre de la cotización |
| `channel_code` | integer | no | - | - | preset de la cuenta | - | Código del canal de venta registrado |
| `channel_identifier_number` | string | no | - | - | `null` | - | Tu identificador comercial (nº del pedido). **Recomendado** |
| `channel_order_number` | string | no | - | 100 | `null` | - | Número del pedido en el canal de venta |
| `declared_freight` | object | no | - | - | preset de la cuenta | - | Flete declarado (ver tabla abajo) |
| `manufacturer` | object | no | - | - | `null` | - | Fabricante mostrado en la Commercial Invoice (ver tabla abajo) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Nivel de detalle del resultado |

### Parámetros — `address_sender` y `address_receiver`

**Esta es la principal diferencia de la v2.** Elige una de las dos formas:

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de una dirección registrada en tu cuenta |

**Forma B — solo país (y estado, si lo sabes):**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `country_code` | string | **sí** | 2 | 2 | País en ISO 3166-1 alpha-2 (ej.: `BR`, `US`) |
| `state_code` | string | no | - | - | Sigla del estado/provincia (ej.: `SP`, `FL`). Cuando se informa, mejora la precisión del cálculo de impuestos regionales |

> No envíes `city`, `postal_code` ni `state` en la cotización v2 — esos datos se informan en el envío.

### Parámetros — `boxes[]`

Cada ítem del array representa un bulto. Hay **tres formas** de describir la caja:

**Forma A — kit registrado** (completa la caja **y** los artículos automáticamente):

| Campo | Tipo | Obligatorio | Máx. | Descripción |
|---|---|---|---|---|
| `kit` | string | **sí** en esta forma | 50 | SKU del kit registrado. Los demás campos de la caja y los artículos de esa posición se completan con el registro del kit |

**Forma B — caja registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `box_code` | integer | **sí** en esta forma | Código de la caja registrada ([Caja](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa a `box_code` | Identificador de la caja registrada (ej.: `CX0236`) |

**Forma C — caja informada manualmente:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `name` | string | no | 4 | 60 | Nombre/etiqueta de la caja |
| `height` | number | **sí** | 1 | - | Altura |
| `width` | number | **sí** | 1 | - | Anchura |
| `depth` | number | **sí** | 1 | - | Profundidad |
| `volume` | number | condicional | 0.001 | - | **Solo cuando `object` es `pallet`.** Alternativa a altura/anchura/profundidad |

**Campos válidos en cualquier forma (excepto `kit`):**

| Campo | Tipo | Obligatorio | Mín. | Descripción |
|---|---|---|---|---|
| `weight` | number | **sí** cuando `object` es `doc`/`package` **y** `type` es `simple` | > 0 | Peso del bulto |
| `price` | number | **sí** cuando `object` es `doc`/`package` **y** `type` es `simple` | > 0 | Valor declarado del bulto |

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `quantity` | integer | **sí** | 1 | 1000 | Cantidad del artículo |
| `unit_value` | number | **sí** | - | - | Valor unitario en la moneda de la cotización |
| `box_id` | integer | **sí**, excepto cuando `type` es `items` | - | - | **Posición** de la caja en el array `boxes` (1 = primera caja). No es el `box_code` |
| `sku` | string | **sí** cuando `type` es `items` | - | 50 | SKU del artículo registrado ([Artículos](api-itens-v1.md)). Cuando se informa, se usan los datos del registro |
| `ean` | string | no | - | 50 | Código EAN del artículo |
| `name` | string | **sí** cuando no se informa `sku` | 6 | 50 | Nombre del artículo (aparece en la invoice) |
| `description` | string | no | - | 155 | Descripción detallada del artículo |
| `country_code` | string | **sí** cuando no se informa `sku` | 2 | 2 | País de origen/fabricación del artículo |
| `hscode` | string | **sí** cuando no se informa `sku` | 6 | 10 | NCM/HS Code. Define la tabla de impuestos aplicada |
| `weight` | number | **sí** cuando no se informa `sku` | 0.001 | - | Peso unitario del artículo |
| `is_groupable` | boolean | no | - | - | Predeterminado `true`. Cuando es `false`, el artículo recibe una caja exclusiva en el agrupamiento `smartbox` |

### Parámetros — `declared_freight`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valor del flete declarado |
| `currency` | string | **sí** cuando se informa `value` | 3 | 3 | Moneda del flete declarado |

### Parámetros — `manufacturer`

Datos del fabricante impresos en la Commercial Invoice. Envía `address_code` **o** los campos manuales.

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa a los manuales | - | - | Código de dirección registrada |
| `name` | string | **sí** sin `address_code` | 5 | 30 | Razón social/nombre del fabricante |
| `country_code` | string | **sí** sin `address_code` | 2 | 2 | País |
| `state_code` | string | **sí** sin `address_code` | 1 | 10 | Sigla del estado |
| `state` | string | no | - | 155 | Nombre del estado |
| `postal_code` | string | **sí** sin `address_code` | - | 12 | Código postal/ZIP |
| `city` | string | **sí** sin `address_code` | - | 100 | Ciudad |
| `street` | string | **sí** sin `address_code` | - | 70 | Calle |
| `number` | string | no | - | 6 | Número |
| `district` | string | no | - | 30 | Barrio |
| `complement` | string | no | - | 20 | Complemento |
| `email` | string | **sí** sin `address_code` | - | 60 | E-mail |
| `phone` | string | **sí** sin `address_code` | - | 25 | Teléfono |
| `phone_extension` | string | no | - | 6 | Extensión |
| `type` | string | **sí** sin `address_code` | - | - | `pf` o `pj` |
| `federal_tax_id` | string | no | - | 20 | CPF/CNPJ o equivalente |
| `state_tax_id` | string | no | - | 20 | Inscripción estatal |

### Payload mínimo

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

### Payload mínimo elegible para envío

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Camiseta algodon",
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

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "channel_order_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "description": "Pedido del marketplace, lote de octubre",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Camiseta algodon S",
      "description": "Camiseta 100% algodon, manga corta, color blanco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    }
  ],
  "manufacturer": {
    "name": "Fabrica Ejemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Calle Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpon 2",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  }
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v2/quotation?level=detailed' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "USD",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodon", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}]
  }'
```

### Ejemplo de response

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
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "insurance": "11.55",
        "insurance_final": "11.55",
        "tax": "24924.09",
        "tax_final": "24924.09",
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.quotation` | string (UUID) | **Código de la cotización.** Úsalo en `POST /v2/shipping` |
| `data.finished` | boolean | Indica si el procesamiento de la cotización fue concluido |
| `data.warnings` | array | **Opcional.** Solo aparece cuando hay alertas aduaneras |
| `carriers[].code` | integer | Código del contrato. Úsalo en `carrier_code` en el envío |
| `carriers[].slug` | string | Identificador textual del contrato. Alternativa al `carrier_code` |
| `carriers[].valid_until` | string | **Validez de la cotización.** Crea el envío antes de esta fecha/hora |
| `carriers[].transit_days` | integer | Plazo estimado de tránsito en días |
| `carriers[].weight_details.type` | string | `real` o `cubed` — cuál peso se usó en la tarifación |
| `carriers[].freight` / `freight_final` | string | Flete antes / después de las tasas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / después de las tasas administrativas |
| `carriers[].tax` / `tax_final` | string | Impuestos antes / después de las tasas administrativas |
| `carriers[].currency_payment_amount` | string | Total a pagar en la moneda de pago |

## Reglas de negocio

### Elegibilidad para envío
1. El `quotation` existe y pertenece a tu cuenta.
2. La cotización **no está expirada** (respeta el `valid_until`).
3. La cotización **todavía no fue usada** en otro envío.
4. La cotización fue creada en la **v2** y será usada en `POST /v2/shipping`.
5. No es una cotización `simple` de `package`/`pallet`.
6. El transportista informado está en la lista retornada por la cotización.

### Restricciones de carga
Valen las mismas restricciones de la v1 (dimensiones, peso por bulto, peso total, límite de artículos,
disponibilidad de contrato, ruta bloqueada, HS Code sin tabla de impuestos).
Consulta la sección **Restricciones de carga** en [Cotización (v1)](api-cotacao-v1.md#restricciones-de-carga).

### Sin endpoint de consulta
La v2 **no posee** `GET /v2/quotation/{quotation}`. El resultado completo se entrega en la respuesta del `POST`,
por lo tanto **almacena la respuesta** si necesitas volver a presentarla al usuario.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Regla de negocio (restricción de carga, ruta, contrato, HS Code) | Leer `message` y ajustar el escenario |
| `403` | Sin permiso de cotización | Solicitar el permiso al soporte |
| `406` | Estructura del payload incompatible | Revisar tipos y anidamiento |
| `422` | Validación de campos | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- La v2 cambia precisión por simplicidad: la dirección completa entra solo en el envío.
- Los flujos operativos de **status, documentos, cancelación y programación de recogida** son los mismos
  endpoints `/v1/...`, incluso para envíos creados en la v2 — consulta
  [Seguimiento](api-rastreio.md), [Documentos](api-documentos.md) y [Programación](api-agendamento.md).
- Guarda la respuesta del `POST`: no hay endpoint de reconsulta en la v2.
