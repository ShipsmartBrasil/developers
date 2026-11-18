# Manifiesto

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Ejecuta **cotización + envío + generación de documentos en una única llamada sincrónica**.
- La respuesta ya trae el número de seguimiento y los documentos (AWB, y los demás cuando aplique).
- Es el flujo indicado para operaciones que necesitan la etiqueta **inmediatamente**, sin esperar
  procesamiento asíncrono.

> ⚠️ **El manifiesto es sincrónico e irreversible.** Como cotiza y despacha en la misma solicitud, no hay
> etapa intermedia de revisión. Valida bien el payload antes de enviarlo.
>
> ⚠️ **Solo acepta pago facturado** (`payment.type: billed`).

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

## Manifiesto vs. cotización + envío

| Tema | Cotización + Envío | Manifiesto |
|---|---|---|
| Llamadas | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Elección del transportista | Tú eliges en la lista retornada | La plataforma lo resuelve (o lo fijas con `carrier_code`/`slug`) |
| Documentos | Consulta posterior en `/docs` | **Vienen en la respuesta** |
| Pago | `billed`, `gateway`, `afterward` | **Solo `billed`** |
| Tipo de respuesta | Asíncrono | **Sincrónico** |
| Revisión antes del despacho | Sí | No |
| Campos obligatorios | Menos (divididos en 2 etapas) | **Más** (todo de una vez) |

Usa el manifiesto cuando: la operación es de alto volumen, el transportista ya está definido por regla de
negocio y necesitas imprimir la etiqueta en el momento.

## Flujo de uso
1. Monta el payload completo (direcciones, cajas, artículos, pago).
2. `POST /v1/manifest?format=pdf` (o `zpl`).
3. Imprime la etiqueta recibida en `data.docs`.
4. Usa el `data.shipping` para [seguimiento](api-rastreio.md), [documentos](api-documentos.md) y
   [programación de recogida](api-agendamento.md).

## Endpoint: crear manifiesto

- **Método y ruta:** `POST /v1/manifest`

### Parámetros de query

| Nombre | Lugar | Tipo | Obligatorio | Predeterminado | Valores | Descripción |
|---|---|---|---|---|---|---|
| `format` | query | string | no | todos | `pdf`, `zpl` | Formato de los documentos en la respuesta. Vacío retorna todos los formatos disponibles |

> `format` también puede enviarse **en el cuerpo** del payload. Cuando se envía en los dos lugares,
> **el valor del cuerpo prevalece**.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Valores / Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `object` | string | **sí** | - | - | `doc`, `package`, `pallet`, `not_doc` | Qué se está enviando |
| `type` | string | **sí** | - | 30 | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalidad del envío |
| `tax` | string | **sí** | - | - | `sender`, `receiver` | Quién paga los impuestos |
| `insurance` | boolean | **sí** | - | - | - | Contratar seguro |
| `residential_delivery` | boolean | **sí** | - | - | - | Entrega en dirección residencial |
| `non_stackable` | boolean | **sí** | - | - | - | Bultos no apilables |
| `measurement` | string | **sí** | - | - | `metric`, `imperial` | Sistema de medida |
| `ior` | boolean | **sí** | - | - | - | Importer of Record |
| `currency_quote` | string | **sí** | 3 | 3 | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda de la cotización |
| `currency_payment` | string | **sí** | 3 | 3 | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda del pago |
| `address_sender` | object | **sí** | - | - | - | Remitente (ver tabla) |
| `address_receiver` | object | **sí** | - | - | - | Destinatario (ver tabla) |
| `boxes` | array | **sí** | 1 ítem | - | - | Bultos (ver tabla) |
| `payment` | object | **sí** | - | - | - | Pago (ver tabla) |
| `items` | array | condicional | - | - | - | Obligatorio cuando `object` **no** es `doc` y ninguna caja usa `kit`. **Prohibido cuando `object` es `doc`** |
| `carrier_code` | integer | no | - | - | - | Fija el contrato a usar |
| `slug` | string | no | - | 20 | - | Fija el contrato por el identificador textual |
| `channel_code` | integer | no | - | - | - | Código del canal de venta registrado |
| `channel_identifier_number` | string | no | - | - | - | Tu identificador comercial. **Recomendado** |
| `nfkey` | string | no | 44 | 44 | - | Clave de la NF-e |
| `description` | string | no | 5 | 155 | - | Descripción libre del manifiesto |
| `declared_freight` | object | no | - | - | - | Flete declarado (ver tabla) |
| `biller` | object | no | - | - | - | Facturador mostrado en la invoice |
| `manufacturer` | object | no | - | - | - | Fabricante mostrado en la invoice |
| `extra` | object | no | - | - | - | Datos auxiliares en pares clave/valor (ver tabla) |
| `format` | string | no | - | - | `pdf`, `zpl` | Alternativa a la query string |
| `size` | string | no | - | - | `label`, `letter` | Tamaño de la etiqueta |

### Parámetros — `address_sender` y `address_receiver`

