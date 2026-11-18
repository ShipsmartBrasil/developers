# Cotización (v1)

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Calcula flete, seguro e impuestos para un envío internacional, en todos los transportistas habilitados en tu cuenta.
- Es el **primer paso obligatorio** del flujo: el envío, el seguimiento, los documentos y la programación de recogida
  están todos vinculados al código de cotización (`quotation`).
- La v1 es la versión completa: acepta dirección detallada en el origen/destino y posee endpoint de reconsulta del resultado.

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
1. `POST /v1/quotation` con origen, destino, objeto, tipo y cajas/artículos.
2. La respuesta ya trae el resultado calculado (`data.carriers`) y el código `data.quotation`.
3. Si necesitas reconsultar (o cambiar el nivel de detalle), usa `GET /v1/quotation/{quotation}/{level}`.
4. Elige un transportista de la lista y usa `code` (o `slug`) + `quotation` para crear el
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
| `doc` | Documento | Cotización por caja, informando `weight` y `price` |
| `package` | Paquete/caja | Flujo más común |
| `pallet` | Palé | Permite informar `volume` en lugar de altura/anchura/profundidad |
| `not_doc` | Legado | Mantenido por compatibilidad; se convierte internamente a `package`. **No lo uses en integraciones nuevas** |

### `type` — cómo se monta el cálculo
| Valor | Lo que envías | Cuándo usarlo |
|---|---|---|
| `simple` | Solo `boxes[]` con `weight` y `price` | Simulación rápida de flete. **No es elegible para envío cuando `object` es `package` o `pallet`** |
| `advanced` | `boxes[]` (dimensiones) + `items[]` con `box_id` apuntando a la posición de la caja | Flujo estándar para envío: controlas la distribución de los artículos en las cajas |
| `items` | Solo `items[]` con `sku` (las cajas son elegidas por la plataforma) | Cuando los artículos y las cajas ya están registrados y quieres que la plataforma haga el encaje |

> **Atención:** `type: simple` con `object: package` o `object: pallet` sirve solo para simulación.
> Al intentar crear el envío, la API responde `Cotação Simples é inelegível para remessa!`.
> Para envío usa `advanced` o `items`.

### `tax` — quién paga los impuestos
| Valor | Significado |
|---|---|
| `sender` | DDP: el remitente paga impuestos y tasas en el destino |
| `receiver` | DDU: el destinatario paga impuestos y tasas en el destino |

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

Enviado como **query string** en el `POST` y como **segmento de path** en el `GET`.

## Endpoint: crear cotización

- **Método y ruta:** `POST /v1/quotation`
- **Query string opcional:** `level=simple|detailed`
- **Cuándo usarlo:** siempre que necesites precio, plazo e impuestos, y antes de crear cualquier envío.

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
| `ior` | boolean | no | - | - | `false` | `true`, `false` | Importer of Record. Cuando es `true`, `items[].selling_unit_value` pasa a ser necesario |
| `residential_delivery` | boolean | no | - | - | `false` | `true`, `false` | Entrega en dirección residencial |
| `non_stackable` | boolean | no | - | - | `false` | `true`, `false` | Bultos no apilables |
| `description` | string | no | - | 155 | `null` | - | Descripción libre de la cotización, para tu referencia |
| `channel_code` | integer | no | - | - | preset de la cuenta | - | Código del canal de venta registrado en tu cuenta |
| `channel_identifier_number` | string | no | - | - | `null` | - | Tu identificador comercial (nº del pedido). **Recomendado**: permite consultar status/documentos por canal |
| `channel_order_number` | string | no | - | 100 | `null` | - | Número del pedido en el canal de venta |
| `declared_freight` | object | no | - | - | preset de la cuenta | - | Flete declarado (ver tabla abajo) |
| `manufacturer` | object | no | - | - | `null` | - | Fabricante mostrado en la Commercial Invoice (ver tabla abajo) |
| `level` | string | no (query) | - | - | `simple` | `simple`, `detailed` | Nivel de detalle del resultado |

### Parámetros — `address_sender` y `address_receiver`

Hay **dos formas** de informar la dirección. Elige una y no las mezcles:

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de una dirección registrada en tu cuenta. Cuando se usa, no envíes los demás campos |

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `country_code` | string | **sí** | 2 | 2 | País en ISO 3166-1 alpha-2 (ej.: `BR`, `US`) |
| `city` | string | **sí** | - | 100 | Ciudad |
| `postal_code` | string | condicional | - | 12 | Código postal/ZIP. Obligatorio si no se informa `state`. **Para direcciones en Brasil es siempre obligatorio** |
| `state` | string | condicional | - | - | Nombre del estado/provincia. Obligatorio si no se informan `state_code` ni `postal_code` |
| `state_code` | string | condicional | - | 10 | Sigla del estado/provincia (ej.: `SP`, `FL`). Obligatorio si no se informan `state` ni `postal_code` |

