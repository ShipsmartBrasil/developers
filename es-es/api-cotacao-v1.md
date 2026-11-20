# Cotización (v1)

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Calcula flete, seguro e impuestos para un envío internacional, en todos los transportistas habilitados en tu cuenta.
- Es el **primer paso obligatorio** del flujo: el envío, el seguimiento, los documentos y la programación de recogida
  están todos vinculados al código de cotización (`quotation`).
- La v1 es la versión completa: recibe la dirección (país, ciudad, código postal/estado) ya en la cotización, valida esa dirección
  y dispone de un endpoint de reconsulta del resultado.
- Atiende cinco escenarios de carga: documento, paquete simple, paquete avanzado, pallet por LAA y pallet por m³ —
  consulta [Escenarios de cotización](#escenarios-de-cotización). La cotización **por artículos** (solo `sku`, sin cajas) es exclusiva de la
  [Cotización (v2)](api-cotacao-v2.md#cotización-de-artículos).

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
2. `POST /v1/quotation` con origen, destino, objeto, tipo y cajas/artículos.
3. La respuesta ya trae el resultado calculado (`data.carriers`) y el código `data.quotation`.
4. Si necesitas reconsultar (o cambiar el nivel de detalle), usa `GET /v1/quotation/{quotation}/{level}`.
5. Verifica si la cotización es [elegible para envío](#cotizaciones-no-elegibles-para-envío).
6. Elige un transportista de la lista y usa `code` (o `slug`) + `quotation` para crear el
   [envío](api-remessa-v1.md) **antes** del `valid_until`.

```
POST /v1/quotation  ──►  quotation + carriers[]
                              │
                              ▼
                   POST /v1/shipping  (carrier_code | slug)
```

## Conceptos esenciales

### `object` — qué se está enviando
| Valor | Significado | Observaciones |
|---|---|---|
| `doc` | Documento | Cotizado por volumen, informando `weight` y `price` de cada sobre/caja |
| `package` | Paquete/caja | Flujo más común |
| `pallet` | Pallet | Acepta dimensiones (LAA) **o** `volume` en m³/ft³ |
| `not_doc` | Legado | Se mantiene por compatibilidad; se convierte internamente en `package`. **No lo uses en integraciones nuevas** |

### `type` — cómo se arma el cálculo
| Valor | Lo que envías | Cuándo usar |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` y `price` | Documentos y simulaciones rápidas de flete. **No es elegible para envío cuando `object` es `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensiones) + `items[]` con `box_id` apuntando a la posición de la caja | Flujo estándar para envío: tú controlas la distribución de los artículos en las cajas |
| `items` | **No disponible en la v1** | Exclusivo de la [Cotización (v2)](api-cotacao-v2.md#cotización-de-artículos) |

### Matriz de elegibilidad

| `object` \ `type` | `simple` | `advanced` | `items` |
|---|---|---|---|
| `doc` | ✅ Cotización y envío | No recomendado | ❌ Solo en la v2 |
| `package` | ⚠️ Solo simulación (no elegible para envío) | ✅ Cotización y envío | ❌ Solo en la v2 |
| `pallet` | ⚠️ Solo simulación (no elegible para envío) | ✅ Cotización y envío | ❌ Solo en la v2 |

> **Atención:** `type: simple` con `object: package` u `object: pallet` sirve solo para simulación.
> Al intentar crear el envío, la API responde `403` con `Cotação Simples é inelegível para remessa!`.
> Detalles en [Cotizaciones no elegibles para envío](#cotizaciones-no-elegibles-para-envío).

### `tax` — quién paga los impuestos
| Valor | Significado |
|---|---|
| `sender` | DDP: el remitente paga impuestos y tasas en el destino |
| `receiver` | DDU: el destinatario paga impuestos y tasas en el destino |

> Si tu cuenta tiene una **predefinición de impuesto** configurada, esta **sustituye** el valor enviado en `tax`.
> Consulta [Predefiniciones aplicadas en la cotización](#predefiniciones-aplicadas-en-la-cotización).

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

Se envía como **query string** en el `POST` y como **segmento de path** en el `GET`.

## Endpoint: crear cotización

- **Método y ruta:** `POST /v1/quotation`
- **Query string opcional:** `level=simple|detailed`
- **Cuándo usar:** siempre que necesites precio, plazo e impuestos, y antes de crear cualquier envío.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Valores aceptados | Descripción |
|---|---|---|---|---|---|---|---|
| `object` | string | **sí** | - | - | - | `doc`, `package`, `pallet`, `not_doc` | Qué se está enviando |
| `type` | string | **sí** | - | - | - | `simple`, `advanced` | Cómo se arma el cálculo (`items` es exclusivo de la v2) |
| `tax` | string | **sí** | - | - | - | `sender`, `receiver` | Quién paga los impuestos (puede ser sustituido por la predefinición de la cuenta) |
| `insurance` | boolean | **sí** | - | - | - | `true`, `false` | Contratar seguro (puede activarse automáticamente por la cuenta) |
| `currency_quote` | string | **sí** | 3 | 3 | - | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda en la que se presenta la cotización y en la que se informan `unit_value`/`price` |
| `currency_payment` | string | **sí** | 3 | 3 | - | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda en la que se realizará el pago |
| `address_sender` | object | **sí** | - | - | - | - | Origen (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | - | Destino (ver tabla abajo) |
| `boxes` | array | **sí** | - | - | - | - | Bultos de la cotización (ver tabla abajo) |
| `items` | array | condicional | - | - | - | - | Obligatorio cuando `type` **no** es `simple` y ninguna caja usa `kit` |
| `measurement` | string | no | - | - | preset de la cuenta o `metric` | `metric`, `imperial` | Sistema de medida de dimensiones, pesos y volúmenes |
| `ior` | boolean | no | - | - | `false` | `true`, `false` | Importer of Record. Cuando es `true`, se pasa a considerar `items[].selling_unit_value` |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Entrega en dirección residencial |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Bultos no apilables |
| `description` | string | no | - | 155 | `null` | - | Descripción libre de la cotización, para tu referencia |
| `channel_code` | integer | no | - | - | preset de la cuenta | - | Código del canal de venta registrado y habilitado para tu usuario |
| `channel_identifier_number` | string | no | - | - | `null` | - | Tu identificador comercial (n.º de pedido). **Recomendado**: permite consultar status/documentos y cancelar por canal |
| `channel_order_number` | string | no | - | 100 | `null` | - | Número del pedido en el canal de venta |
| `declared_freight` | object | no | - | - | preset de la cuenta | - | Flete declarado (ver tabla abajo). Exige un permiso específico |
| `manufacturer` | object | no | - | - | `null` | - | Fabricante mostrado en la Commercial Invoice (ver tabla abajo) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Nivel de detalle del resultado |

### Parámetros — `address_sender` y `address_receiver`

Hay **dos formas** de informar la dirección. Elige una y no las mezcles:

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de una dirección registrada en el panel de tu cuenta. Cuando se usa, no envíes los demás campos |

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `country_code` | string | **sí** | 2 | 2 | País en ISO 3166-1 alpha-2 (ej.: `BR`, `US`) |
| `city` | string | **sí** | - | 100 | Ciudad |
| `postal_code` | string | condicional | - | 12 | Código postal/ZIP. Obligatorio si no se informa `state`. **Para direcciones en Brasil es siempre obligatorio** |
| `state` | string | condicional | - | - | Nombre del estado/provincia. Obligatorio si no se informan `state_code` ni `postal_code` |
| `state_code` | string | condicional | - | - | Sigla del estado/provincia (ej.: `SP`, `FL`). Obligatorio si no se informan `state` ni `postal_code` |

> **Regla práctica:** informa siempre `country_code` + `city` + `postal_code` (+ `state_code`, cuando lo sepas).
> Esta combinación atiende a todos los países y es la que genera el cálculo más preciso.

> **Remitente registrado obligatorio en algunos tipos de cuenta:** las cuentas de persona física y las cuentas empresariales
> con un único centro de distribución deben informar el remitente mediante la **Forma A** (`address_code`).
> Sin él, la API responde `406` con `Codigo de endereço do remetente não encontrado!`.

### Parámetros — `boxes[]`

Cada elemento del array representa un bulto (caja, sobre o pallet). Hay **tres formas** de describir el bulto:

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
| `name` | string | no | 4 | 60 | Nombre/etiqueta del bulto (usado en los mensajes de error) |
| `height` | number | **sí** (excepto pallet con `volume`) | 1 | - | Alto |
| `width` | number | **sí** (excepto pallet con `volume`) | 1 | - | Ancho |
| `depth` | number | **sí** (excepto pallet con `volume`) | 1 | - | Profundidad (largo) |
| `volume` | number | condicional | 0.001 | - | **Solo cuando `object` es `pallet`.** Alternativa a alto/ancho/profundidad, en m³ (`metric`) o ft³ (`imperial`) |

**Campos válidos en las formas B y C:**

| Campo | Tipo | Obligatorio | Mín. | Descripción |
|---|---|---|---|---|
| `weight` | number | **sí** cuando `object` es `doc`/`package` **y** `type` es `simple` | > 0 | Peso del bulto. En `advanced`, cuando se omite, es la suma de los pesos de los artículos de esa caja |
| `price` | number | **sí** cuando `object` es `doc`/`package` **y** `type` es `simple` | > 0 | Valor declarado del bulto. En `advanced`, cuando se omite, es la suma de `unit_value × quantity` de los artículos de esa caja |

> `weight` y `price` se **ignoran** cuando la caja usa `kit`.

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `quantity` | integer | **sí** | 1 | 1000 | Cantidad del artículo |
| `unit_value` | number | **sí** | > 0 | - | Valor unitario en la moneda de la cotización. Cero o negativo se rechaza |
| `box_id` | integer | **sí** | 1 | n.º de cajas | **Posición** de la caja en el array `boxes` (1 = primera caja, 2 = segunda...). No es el `box_code` |
| `sku` | string | no | - | 50 | SKU del artículo registrado ([Artículos](api-itens-v1.md)). Los campos omitidos se completan a partir del registro |
| `ean` | string | no | - | 50 | Código EAN del artículo |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: identificador del producto en el fabricante. Se imprime en la invoice y se usa en la información regulatoria exigida por algunos transportistas para destinos en la Unión Europea |
| `name` | string | **sí** cuando no se informa `sku` | 6 | 50 | Nombre del artículo (aparece en la invoice) |
| `description` | string | no | - | 155 | Descripción detallada del artículo |
| `country_code` | string | **sí** cuando no se informa `sku` | 2 | 2 | País de origen/fabricación del artículo |
| `hscode` | string | **sí** cuando no se informa `sku` | 6 | 10 | NCM/HS Code del artículo. Define la tabla de impuestos aplicada |
| `weight` | number | **sí** cuando no se informa `sku` | 0.001 | - | Peso unitario del artículo |
| `selling_unit_value` | number | no | - | - | Valor de venta unitario. Se considera cuando `ior` es `true`; si se omite, usa el valor del registro del artículo y, en su ausencia, el `unit_value` |
| `is_groupable` | boolean | no | - | - | Predeterminado `true`. Se usa en las agrupaciones automáticas de cajas de la cotización por artículos (v2) |

> **Artículos con el mismo `name`** generan un `warning` en la respuesta
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> La cotización se realiza, pero conviene diferenciar los nombres para evitar retenciones en la aduana.

### Parámetros — `declared_freight`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valor del flete declarado |
| `currency` | string | **sí** cuando se informa `value` | 3 | 3 | Moneda del flete declarado. Misma lista que `currency_quote` |

> El flete declarado solo se aplica si tu perfil tiene el **permiso de flete declarado**. Sin él, el campo se
> ignora en silencio (la cotización se realiza normalmente, sin flete declarado).

### Parámetros — `manufacturer`

Datos del fabricante impresos en la Commercial Invoice. Envía `address_code` **o** los campos manuales.

| Campo | Tipo | Recomendado | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa a los manuales | - | - | Código de dirección registrada. Cuando se usa, los datos del registro sustituyen los campos manuales |
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

> La API **no rechaza** un `manufacturer` incompleto en la cotización, pero los datos se imprimen en la invoice:
> envía todos los campos marcados como recomendados para evitar problemas en el despacho aduanero.
> Para artículos con `hscode` que empieza por `64` (calzado), la aduana exige el nombre del fabricante.

### Payload mínimo

El menor payload aceptado por el endpoint (cotización de documento). Los payloads mínimos de cada tipo de carga están
en [Escenarios de cotización](#escenarios-de-cotización).

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "postal_code": "33101"
  },
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
  "ior": true,
  "description": "Pedido del marketplace, lote de octubre",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "value": 35.5,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
  "boxes": [
    {
      "name": "Caja master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 2.5,
      "price": 150
    },
    {
      "box_identifier": "CX0236"
    },
    {
      "box_code": 3
    },
    {
      "kit": "KIT-001"
    }
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
      "selling_unit_value": 49.9,
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
      "selling_unit_value": 349.9,
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

> En `type: advanced`, **toda caja necesita al menos un artículo** (excepto la caja de `kit`, que ya trae los suyos).
> Por eso el ejemplo asocia un artículo a las cajas 1, 2 y 3.

### Ejemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/quotation?level=detailed' \
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
    "address_sender": {"country_code": "BR", "city": "Diadema", "postal_code": "09941070"},
    "address_receiver": {"country_code": "US", "city": "Miami", "state_code": "FL", "postal_code": "33101"},
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
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
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
      }
    ]
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.quotation` | string (UUID) | **Código de la cotización.** Guárdalo: es la clave de todo el flujo |
| `data.finished` | boolean | `true` cuando la cotización **ya se usó** para crear un envío; `false` mientras todavía está disponible |
| `data.date` | string | Fecha/hora de la cotización (`AAAA-MM-DD HH:MM:SS`) |
| `data.description` | string | Descripción enviada en `description` |
| `data.channel` | string | Nombre del canal de venta, cuando hay `channel_code` |
| `data.channel_identifier_number` | string | Identificador comercial enviado en la cotización |
| `data.object` / `data.type` / `data.tax` / `data.insurance` | - | Valores efectivamente usados en el cálculo (después de las predefiniciones de la cuenta) |
| `data.currency_quote` / `data.currency_payment` | string | Monedas de la cotización y del pago |
| `data.currency_quote_exchange_value` | string | **Solo en `detailed`.** Tipo de cambio aplicado a la moneda de la cotización |
| `data.currency_payment_exchange_value` | string | **Solo en `detailed`.** Tipo de cambio aplicado a la moneda de pago |
| `data.warnings` | array | **Opcional.** Solo aparece cuando hay alertas (ver [Alertas de la cotización](#alertas-de-la-cotización-warnings)) |
| `data.carriers[]` | array | Opciones de envío calculadas |
| `carriers[].code` | integer | Código del contrato. Úsalo en `carrier_code` en el envío |
| `carriers[].slug` | string | Identificador textual del contrato. Alternativa al `carrier_code` en el envío |
| `carriers[].name` | string | Nombre comercial del transportista/contrato |
| `carriers[].url_image` | string | Logo del transportista |
| `carriers[].valid_until` | string | **Validez de la cotización.** El envío debe crearse antes de esta fecha/hora |
| `carriers[].transit_days` | integer | Plazo estimado de tránsito en días |
| `carriers[].weight_details.type` | string | `real` (peso real) o `cubed` (peso volumétrico) — cuál se usó en la tarificación |
| `carriers[].weight_details.weight` | string | Peso considerado |
| `carriers[].freight` / `freight_final` | string | Flete antes / después de las tasas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / después de las tasas administrativas |
| `carriers[].tax` / `tax_final` | string | Impuestos antes / después de las tasas administrativas |
| `carriers[].*_subsidy_rule_code` | integer | **Solo en `detailed`.** Código de la regla de subsidio aplicada al flete, al seguro o al impuesto (`null` cuando no hay) |
| `carriers[].*_details` | object | **Solo en `detailed`.** Composición detallada del flete, del seguro o del impuesto (`null` cuando no hay) |
| `carriers[].currency_quote_price` | string | Total en la moneda de la cotización, antes de las tasas administrativas |
| `carriers[].currency_quote_amount` | string | Total a pagar en la moneda de la cotización |
| `carriers[].currency_payment_amount` | string | Total a pagar en la moneda de pago |

## Endpoint: consultar resultado de la cotización

- **Método y ruta:** `GET /v1/quotation/{quotation}/{level}`
- **Cuándo usar:** para reconsultar un resultado ya calculado, cambiar el nivel de detalle o saber si la cotización
  ya se usó (`finished`).

### Parámetros

| Parámetro | Ubicación | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|---|
| `quotation` | path | string | **sí** | UUID | Código retornado en la creación |
| `level` | path | string | no | `simple`, `detailed` | Predeterminado `simple` |

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

> La respuesta tiene la misma estructura de `data` que el `POST` (excepto `warnings`).

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Não foi encontrada a cotação especificada!` | El `quotation` no existe o no pertenece a tu cuenta |
| `400` | `Nivel de detalhe da cotação não existe` | `level` distinto de `simple`/`detailed` |
| `403` | `Has no permission!` | Tu perfil no tiene permiso de consulta de cotización |

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
| [Artículos](#cotización-de-artículos) | `package` | `items` | - | - | **Exclusivo de la v2** |

### Cotización de Documento

- **Cuándo usar:** envío de papeles/documentos sin valor comercial relevante (contratos, certificados, impresos).
- **Cómo armar:** `object: doc` + `type: simple`. Cada bulto (sobre/caja) informa dimensiones, `weight` y `price`.
- **No usa** `items[]`, `ior` ni `manufacturer`: los documentos no pasan por el cálculo de impuestos por artículo.
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  [Paquete — avanzado](#cotización-de-paquete-caja--avanzada) (o, en la v2, [Artículos](api-cotacao-v2.md#cotización-de-artículos)).

#### Payload mínimo

```json
{
  "object": "package",
  "type": "simple",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": true,
  "description": "Pedido con dos cajas",
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": { "value": 35.5, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
      "selling_unit_value": 49.9,
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
      "selling_unit_value": 349.9
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Dos pallets de piezas industriales",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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
  "ior": false,
  "description": "Pallet por cubicaje y pallet por medidas",
  "currency_quote": "USD",
  "currency_payment": "USD",
  "declared_freight": { "value": 250, "currency": "USD" },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami"
  },
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

> **Exclusiva de la [Cotización (v2)](api-cotacao-v2.md#cotización-de-artículos).** La cotización por artículos (`type: items`), en la que
> envías solo el `sku` de los artículos registrados y la plataforma arma las cajas, no forma parte del contrato de la v1.
> En la v1, informa siempre las cajas y usa [Paquete — avanzado](#cotización-de-paquete-caja--avanzada) con `items[].sku`
> para aprovechar el registro de artículos.

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
  "address_sender": { "country_code": "BR", "city": "Diadema", "postal_code": "09941070" },
  "address_receiver": { "country_code": "US", "city": "Miami", "postal_code": "33101" },
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

Una cotización calculada con éxito **no siempre** puede convertirse en envío. Antes de llamar a `POST /v1/shipping`,
verifica las condiciones a continuación. Se comprueban en este orden en el momento del envío:

| # | Condición | Status | Mensaje cuando falla |
|---|---|---|---|
| 1 | La cotización existe y fue creada en la **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La cotización pertenece a tu cuenta | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Cuenta con impuesto bloqueado en DDP: cotización DDU (`tax: receiver`) sin pago aprobado | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | La cotización **todavía no se ha usado** (`finished: false`) | `406` | `Cotação já foi utilizada em outra remessa!` |
| 5 | No es `type: simple` con `object: package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 6 | La cotización **no ha expirado** (`valid_until` del transportista) | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 7 | El transportista elegido está en `carriers[]` de esta cotización | `404` | `Contrato especificado não foi listado para esta cotação!` |

> **Cómo evitarlo:** usa `advanced` para paquete/pallet, crea el envío inmediatamente después de la cotización,
> usa el `code`/`slug` retornado **en esta** cotización y haz **una cotización por envío**.
> Todas las validaciones del envío están en [Envío (v1)](api-remessa-v1.md#cotizaciones-no-elegibles-para-envío).

## Reglas de negocio

### Predefiniciones aplicadas en la cotización
Las [predefiniciones de la cuenta](api-primeiros-passos.md#predefiniciones-de-la-cuenta-presets) influyen en el cálculo:

| Predefinición | Efecto en la cotización |
|---|---|
| Impuesto (`tax`) | Cuando está configurada, **sustituye** el `tax` enviado. Los países de destino listados como excepción reciben lo opuesto al valor predeterminado. Si la cuenta está bloqueada en DDP, el impuesto es siempre `sender` |
| Seguro automático | Si está activo, el seguro se contrata automáticamente cuando el valor declarado (convertido a USD) alcanza el mínimo configurado, incluso con `insurance: false` |
| Flete declarado | Usado cuando no se envía `declared_freight`. Exige el permiso de flete declarado |
| Sistema de medida | Usado cuando no se envía `measurement`. En su ausencia, `metric` |
| Entrega residencial / no apilable | Usados cuando no se envían `residential_delivery` / `non_stackable` |
| Canal de venta | Usado cuando no se envía `channel_code` |
| Agrupación de cajas | No se aplica a la v1 (se usa solo en la cotización por artículos de la v2) |

### Validación de la dirección
En la v1 la dirección manual se **valida ya en la cotización**:

| Caso | Status | Mensaje |
|---|---|---|
| Dirección en Brasil sin `postal_code` | `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` |
| Código postal brasileño inexistente/inválido (cuando no se envía `state_code`) | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Dirección internacional no localizada | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Estado/provincia no reconocido | `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |
| `address_code` inexistente o de otra cuenta | `400` | `Nenhum endereco encontrado para o usuário especificado!` |
| Remitente y destinatario idénticos | `400` | `Endereço de Remetente e Destinatário são iguais!` |

### Restricciones de carga
La cotización puede rechazarse cuando la carga no encaja en los contratos de tu cuenta:

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Caja sin artículo en `advanced` |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` mayor que el número de cajas o menor que 1 |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Artículo manual sin `box_id` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | `weight` por encima del peso máximo de la caja registrada (`simple`) |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Suma de los artículos por encima del peso máximo de la caja registrada |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inexistente o de otra cuenta |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` no registrado y datos manuales incompletos |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...` | `unit_value` igual o menor que cero |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` de otra cuenta |
| `404` | `Nenhum transportador encontrado para este usuário!` | Cuenta sin contrato habilitado para la ruta/objeto |
| `404` | `Não há contratos que atendem as especificações do pacote enviado...` | Ningún contrato cubre la combinación peso/dimensión/ruta |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Carga fuera del rango automático — habla con el equipo comercial |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Ningún transportista retornó resultado |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item)...` | HS Code sin tabla de impuesto en el destino — revisa el `hscode` |

> Los rechazos relacionados con el **peso** pueden venir con `status: "error"` en lugar de `"warning"`, manteniendo el código HTTP
> de la tabla. Trátalos por el código HTTP y por el `message`.

### Alertas de la cotización (`warnings`)
La cotización se realiza, pero `data.warnings` aparece con:

| Alerta | Cuándo |
|---|---|
| `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` | Dos o más artículos con el mismo `name` |
| `Valores declarados dos itens podem ocasionar problemas aduaneiros` | Valor declarado superior a USD 800 con transportista exprés (DHL/FedEx) en el resultado |

### Registros referenciados
Cuando usas códigos de registro, deben existir, estar activos y pertenecer a tu cuenta:

| Campo | Registro | Dónde gestionar |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Caja | [Caja](api-caixa-v1.md) |
| `items[].sku` | Artículo | [Artículos](api-itens-v1.md) |
| `boxes[].kit` | Kit | Panel de Shipsmart Global |
| `address_*.address_code` / `manufacturer.address_code` | Dirección | Panel de Shipsmart Global |
| `channel_code` | Canal de venta | Panel de Shipsmart Global |

## Errores comunes

| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Regla de negocio (carga, caja, artículo, kit, dirección) | Leer `message` y ajustar el escenario |
| `403` | Sin permiso de cotización o canal de otra cuenta | Solicitar el permiso al soporte / revisar `channel_code` |
| `404` | Dirección no localizada o ningún contrato elegible | Revisar dirección y escenario |
| `406` | La carga exige atención personalizada, o remitente registrado obligatorio para el tipo de cuenta | Hablar con el equipo comercial / usar `address_sender.address_code` |
| `422` | Validación de campos o HS Code sin tabla de impuesto | Corregir los campos listados en `data` / revisar `hscode` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- La respuesta del `POST` ya trae el resultado calculado — no es obligatorio llamar al `GET` a continuación.
- `data.warnings` es informativo: la cotización se realizó, pero hay un riesgo aduanero que revisar.
- Para un payload de cotización aún más reducido (solo país y estado), consulta [Cotización V2](api-cotacao-v2.md).
- Para elegir entre las versiones, consulta [Cotización y Envío: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Para cotización + envío en una única llamada sincrónica, consulta [Manifiesto](api-manifesto.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