A diferencia del envío v1/v2, aquí la dirección se informa **completa**, porque la cotización ocurre
en la misma llamada.

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de la dirección registrada. **No envíes ningún otro campo de dirección junto** |

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `country_code` | string | **sí** | 2 | 2 | - | País en ISO 3166-1 alpha-2 |
| `city` | string | condicional | 1 | 100 | - | Ciudad. Obligatoria si no se informa `postal_code` |
| `postal_code` | string | condicional | - | 12 | - | Código postal/ZIP. Obligatorio si no se informan `state` ni `state_code` |
| `state` | string | condicional | - | 100 | - | Nombre del estado. Obligatorio si no se informan `state_code` ni `postal_code` |
| `state_code` | string | condicional | 1 | 10 | - | Sigla del estado, validada contra el país. Obligatoria si no se informan `state` ni `postal_code` |
| `street` | string | **sí** | 5 | 70 | - | Calle |
| `name` | string | **sí** | 1 | 70 | - | Nombre del contacto/razón social |
| `email` | string | **sí** | - | 60 | - | E-mail válido |
| `type` | string | **sí** | - | - | `pj` | `pf` o `pj` |
| `foreign` | boolean | **sí** | - | - | - | Indica si la persona es extranjera (usado para validar el `federal_tax_id`) |
| `federal_tax_id` | string | condicional | - | 20 | - | **Obligatorio cuando el país es `BR` o `AR`** |
| `phone` | string | no | 1 | 25 | - | Teléfono |
| `phone_extension` | string | no | - | 5 | - | Extensión |
| `number` | string | no | 1 | 6 | - | Número del inmueble |
| `district` | string | no | 1 | 30 | - | Barrio |
| `complement` | string | no | 1 | 20 | - | Complemento |
| `description` | string | no | 1 | 155 | - | Etiqueta interna de la dirección |
| `state_tax_id` | string | no | - | 20 | - | Inscripción estatal |
| `company_name` | string | no | 5 | 50 | - | **Solo en `address_sender`.** Razón social exigida por algunos envíos FedEx |

### Parámetros — `boxes[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `kit` | string | alternativa a las demás formas | - | 50 | SKU del kit registrado. Completa la caja **y** los artículos de esa posición |
| `box_code` | integer | alternativa | - | - | Código de la caja registrada ([Caja](api-caixa-v1.md)) |
| `box_identifier` | string | alternativa | - | 50 | Identificador de la caja registrada |
| `name` | string | **sí** sin `kit`/`box_code`/`box_identifier` | 5 | 155 | Nombre de la caja |
| `height` | number | **sí** sin `kit`/`box_code`/`box_identifier` | 1 | - | Altura (cm en `metric`, in en `imperial`) |
| `width` | number | **sí** sin `kit`/`box_code`/`box_identifier` | 1 | - | Anchura |
| `depth` | number | **sí** sin `kit`/`box_code`/`box_identifier` | 1 | - | Profundidad |
| `weight` | number | **sí** cuando `object` es `doc` | 0.001 | - | Peso del bulto (kg en `metric`, lb en `imperial`) |
| `price` | number | **sí** cuando `object` es `doc` | 0 | - | Valor declarado del bulto |

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `box_id` | integer | **sí** | - | - | **Posición** de la caja en el array `boxes` (1 = primera) |
| `quantity` | integer | **sí** | 1 | 99 | Cantidad del artículo |
| `sku` | string | alternativa a los campos manuales | 1 | 50 | SKU del artículo registrado ([Artículos](api-itens-v1.md)) |
| `name` | string | **sí** sin `sku` | 6 | 50 | Nombre del artículo |
| `country_code` | string | **sí** sin `sku` | 2 | 2 | País de origen del artículo |
| `hscode` | string | **sí** sin `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **sí** sin `sku` | 0.001 | 99 | Peso unitario |
| `unit_value` | number | **sí** sin `sku` | 0 | - | Valor unitario |
| `description` | string | no | 5 | 450 | Descripción detallada del artículo |
| `selling_unit_value` | number | condicional | 0 | - | Valor de venta unitario. **Necesario cuando `ior` es `true`** |

> **Atención a las diferencias respecto a la cotización:** aquí `quantity` tiene un máximo de **99** (no 1000) y
> `description` acepta hasta **450** caracteres (no 155).

### Parámetros — `payment`

| Campo | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|
| `payment` | object | **sí** | - | Envelope |
| `payment.type` | string | **sí** | **`billed`** | El manifiesto acepta **solamente** pago facturado |

### Parámetros — `declared_freight`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `declared` | boolean | no | - | - | Habilita el flete declarado. Con `true`, el valor puede ser `0` y los impuestos se calculan con flete declarado cero |
| `value` | number | no | 0 | - | Valor del flete declarado |
| `currency` | string | **sí** cuando se informa `value` | 3 | 3 | Moneda del flete declarado |

### Parámetros — `extra`

Pares clave/valor auxiliares, transmitidos al proceso del manifiesto.

| Regla | Detalle |
|---|---|
| Estructura | Objeto **plano** (un nivel, sin anidamiento) |
| Claves | Solo letras minúsculas y `_` |
| Valores | Texto, entero o decimal |

| Clave conocida | Tipo | Regla | Descripción |
|---|---|---|---|
| `federal_tax_provision` | number | - | Provisión de impuesto federal |
| `state_tax_provision` | number | - | Provisión de impuesto estatal |
| `usd_rate_provision` | number | > 0 | Provisión de cambio del dólar |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parámetros — `biller` y `manufacturer`

- `biller`: mismo formato descrito en [Envío (v1)](api-remessa-v1.md#parámetros--biller-facturador).
  En el manifiesto, `biller.name` acepta hasta **355** caracteres.
- `manufacturer`: mismo formato descrito en [Cotización (v1)](api-cotacao-v1.md#parámetros--manufacturer).

### Payload mínimo

Manifiesto de documento (`object: doc`) — `items` está prohibido en este caso:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
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
    "street": "Calle Principal",
    "name": "Tienda Ejemplo LTDA",
    "email": "contato@example.com",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Sobre documentos",
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

### Payload máximo

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
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
  "description": "Manifiesto del lote de marzo",
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
    "description": "Centro de distribucion SP",
    "street": "Calle Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpon 2",
    "company_name": "Tienda Ejemplo Comercio LTDA",
    "name": "Tienda Ejemplo LTDA",
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
    "description": "Oficina Miami",
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
  "boxes": [
    {
      "name": "Caja master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Camiseta algodon S",
      "description": "Camiseta 100% algodon, manga corta, color blanco, talla S",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Zapatillas deportivas",
      "description": "Zapatillas de running para hombre numero 44",
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
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Calle Principal",
    "number": "100",
    "name": "Holding Ejemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabrica Ejemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Calle Principal",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "type": "pj"
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

### Ejemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Calle Principal", "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "type": "pj", "federal_tax_id": "12345678000199", "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "type": "pf", "foreign": true},
    "boxes": [{"name": "Sobre documentos", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "271249639538",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.shipping` | string (UUID) | **Identificador del envío.** Úsalo en seguimiento, documentos y programación |