> **Regla práctica:** informa siempre `country_code` + `city` + (`postal_code` **o** `state_code`).
> Esa combinación atiende a todos los países.

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

> `weight` y `price` son **ignorados** cuando la caja usa `kit`.

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `quantity` | integer | **sí** | 1 | 1000 | Cantidad del artículo |
| `unit_value` | number | **sí** | - | - | Valor unitario en la moneda de la cotización |
| `box_id` | integer | **sí**, excepto cuando `type` es `items` | - | - | **Posición** de la caja en el array `boxes` (1 = primera caja, 2 = segunda...). No es el `box_code` |
| `sku` | string | **sí** cuando `type` es `items` | - | 50 | SKU del artículo registrado ([Artículos](api-itens-v1.md)). Cuando se informa, se usan los datos del registro y los demás campos pueden omitirse |
| `ean` | string | no | - | 50 | Código EAN del artículo |
| `name` | string | **sí** cuando no se informa `sku` | 6 | 50 | Nombre del artículo (aparece en la invoice) |
| `description` | string | no | - | 155 | Descripción detallada del artículo |
| `country_code` | string | **sí** cuando no se informa `sku` | 2 | 2 | País de origen/fabricación del artículo |
| `hscode` | string | **sí** cuando no se informa `sku` | 6 | 10 | NCM/HS Code del artículo. Define la tabla de impuestos aplicada |
| `weight` | number | **sí** cuando no se informa `sku` | 0.001 | - | Peso unitario del artículo |
| `selling_unit_value` | number | condicional | - | - | Valor de venta unitario. **Necesario cuando `ior` es `true`** |
| `is_groupable` | boolean | no | - | - | Predeterminado `true`. Cuando es `false`, el artículo recibe una caja exclusiva en el agrupamiento `smartbox` |

> **Descripciones iguales en artículos diferentes** generan un `warning` en la respuesta
> (`Itens com descrições iguais podem causar problemas na fiscalização aduaneira`).
> La cotización se realiza, pero conviene revisar las descripciones para evitar la retención en la aduana.

### Parámetros — `declared_freight`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valor del flete declarado |
| `currency` | string | **sí** cuando se informa `value` | 3 | 3 | Moneda del flete declarado. Misma lista de `currency_quote` |

### Parámetros — `manufacturer`

Datos del fabricante impresos en la Commercial Invoice. Envía `address_code` **o** los campos manuales.

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `address_code` | integer | alternativa a los manuales | - | - | Código de dirección registrada. Cuando se usa, los demás campos son opcionales |
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

> Para artículos con `hscode` que empieza en `64` (calzado), el nombre del fabricante es exigido por la aduana.

### Payload mínimo

Cotización de documento, tipo simple — el menor payload aceptado:

```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
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
    "state_code": "FL",
    "postal_code": "33101"
  },
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

Para generar un envío de paquete es necesario usar `advanced` (o `items`) y declarar los artículos:

```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
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

Todos los campos aceptados por el endpoint:

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
      "name": "Camiseta algodon S",
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
        "weight_details": {
          "type": "real",
          "weight": "5"
        },
        "freight": "181.95",
        "freight_final": "181.95",
        "insurance": "11.45",
        "insurance_final": "11.45",
        "tax": "28153.894",
        "tax_final": "28153.894",
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
| `data.finished` | boolean | Indica si el procesamiento de la cotización fue concluido |
| `data.date` | string | Fecha/hora de la cotización |
| `data.channel` | string | Nombre del canal de venta, cuando se informó `channel_code` |
| `data.warnings` | array | **Opcional.** Solo aparece cuando hay alertas (ej.: descripciones duplicadas, valores declarados altos) |
| `data.carriers[]` | array | Opciones de envío calculadas |
| `carriers[].code` | integer | Código del contrato. Úsalo en `carrier_code` en el envío |
| `carriers[].slug` | string | Identificador textual del contrato. Alternativa al `carrier_code` en el envío |
| `carriers[].name` | string | Nombre comercial del transportista |
| `carriers[].url_image` | string | Logo del transportista |
| `carriers[].valid_until` | string | **Validez de la cotización.** El envío debe crearse antes de esta fecha/hora |
| `carriers[].transit_days` | integer | Plazo estimado de tránsito en días |
| `carriers[].weight_details.type` | string | `real` (peso real) o `cubed` (peso volumétrico) — cuál se usó en la tarifación |
| `carriers[].weight_details.weight` | string | Peso considerado |
| `carriers[].freight` / `freight_final` | string | Flete antes / después de las tasas administrativas |
| `carriers[].insurance` / `insurance_final` | string | Seguro antes / después de las tasas administrativas |
| `carriers[].tax` / `tax_final` | string | Impuestos antes / después de las tasas administrativas |
| `carriers[].currency_quote_price` | string | Total en la moneda de la cotización |
| `carriers[].currency_quote_amount` | string | Total a pagar en la moneda de la cotización |
| `carriers[].currency_payment_amount` | string | Total a pagar en la moneda de pago |

