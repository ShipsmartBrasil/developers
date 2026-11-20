# Cotización (v2)

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Mismo cálculo que la [Cotización v1](api-cotacao-v1.md), con un **contrato de dirección mucho más reducido**.
- Usa la v2 cuando, en el momento de la cotización, todavía **no tienes** la ciudad ni el código postal — solo el país
  (y opcionalmente el estado).
- La ciudad, el código postal y la dirección completa se informan y validan después, en el [Envío V2](api-remessa-v2.md).
- Atiende los escenarios de carga de la v1 **y** la **cotización por artículos** (`type: items`), **exclusiva de la v2**: envías
  solo el `sku` de los artículos registrados, sin informar cajas — consulta [Escenarios de cotización](#escenarios-de-cotización).

> **Importante:** una cotización creada en la v2 solo puede generar un envío en la v2 (`POST /v2/shipping`).
> Consulta el [comparativo](api-comparativo-cotacao-remessa.md) para elegir la versión.

## URL base
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
1. Elige el [escenario de cotización](#escenarios-de-cotización) que corresponde a tu carga (`object` + `type`).
2. `POST /v2/quotation` informando solo el país (y el estado, si lo sabes) en el origen y en el destino.
3. La respuesta ya trae el resultado calculado (`data.carriers`) y el código `data.quotation`. **Guarda la respuesta**:
   no hay endpoint de reconsulta en la v2.
4. Verifica si la cotización es [elegible para envío](#cotizaciones-no-elegibles-para-envío).
5. Elige un transportista y crea el [Envío V2](api-remessa-v2.md) con la dirección completa,
   **antes** del `valid_until`.

```
POST /v2/quotation  ──►  quotation + carriers[]
   (solo país/estado)          │
                               ▼
                    POST /v2/shipping
              (dirección completa + carrier_code|slug)
```

## Diferencias con respecto a la v1

| Tema | v1 | v2 |
|---|---|---|
| Dirección en la cotización | `country_code` + `city` + (`postal_code` o `state_code`/`state`) | Solo `country_code` (+ `state_code` opcional) |
| Cotización por artículos (`type: items`) | **No disponible**: las cajas siempre se informan | **Disponible**: solo el `sku` de los artículos registrados, sin cajas |
| Validación de la dirección | En la cotización (código postal, ciudad, estado) | En el envío, cuando se envía la dirección completa |
| Dirección en el envío | Calle, nombre, e‑mail, teléfono, tipo | Calle, nombre, e‑mail, teléfono, tipo **+ ciudad, estado y código postal** |
| Consulta del resultado | `GET /v1/quotation/{quotation}/{level}` | **No existe.** El resultado viene en el propio `POST` |
| `ior` (Importer of Record) | Soportado | **No forma parte del contrato de la v2** (ignorado) |
| `items[].selling_unit_value` | Soportado | **No forma parte del contrato de la v2** (ignorado) |
| Precisión del cálculo | Mayor: ciudad/código postal permiten aplicar tasas regionales ya en la cotización | El valor considera país/estado; la dirección completa entra en el envío |

> Estas son las dos razones de ser de la v2: **dirección reducida** (solo país/estado) y **cotización por artículos**.
> Todo lo demás (objeto, cajas, kits, monedas, flete declarado, fabricante, warnings, predefiniciones de la cuenta) es idéntico a la v1.

## Conceptos esenciales

### `object` — qué se está enviando
| Valor | Significado | Observaciones |
|---|---|---|
| `doc` | Documento | Cotizado por volumen, informando `weight` y `price` de cada sobre/caja |
| `package` | Paquete/caja | Flujo más común |
| `pallet` | Pallet | Acepta dimensiones (LAA) **o** `volume` en m³/ft³ |
| `not_doc` | Legado | Se convierte internamente en `package`. **No lo uses en integraciones nuevas** |

### `type` — cómo se arma el cálculo
| Valor | Lo que envías | Cuándo usar |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` y `price` | Documentos y simulaciones rápidas. **No es elegible para envío cuando `object` es `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensiones) + `items[]` con `box_id` | Flujo estándar para envío |
| `items` | `items[]` con `sku` (y, según la agrupación de la cuenta, `boxes[]` candidatas) | **Exclusivo de la v2.** Cuando los artículos ya están registrados y la plataforma arma las cajas |

### Matriz de elegibilidad

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Cotización y envío | No recomendado | No recomendado |
| `package` | ⚠️ Solo simulación (no elegible para envío) | ✅ Cotización y envío | ✅ Cotización y envío |
| `pallet` | ⚠️ Solo simulación (no elegible para envío) | ✅ Cotización y envío | ✅ Cotización y envío |

### `tax` — quién paga los impuestos
| Valor | Significado |
|---|---|
| `sender` | DDP: el remitente paga impuestos y tasas en el destino |
| `receiver` | DDU: el destinatario paga impuestos y tasas en el destino |

> Si tu cuenta tiene una **predefinición de impuesto** configurada, esta **sustituye** el valor enviado en `tax`.
> Consulta [Predefiniciones aplicadas en la cotización](api-cotacao-v1.md#predefiniciones-aplicadas-en-la-cotización).

### `measurement` — sistema de medida
| Valor | Dimensiones | Peso | Volumen |
|---|---|---|---|
| `metric` (predeterminado) | centímetros | kilogramos | m³ |
| `imperial` | pulgadas | libras | ft³ |

### `level` — nivel de detalle del resultado
| Valor | Retorno |
|---|---|
| `simple` (predeterminado) | Lista reducida de transportistas con totales |
| `detailed` | Incluye el tipo de cambio y la composición detallada de flete, seguro e impuesto, con las reglas de subsidio aplicadas |

Se envía como **query string**: `POST /v2/quotation?level=detailed`.

## Endpoint: crear cotización

- **Método y ruta:** `POST /v2/quotation`
- **Query string opcional:** `level=simple|detailed`
- **Cuándo usar:** cuando, en el momento de cotizar, solo conoces el país (y tal vez el estado) de origen y destino.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Valores aceptados | Descripción |
|---|---|---|---|---|---|---|---|
| `object` | string | **sí** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Qué se está enviando |
| `type` | string | **sí** | - | - | - | `simple`, `advanced`, `items` | Cómo se arma el cálculo (`items` solo existe en la v2) |
| `tax` | string | **sí** | - | - | - | `sender`, `receiver` | Quién paga los impuestos (puede ser sustituido por la predefinición de la cuenta) |
| `insurance` | boolean | **sí** | - | - | - | `true`, `false` | Contratar seguro (puede activarse automáticamente por la cuenta) |
| `currency_quote` | string | **sí** | 3 | 3 | - | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda en la que se presenta la cotización y en la que se informan `unit_value`/`price` |
| `currency_payment` | string | **sí** | 3 | 3 | - | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda en la que se realizará el pago |
| `address_sender` | object | **sí** | - | - | - | - | Origen (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | - | Destino (ver tabla abajo) |
| `boxes` | array | condicional | - | - | - | - | Obligatorio cuando `type` **no** es `items`. En `type: items` con agrupación `smartbox`, también es obligatorio |
| `items` | array | condicional | - | - | - | - | Obligatorio cuando `type` **no** es `simple` y ninguna caja usa `kit` |
| `measurement` | string | no | - | - | preset de la cuenta o `metric` | `metric`, `imperial` | Sistema de medida |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Entrega en dirección residencial |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Bultos no apilables |
| `description` | string | no | - | 155 | `null` | - | Descripción libre de la cotización |
| `channel_code` | integer | no | - | - | preset de la cuenta | - | Código del canal de venta registrado y habilitado para tu usuario |
| `channel_identifier_number` | string | no | - | - | `null` | - | Tu identificador comercial (n.º de pedido). **Recomendado** |
| `channel_order_number` | string | no | - | 100 | `null` | - | Número del pedido en el canal de venta |
| `declared_freight` | object | no | - | - | preset de la cuenta | - | Flete declarado. Exige un permiso específico |
| `manufacturer` | object | no | - | - | `null` | - | Fabricante mostrado en la Commercial Invoice |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Nivel de detalle del resultado |

### Parámetros — `address_sender` y `address_receiver`

**Esta es la principal diferencia de la v2.** Elige una de las dos formas y no las mezcles:

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de una dirección registrada en el panel de tu cuenta |

**Forma B — solo el país (y el estado, si lo sabes):**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `country_code` | string | **sí** | 2 | 2 | País en ISO 3166-1 alpha-2 (ej.: `BR`, `US`) |
| `state_code` | string | no | - | - | Sigla del estado/provincia (ej.: `SP`, `FL`). Cuando se informa, mejora la precisión del cálculo de impuestos regionales y se reutiliza en el envío |

> No envíes `city`, `postal_code` ni `state` en la cotización v2: estos campos se **ignoran** y deben
> informarse en el [Envío V2](api-remessa-v2.md).

> **Remitente registrado obligatorio en algunos tipos de cuenta:** las cuentas de persona física y las cuentas empresariales
> con un único centro de distribución deben informar el remitente mediante la **Forma A** (`address_code`).
> Sin él, la API responde `406` con `Codigo de endereço do remetente não encontrado!`.

### Parámetros — `boxes[]`

Idéntico a la v1. Hay **tres formas** de describir el bulto:

**Forma A — kit registrado** (completa la caja **y** los artículos automáticamente):

| Campo | Tipo | Obligatorio | Máx. | Descripción |
|---|---|---|---|---|
| `kit` | string | **sí** en esta forma | 50 | SKU del kit registrado. La caja y los artículos de esa posición se completan a partir del registro del kit |

**Forma B — caja registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `box_code` | integer | **sí** en esta forma | Código de la caja registrada ([Caja](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa al `box_code` | Identificador de la caja registrada (ej.: `CX0236`) |

**Forma C — caja informada manualmente:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `name` | string | no | 4 | 60 | Nombre/etiqueta del bulto |
| `height` | number | **sí** (excepto pallet con `volume`) | 1 | - | Alto |
| `width` | number | **sí** (excepto pallet con `volume`) | 1 | - | Ancho |
| `depth` | number | **sí** (excepto pallet con `volume`) | 1 | - | Profundidad (largo) |
| `volume` | number | condicional | 0.001 | - | **Solo cuando `object` es `pallet`.** Alternativa a alto/ancho/profundidad, en m³ o ft³ |

**Campos válidos en las formas B y C:**

| Campo | Tipo | Obligatorio | Mín. | Descripción |
|---|---|---|---|---|
| `weight` | number | **sí** cuando `object` es `doc`/`package` **y** `type` es `simple` | > 0 | Peso del bulto. En `advanced`/`items`, cuando se omite, es la suma de los artículos |
| `price` | number | **sí** cuando `object` es `doc`/`package` **y** `type` es `simple` | > 0 | Valor declarado del bulto. En `advanced`/`items`, cuando se omite, es la suma de los artículos |

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `quantity` | integer | **sí** | 1 | 1000 | Cantidad del artículo |
| `unit_value` | number | **sí** | > 0 | - | Valor unitario en la moneda de la cotización. Cero o negativo se rechaza |
| `box_id` | integer | **sí**, excepto cuando `type` es `items` | 1 | n.º de cajas | **Posición** de la caja en el array `boxes` (1 = primera caja). No es el `box_code` |
| `sku` | string | **sí** cuando `type` es `items` | - | 50 | SKU del artículo registrado ([Artículos](api-itens-v1.md)). Los campos omitidos se completan a partir del registro |
| `ean` | string | no | - | 50 | Código EAN del artículo |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: identificador del producto en el fabricante. Se imprime en la invoice y se usa en la información regulatoria exigida por algunos transportistas para destinos en la Unión Europea |
| `name` | string | **sí** cuando no se informa `sku` | 6 | 50 | Nombre del artículo (aparece en la invoice) |
| `description` | string | no | - | 155 | Descripción detallada del artículo |
| `country_code` | string | **sí** cuando no se informa `sku` | 2 | 2 | País de origen/fabricación del artículo |
| `hscode` | string | **sí** cuando no se informa `sku` | 6 | 10 | NCM/HS Code. Define la tabla de impuestos aplicada |
| `weight` | number | **sí** cuando no se informa `sku` | 0.001 | - | Peso unitario del artículo |
| `is_groupable` | boolean | no | - | - | Predeterminado `true`. Cuando es `false`, el artículo recibe una caja exclusiva en las agrupaciones de `type: items` |

### Parámetros — `declared_freight`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valor del flete declarado |
| `currency` | string | **sí** cuando se informa `value` | 3 | 3 | Moneda del flete declarado |

> Se aplica solo si tu perfil tiene el **permiso de flete declarado**; de lo contrario, se ignora en silencio.

### Parámetros — `manufacturer`

Idéntico a la v1: envía `address_code` **o** los campos manuales (`name`, `country_code`, `state_code`, `state`,
`postal_code`, `city`, `street`, `number`, `district`, `complement`, `email`, `phone`, `phone_extension`, `type`,
`federal_tax_id`, `state_tax_id`). Límites y recomendaciones en la
[tabla del fabricante de la v1](api-cotacao-v1.md#parámetros--manufacturer).

| Campo | Tipo | Recomendado | Descripción |
|---|---|---|---|
| `address_code` | integer | alternativa a los manuales | Dirección registrada del fabricante |
| `name` | string | **sí** sin `address_code` | Razón social/nombre del fabricante (de 5 a 30 caracteres) |

### Payload mínimo

El menor payload aceptado por el endpoint (cotización de documento):

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    {
      "height": 30,
      "width": 25,
      "depth": 2,
      "weight": 0.5,
      "price": 10
    }
  ]
}
```

### Payload máximo

Todos los campos aceptados por el endpoint, con las tres formas de caja:

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
    { "name": "Caja master", "height": 20, "width": 20, "depth": 20, "weight": 2.5, "price": 150 },
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta 100% algodon, manga corta, color blanco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Zapatillas deportivas",
      "description": "Zapatillas de running para hombre",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    },
    {
      "box_id": 3,
      "sku": "CNC-010-M",
      "quantity": 2,
      "unit_value": 15
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
    "channel": "Marketplace Ejemplo",
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
        "url_image": "https://files.example.com/contratos/dhl.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "real", "weight": "5" },
        "freight": "181.95",
        "freight_final": "181.95",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.45",
        "insurance_final": "11.45",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "28153.894",
        "tax_final": "28153.894",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "28347.29",
        "currency_quote_amount": "28347.29",
        "currency_payment_amount": "24288.10"
      },
      {
        "code": 2,
        "slug": "fedex-mx",
        "name": "FedEx",
        "url_image": "https://files.example.com/contratos/fedex.png",
        "valid_until": "2026-02-21 20:28:10",
        "transit_days": 5,
        "weight_details": { "type": "cubed", "weight": "5" },
        "freight": "48.05",
        "freight_final": "48.05",
        "freight_subsidy_rule_code": null,
        "freight_details": null,
        "insurance": "11.55",
        "insurance_final": "11.55",
        "insurance_subsidy_rule_code": null,
        "insurance_details": null,
        "tax": "24924.09",
        "tax_final": "24924.09",
        "tax_subsidy_rule_code": null,
        "tax_details": null,
        "currency_quote_price": "24983.69",
        "currency_quote_amount": "24983.69",
        "currency_payment_amount": "21406.15"
      }
    ]
  }
}
```

### Campos de la respuesta

Misma estructura que la v1 — descripción completa en [Campos de la respuesta de la v1](api-cotacao-v1.md#campos-de-la-respuesta).
Los campos más usados:

| Campo | Tipo | Descripción |
|---|---|---|
| `data.quotation` | string (UUID) | **Código de la cotización.** Úsalo en `POST /v2/shipping` |
| `data.finished` | boolean | `true` cuando la cotización **ya se usó** para crear un envío |
| `data.warnings` | array | **Opcional.** Solo aparece cuando hay alertas aduaneras |
| `carriers[].code` | integer | Código del contrato. Úsalo en `carrier_code` en el envío |
| `carriers[].slug` | string | Identificador textual del contrato. Alternativa al `carrier_code` |
| `carriers[].valid_until` | string | **Validez de la cotización.** Crea el envío antes de esta fecha/hora |
| `carriers[].transit_days` | integer | Plazo estimado de tránsito en días |
| `carriers[].weight_details.type` | string | `real` o `cubed` — qué peso se usó en la tarificación |
| `carriers[].freight` / `freight_final` | string | Flete antes / después de las tasas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / después de las tasas administrativas |
| `carriers[].tax` / `tax_final` | string | Impuestos antes / después de las tasas administrativas |
| `carriers[].currency_quote_amount` | string | Total a pagar en la moneda de la cotización |
| `carriers[].currency_payment_amount` | string | Total a pagar en la moneda de pago |

## Escenarios de cotización

Cada escenario a continuación muestra lo que cambia en el payload, el payload mínimo y el payload máximo **de ese escenario**.
Los campos raíz comunes (monedas, impuesto, seguro, canal, flete declarado, etc.) siguen la
[tabla de la raíz del payload](#parámetros--raíz-del-payload).

| Escenario | `object` | `type` | `boxes[]` | `items[]` | Elegible para envío |
|---|---|---|---|---|---|
| [Documento](#cotización-de-documento) | `doc` | `simple` | Dimensiones + `weight` + `price` | No usa | ✅ |
| [Paquete — simple](#cotización-de-paquete-caja--simple) | `package` | `simple` | Dimensiones + `weight` + `price` | No usa | ❌ solo simulación |
| [Paquete — avanzado](#cotización-de-paquete-caja--avanzada) | `package` | `advanced` | Dimensiones (peso/valor opcionales) | Obligatorio, con `box_id` | ✅ |
| [Pallet — por LAA](#cotización-de-pallet--por-laa) | `pallet` | `advanced` | `height` + `width` + `depth` | Obligatorio, con `box_id` | ✅ |
| [Pallet — por m³](#cotización-de-pallet--por-m³) | `pallet` | `advanced` | `volume` | Obligatorio, con `box_id` | ✅ |
| [Artículos](#cotización-de-artículos) — **exclusivo de la v2** | `package` | `items` | Depende de la agrupación de la cuenta | Obligatorio, con `sku` | ✅ |

### Cotización de Documento

- **Cuándo usar:** envío de papeles/documentos sin valor comercial relevante (contratos, certificados, impresos).
- **Cómo armar:** `object: doc` + `type: simple`. Cada bulto (sobre/caja) informa dimensiones, `weight` y `price`.
- **No usa** `items[]` ni `manufacturer`: los documentos no pasan por el cálculo de impuestos por artículo.
- **Es elegible para envío.**

#### Payload mínimo

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0002",
  "channel_order_number": "334456",
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Envio de contratos firmados",
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "declared_freight": { "value": 20, "currency": "BRL" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Sobre A4", "height": 30, "width": 25, "depth": 2, "weight": 0.5, "price": 10 },
    { "box_code": 7, "weight": 1.2, "price": 15 }
  ]
}
```

> En `doc`, `weight` y `price` son obligatorios **en todo bulto**, incluso cuando la caja está registrada (`box_code`/`box_identifier`).

### Cotización de Paquete (Caja) — simple

- **Cuándo usar:** simulación rápida de flete cuando solo conoces el peso y el valor de cada caja.
- **Cómo armar:** `object: package` + `type: simple`. Cada caja informa dimensiones, `weight` y `price`. Sin artículos.
- ⚠️ **No es elegible para envío.** Para enviar de hecho, rehaz la cotización como
  [Paquete — avanzado](#cotización-de-paquete-caja--avanzada) o [Artículos](#cotización-de-artículos).

#### Payload mínimo

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "SIM-2026-0001",
  "channel_order_number": "334457",
  "object": "package",
  "type": "simple",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": true,
  "description": "Simulacion de flete para la vitrina",
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "declared_freight": { "value": 30, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Caja mediana", "height": 20, "width": 20, "depth": 20, "weight": 2, "price": 100 },
    { "box_identifier": "CX0236", "weight": 3.5, "price": 180 },
    { "box_code": 3, "weight": 1, "price": 60 },
    { "kit": "KIT-001" }
  ]
}
```

> En la caja de `kit`, el peso y las dimensiones provienen del registro de la caja del kit y el `price` es la suma de los valores de los artículos del kit.
> Si la caja registrada tiene **peso máximo** configurado, el `weight` informado no puede superarlo.

### Cotización de Paquete (Caja) — avanzada

- **Cuándo usar:** es el **flujo estándar para envío** de mercancías. Declaras los artículos e indicas en qué caja
  está cada uno.
- **Cómo armar:** `object: package` + `type: advanced`. Las cajas traen las dimensiones; cada artículo apunta a la
  posición de su caja en `box_id` (1 = primera caja).
- El peso y el valor de cada caja se calculan sumando los artículos, si no se informan.
- **Toda caja necesita al menos un artículo**; un `box_id` fuera del rango de cajas se rechaza.
- **Es elegible para envío.**

#### Payload mínimo

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
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

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0003",
  "channel_order_number": "334458",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Pedido con dos cajas",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Caja ropa", "height": 30, "width": 25, "depth": 15, "weight": 1.2, "price": 120 },
    { "box_code": 3 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta 100% algodon, manga corta, color blanco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 4,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "box_id": 2,
      "name": "Zapatillas deportivas",
      "description": "Zapatillas de running para hombre",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9
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

> Un artículo con `sku` registrado puede enviarse solo con `box_id`, `sku`, `quantity` y `unit_value`: el resto
> (nombre, país, HS Code, peso, EAN, `nsmp`) se completa a partir del [registro de artículos](api-itens-v1.md).
> Lo que envíes en el payload tiene prioridad sobre el registro.

### Cotización de Pallet — por LAA

- **Cuándo usar:** carga paletizada cuyas medidas conoces — **L**argo (`depth`), **A**ncho (`width`)
  y **A**lto (`height`).
- **Cómo armar:** `object: pallet` + `type: advanced`, con `height`, `width` y `depth` en cada pallet y los artículos
  apuntando al pallet en `box_id`.
- La plataforma calcula el volumen (`L × A × A ÷ 1.000.000` = m³) a partir de las medidas.
- El peso del pallet, si se omite, es la suma de los pesos de los artículos. Informa `weight` para incluir el peso de la base/embalaje.
- `type: simple` se acepta para simulación, pero **no es elegible para envío**.

#### Payload mínimo

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "height": 150, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Pieza de repuesto industrial",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0001",
  "channel_order_number": "990011",
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Dos pallets de piezas industriales",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Pallet 01", "height": 150, "width": 100, "depth": 120, "weight": 280, "price": 900 },
    { "name": "Pallet 02", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Pieza de repuesto industrial",
      "description": "Engranaje de acero para maquina de embalaje",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Rodamiento industrial",
      "description": "Rodamiento de bolas de acero",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

### Cotización de Pallet — por m³

- **Cuándo usar:** carga paletizada de la que solo conoces el **volumen** (cubicaje), sin las medidas individuales.
- **Cómo armar:** `object: pallet` + `type: advanced`, con `volume` en cada pallet (m³ en `metric`, ft³ en `imperial`).
- La plataforma convierte el volumen en un cubo equivalente para la tarificación:
  `lado (cm) = ∛volumen (m³) × 100`. Ej.: `1.0` m³ → 100 × 100 × 100 cm; `1.8` m³ → 121,64 × 121,64 × 121,64 cm.
- Si envías `volume` **y** las tres medidas, las medidas (LAA) tienen prioridad.
- Puedes mezclar pallets por LAA y por m³ en el mismo payload (cada posición de `boxes[]` decide su forma).

#### Payload mínimo

```json
{
  "object": "pallet",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "volume": 1.8 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Pieza de repuesto industrial",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    }
  ]
}
```

#### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PLT-2026-0002",
  "channel_order_number": "990012",
  "object": "pallet",
  "type": "advanced",
  "tax": "receiver",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": true,
  "description": "Pallet por cubicaje y pallet por medidas",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "name": "Pallet cubicaje", "volume": 1.8, "weight": 300, "price": 900 },
    { "name": "Pallet medido", "height": 110, "width": 100, "depth": 120 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "PEC-0001",
      "ean": "7894900020014",
      "nsmp": "MFR-PEC-0001",
      "name": "Pieza de repuesto industrial",
      "description": "Engranaje de acero para maquina de embalaje",
      "country_code": "BR",
      "hscode": "848790",
      "weight": 12.5,
      "quantity": 20,
      "unit_value": 45
    },
    {
      "box_id": 2,
      "name": "Rodamiento industrial",
      "description": "Rodamiento de bolas de acero",
      "country_code": "BR",
      "hscode": "848210",
      "weight": 3,
      "quantity": 50,
      "unit_value": 12
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Un pallet sin medidas **y** sin `volume` se rechaza con `422` (validación) o con
> `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³`.

### Cotización de Artículos

- **Exclusiva de la v2.** Es uno de los motivos para elegir la v2: el payload no necesita describir el paquete (caja).
- **Cuándo usar:** ya tienes los productos en el [registro de artículos](api-itens-v1.md) y quieres que la plataforma
  arme las cajas por sí sola.
- **Cómo armar:** `object: package` + `type: items`. Cada artículo informa `sku`, `quantity` y `unit_value`; **no**
  informes `box_id`.
- La forma en que se arman las cajas depende de la **agrupación** configurada en tu cuenta
  ([Primeros Pasos](api-primeiros-passos.md#agrupación-de-cajas)):

| Agrupación | `boxes[]` en el payload | Requisito previo en el registro del artículo | Cómo se arman las cajas |
|---|---|---|---|
| `default` | No se usa | Artículo vinculado a una caja registrada (`box_code` o `box_identifier`) | Los artículos agrupables que usan la misma caja se reúnen; los artículos no agrupables reciben una caja por unidad |
| `all_in_one` | No se usa | Artículo con dimensiones registradas | Los artículos agrupables van a **una** caja virtual con el volumen total; los no agrupables reciben una caja por unidad |
| `smartbox` | **Obligatorio** (cajas candidatas) | Artículo con dimensiones registradas | Los artículos se encajan en las cajas enviadas, considerando volumen y dimensiones con 10% de holgura e `is_groupable`/`is_foldable` |

#### Payload mínimo

Agrupación `default` o `all_in_one`:

```json
{
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "items": [
    { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
  ]
}
```

#### Payload máximo

Agrupación `smartbox` (cajas candidatas informadas):

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0004",
  "channel_order_number": "334459",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": true,
  "non_stackable": false,
  "description": "Pedido armado por el registro de articulos",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 25, "currency": "USD" },
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [
    { "box_identifier": "CX0236" },
    { "box_code": 3 },
    { "name": "Caja grande", "height": 40, "width": 40, "depth": 40 }
  ],
  "items": [
    {
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta 100% algodon, manga corta, color blanco",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "is_groupable": true
    },
    {
      "sku": "TEN-042-M",
      "quantity": 1,
      "unit_value": 199.9,
      "is_groupable": false
    }
  ],
  "manufacturer": {
    "address_code": 15
  }
}
```

> Errores típicos de este escenario: `O item com o sku :name não existe!` (`404`),
> `A caixa com o box_code/box_identifier :name não existe ou está desativada` (`404`),
> `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (`400`) y
> `Nenhuma caixa comporta o item :name (Smart Box).` (`400`).

### Cotización con Kit

Un **kit** es un conjunto prerregistrado en el panel (una caja + artículos con cantidades). Al informar
`boxes[].kit`, esa posición se sustituye por la caja del kit y los artículos del kit se incluyen automáticamente.

- Puede combinarse con cajas comunes en el mismo payload.
- Si **todas** las cajas son kits, `items[]` puede omitirse.
- En `type: simple`, el kit se convierte en una caja con `price` = suma de los artículos del kit; en `advanced`, los artículos del kit
  entran en la cotización apuntando a la posición del kit.

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR" },
  "address_receiver": { "country_code": "US" },
  "boxes": [
    { "kit": "KIT-001" }
  ]
}
```

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `O kit :sku é inválido!` | Kit inexistente, inactivo o de otra cuenta |
| `400` | `A caixa do kit :sku é inválida!` | La caja vinculada al kit no existe o fue desactivada |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Algún artículo del kit fue eliminado del registro |

## Cotizaciones no elegibles para envío

Las mismas condiciones de la v1 valen para la v2, y se comprueban en el `POST /v2/shipping` en este orden:

| # | Condición | Status | Mensaje cuando falla |
|---|---|---|---|
| 1 | La cotización existe y fue creada en la **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La cotización pertenece a tu cuenta | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Cuenta con impuesto bloqueado en DDP: cotización DDU (`tax: receiver`) sin pago aprobado | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | La cotización **todavía no se ha usado** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | No es `type: simple` con `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | La cotización **no ha expirado** (`valid_until` del transportista) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | El transportista elegido está en `carriers[]` de esta cotización | `404` | `Contrato especificado não foi listado para esta cotação!` |

> En la v2 la dirección completa solo se valida en el envío. Un código postal o una dirección inválidos informados en el
> [Envío V2](api-remessa-v2.md#validación-de-la-dirección-completa) también impiden la creación.

## Reglas de negocio

### Predefiniciones y restricciones de carga
Valen las mismas reglas de la v1:
[Predefiniciones aplicadas en la cotización](api-cotacao-v1.md#predefiniciones-aplicadas-en-la-cotización),
[Restricciones de carga](api-cotacao-v1.md#restricciones-de-carga),
[Alertas de la cotización](api-cotacao-v1.md#alertas-de-la-cotización-warnings) y
[Registros referenciados](api-cotacao-v1.md#registros-referenciados).

### Dirección sin validación en la cotización
- En la v2, la cotización **no** valida código postal, ciudad ni estado; solo el país (y el `state_code`, cuando se envía).
- No se exige código postal para direcciones en Brasil en la cotización v2.
- El `state_code` informado en la cotización se reutiliza en el envío v2.

### Sin endpoint de consulta
La v2 **no tiene** `GET /v2/quotation/{quotation}`. El resultado completo se entrega en la respuesta del `POST`,
por lo tanto **almacena la respuesta** si necesitas volver a mostrarla al usuario.

## Errores comunes

| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Regla de negocio (carga, caja, artículo, kit) | Leer `message` y ajustar el escenario |
| `403` | Sin permiso de cotización o canal de otra cuenta | Solicitar el permiso al soporte / revisar `channel_code` |
| `404` | Artículo/caja no encontrado en la agrupación o ningún contrato elegible | Revisar registros y escenario |
| `406` | La carga exige atención personalizada, o remitente registrado obligatorio para el tipo de cuenta | Hablar con el equipo comercial / usar `address_sender.address_code` |
| `422` | Validación de campos o HS Code sin tabla de impuesto | Corregir los campos listados en `data` / revisar `hscode` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- La v2 cambia precisión por simplicidad: la dirección completa entra solo en el envío.
- Los flujos operativos de **status, documentos, cancelación y programación de recogida** usan los mismos
  endpoints `/v1/...`, incluso para envíos creados en la v2 — consulta
  [Seguimiento](api-rastreio.md), [Documentos](api-documentos.md) y [Programación](api-agendamento.md).
- Guarda la respuesta del `POST`: no hay endpoint de reconsulta en la v2.
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