| `data.carrier` | string | Transportista que procesó el envío |
| `data.tracking_id` | string | Número de seguimiento (AWB) |
| `data.docs` | object | Documentos generados, en el formato `nombre → { url, base64 }` |

> Los documentos adicionales (invoice, packing list, declaración) pueden generarse en segundo plano.
> Consulta [Documentos del Envío](api-documentos.md) algunos instantes después para obtener la lista completa.

## Error del transportista

Cuando el transportista rechaza el envío, la respuesta es `400` y trae el retorno bruto de este:

```json
{
  "status": "error",
  "message": "[1234] CARRIER - Carrier error message",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Campo | Descripción |
|---|---|
| `message` | `[código] TRANSPORTISTA - mensaje` retornado por el transportista |
| `data` | Payload bruto del transportista. **La estructura varía por transportista** |

> Registra `message` y `data` en tus logs: es la única fuente del motivo técnico del rechazo.

## Reglas de negocio
- **Pago:** solamente `billed`. Cualquier otro valor es rechazado en la validación.
- **`items` y `object: doc`:** con `object: doc`, el campo `items` **no puede** enviarse.
  Con cualquier otro `object`, `items` es obligatorio — excepto cuando alguna caja usa `kit`.
- **Kits:** cuando una caja informa `kit`, la caja y los artículos de esa posición vienen del registro,
  y `boxes[].weight`/`boxes[].price` son ignorados.
- **`state_code`:** validado contra el `country_code` de la misma dirección.
- **`federal_tax_id`:** obligatorio para `BR` y `AR`.
- **`address_code`:** cuando se usa, **ningún** otro campo de dirección puede enviarse en el mismo objeto.
- **Los preajustes de la cuenta** se aplican de la misma forma que en el borrador: el payload siempre vence al preset.
  Informar `declared_freight` descarta el preset de flete declarado; informar
  `address_sender.country_code` descarta el preset de dirección del remitente.
- **`not_doc`** es aceptado por compatibilidad y convertido internamente a `package`.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Error del transportista o regla de negocio | Leer `message` y `data`; ajustar el payload o contactar al soporte |
| `403` | Sin permiso para crear manifiesto | Solicitar el permiso al soporte |
| `422` | Validación de campos | Corregir los campos listados en `data` |
| `500` | Error inesperado | **Antes de repetir**, verifica en [Seguimiento](api-rastreio.md) si el envío fue creado |

> ⚠️ **Nunca repitas un manifiesto automáticamente después de un `500`.** Como la llamada crea el envío de verdad,
> una repetición puede generar un envío duplicado. Verifica primero por el `channel_identifier_number`.

## Observaciones finales
- Informa siempre `channel_identifier_number`: es tu protección contra la duplicidad y la forma de localizar
  el envío si la respuesta se pierde.
- El manifiesto concentra mucha responsabilidad en una sola llamada. Para integraciones que pueden esperar,
  el flujo de [cotización](api-cotacao-v1.md) + [envío](api-remessa-v1.md) es más seguro.
- Para el flujo opuesto (payload parcial, validado después), consulta [Borrador](api-rascunho.md).