## Endpoint: consultar el resultado de la cotización

- **Método y ruta:** `GET /v1/quotation/{quotation}/{level}`
- **Cuándo usarlo:** para reconsultar un resultado ya calculado o cambiar el nivel de detalle.

| Parámetro | Lugar | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|---|
| `quotation` | path | string | **sí** | UUID | Código retornado en la creación |
| `level` | path | string | no | `simple`, `detailed` | Predeterminado `simple` |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

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

> Si el `quotation` no existe o no pertenece a tu cuenta, la API responde `400` con
> `Não foi encontrada a cotação especificada!`.

## Reglas de negocio

### Elegibilidad para envío
Para que una cotización pueda generar un envío, todas las condiciones de abajo deben ser verdaderas:

1. El `quotation` existe y pertenece a tu cuenta.
2. La cotización **no está expirada** (respeta el `valid_until` del transportista elegido).
3. La cotización **todavía no fue usada** en otro envío (`Cotação já foi utilizada em outra remessa!`).
4. La cotización fue creada en la **misma versión** de la API usada para el envío (v1 → v1).
5. No es una cotización `simple` de `package`/`pallet` (`Cotação Simples é inelegível para remessa!`).
6. El transportista informado está en la lista retornada por la cotización
   (`Contrato especificado não foi listado para esta cotação!`).

### Restricciones de carga
La cotización puede ser rechazada con `400` cuando:

| Mensaje | Causa |
|---|---|
| `A caixa :name tem a altura/largura/profundidade maior do que a permitida!` | Dimensión por encima del límite de los contratos |
| `A caixa :name está com o peso maior que o permitido!` | Peso por bulto por encima del límite |
| `Remessa está com o peso acima do permitido!` | Peso total por encima del límite |
| `Nenhum item foi associado para a caixa :name !` | Caja sin artículo en `advanced` |
| `A cotação excedeu o máximo de itens permitido!` | Número de artículos por encima del límite |
| `Nenhum transportador encontrado para este usuário!` | Cuenta sin contrato habilitado |
| `Não há contratos que atendem as especificações do pacote enviado...` | Ningún contrato cubre la combinación peso/dimensión/ruta |
| `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado...` | Carga fuera del rango automático — habla con el equipo comercial |
| `Endereço de Remetente e Destinatário são iguais!` | Origen y destino idénticos |
| `O país de origem é o mesmo de destino!` | La API atiende envíos internacionales |
| `Não foi passado um postal code para o endereço do Brasil no :name !` | Dirección brasileña sin código postal |
| `Este país de Origem/Destino está temporariamente inabilitado para envios!` | Ruta bloqueada en este momento |
| `Não existe tabela de duty para o país de destino :name com o hscode :hscode...` | HS Code sin tabla de impuestos en el destino — revisa el `hscode` |

### Registros referenciados
Cuando usas códigos de registro, estos deben existir y estar activos:

| Campo | Registro | Página |
|---|---|---|
| `boxes[].box_code` / `boxes[].box_identifier` | Caja | [Caja](api-caixa-v1.md) |
| `items[].sku` | Artículo | [Artículos](api-itens-v1.md) |
| `address_*.address_code` | Dirección | Registrada en el panel |
| `channel_code` | Canal de venta | Registrado en el panel |
| `boxes[].kit` | Kit | Registrado en el panel |

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Regla de negocio (restricción de carga, ruta, contrato, HS Code) | Leer `message` y ajustar el escenario |
| `403` | Sin permiso de cotización | Solicitar el permiso al soporte |
| `406` | Estructura del payload incompatible | Revisar tipos y anidamiento |
| `422` | Validación de campos | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- La respuesta del `POST` ya trae el resultado calculado — no es obligatorio llamar al `GET` a continuación.
- `data.warnings` es informativo: la cotización se realizó, pero hay riesgo aduanero por revisar.
- Para un payload de cotización aún más reducido (solo país y estado), consulta [Cotización V2](api-cotacao-v2.md).
- Para cotización + envío en una única llamada sincrónica, consulta [Manifiesto](api-manifesto.md).
