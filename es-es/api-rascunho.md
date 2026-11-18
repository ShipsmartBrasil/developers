# Borrador

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Permite enviar un envío **incompleto**: mandas lo que ya tienes, y la plataforma valida después.
- El borrador se procesa de forma **asíncrona** y, cuando está completo, evoluciona a cotización + envío
  automáticamente.
- Es el flujo indicado cuando la integración recibe los datos del pedido en etapas (ej.: el pedido llega antes
  de la dirección final o de la nota fiscal).

> **Prácticamente todos los campos son opcionales en el envío.** La validación rígida ocurre en el momento del
> despacho. Mientras falten datos, el borrador queda con status `incomplete` y puedes completarlo.

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

## Borrador vs. otros flujos

| Tema | Cotización + Envío | Manifiesto | Borrador |
|---|---|---|---|
| Payload | Completo en 2 etapas | Completo en 1 llamada | **Parcial, en cuantas etapas necesites** |
| Validación | Sincrónica, por etapa | Sincrónica, total | **Asíncrona, en el despacho** |
| Respuesta | Cotización con precios | Documentos listos | Confirmación de recepción |
| Pago | `billed`, `gateway`, `afterward` | `billed` | **`billed` o `afterward`** |
| Corrección posterior | Nueva cotización | No | **Sí, vía `PUT`** |

## Ciclo de vida del borrador

```
POST /v1/draft
     │
     ▼
  received  ──►  validating  ──►  incomplete  ──┐
                      │                          │ PUT /v1/draft/{code}
                      ▼                          │
                   complete  ◄───────────────────┘
                      │
                      ▼
                 processing  ──►  cotización + envío
                      │
                      └──►  error
```

| Status | Significado | ¿Se puede editar? |
|---|---|---|
| `received` | Borrador recibido, esperando validación | No |
| `validating` | En validación | No |
| `incomplete` | **Faltan datos.** El motivo está en `status_descricao` | **Sí** |
| `edited` | Borrador actualizado vía `PUT`, esperando nueva validación | No |
| `complete` | Todos los datos presentes, listo para el despacho | No |
| `processing` | En procesamiento: generando cotización y envío | No |
| `error` | Fallo en el procesamiento | No |

> **Solo los borradores con status `incomplete` pueden editarse.** Intentar editar en otro status
> retorna `400 Apenas rascunhos com o status incomplete podem ser editados`.

## Flujo de uso
1. `POST /v1/draft` con los datos que ya tienes.
2. Espera el procesamiento asíncrono.
3. `GET /v1/draft` para ver el status y, si está `incomplete`, leer el `status_descricao`.
4. `PUT /v1/draft/{code}` para completar lo que faltó.
5. Cuando quede `complete`, la plataforma genera la cotización y el envío.
6. Acompaña por el [seguimiento](api-rastreio.md) y descarga los [documentos](api-documentos.md).
7. `DELETE /v1/draft` para descartar los borradores que no se usarán.

## Endpoint: crear borrador

- **Método y ruta:** `POST /v1/draft`
- **Procesamiento:** asíncrono.

### Parámetros — raíz del payload

Ningún campo es obligatorio, pero **al menos uno** debe enviarse.

| Campo | Tipo | Mín. | Máx. | Predeterminado | Valores | Descripción |
|---|---|---|---|---|---|---|
| `object` | string | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` | Qué se está enviando |
| `type` | string | - | 30 | `Commercial` | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalidad del envío |
| `tax` | string | - | - | preset de la cuenta | `sender`, `receiver` | Quién paga los impuestos |
| `insurance` | boolean | - | - | preset de la cuenta | - | Contratar seguro |
| `measurement` | string | - | - | `metric` | `metric`, `imperial` | Sistema de medida |
| `ior` | boolean | - | - | `false` | - | Importer of Record |
| `residential_delivery` | boolean | - | - | `false` | - | Entrega residencial |
| `non_stackable` | boolean | - | - | `false` | - | Bultos no apilables |
| `currency_quote` | string | 3 | 3 | preset de la cuenta | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda de la cotización |
| `currency_payment` | string | 3 | 3 | preset de la cuenta | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda del pago |
| `nfkey` | string | 44 | 44 | - | - | Clave de la NF-e |
| `description` | string | - | 155 | - | - | Descripción libre del borrador |
| `channel_code` | integer | - | - | preset de la cuenta | - | Código del canal de venta registrado |
| `channel_identifier_number` | string | - | - | - | - | Tu identificador comercial. **Recomendado** |
| `carrier_code` | integer | - | - | - | - | Contrato preferido, si está disponible en la cotización |
| `address_sender` | object | - | - | preset de la cuenta | - | Remitente (ver tabla) |
| `address_receiver` | object | - | - | - | - | Destinatario (ver tabla) |
| `boxes` | array | - | - | - | - | Bultos (ver tabla) |
| `items` | array | - | - | - | - | Artículos (ver tabla) |
| `payment` | object | - | - | - | - | Pago (ver tabla) |
| `biller` | object | - | - | preset de la cuenta | - | Facturador mostrado en la invoice |
| `manufacturer` | object | - | - | - | - | Fabricante mostrado en la invoice |

### Parámetros — `address_sender` y `address_receiver`

Todos opcionales en el envío. Las columnas "Obligatorio en el despacho" indican lo que la plataforma exige para que el
borrador pase a `complete`.

| Campo | Tipo | Mín. | Máx. | Obligatorio en el despacho | Descripción |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Código de dirección registrada (solo en `address_sender`) |
| `country_code` | string | 2 | 2 | **Sí** | País en ISO 3166-1 alpha-2 |
| `city` | string | - | 100 | **Sí** | Ciudad |
| `postal_code` | string | - | 12 | Sí, si no se informa `state` | Código postal/ZIP |
| `state` | string | - | - | Sí, si no se informan `state_code`/`postal_code` | Nombre del estado |
| `state_code` | string | - | - | Sí, si no se informan `state`/`postal_code` | Sigla del estado, validada contra el país |
| `street` | string | - | 70 | **Sí** | Calle |
| `name` | string | 5 | 35 | **Sí** | Nombre del contacto/razón social |
| `email` | string | - | 60 | **Sí** | E-mail válido |
| `phone` | string | - | 25 | **Sí** | Teléfono |
| `type` | string | - | - | **Sí** | `pf` o `pj` (predeterminado `pj`) |
| `federal_tax_id` | string | - | 20 | Sí cuando el país es `BR` | CPF/CNPJ o documento federal |
| `state_tax_id` | string | - | 20 | Sí cuando el país es `BR` | Inscripción estatal |
| `number` | string | - | 100 | - | Número del inmueble |
| `district` | string | - | 30 | - | Barrio |
| `complement` | string | - | 20 | - | Complemento |
| `description` | string | - | 155 | - | Etiqueta interna de la dirección |
| `phone_extension` | string | - | 5 | - | Extensión |
| `foreign` | boolean | - | - | - | Indica si la persona es extranjera |

### Parámetros — `boxes[]`

| Campo | Tipo | Mín. | Máx. | Descripción |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU del kit registrado. Completa la caja **y** los artículos de esa posición en el despacho |
| `box_code` | integer | - | - | Código de la caja registrada ([Caja](api-caixa-v1.md)) |
| `box_identifier` | string | - | - | Identificador de la caja registrada |
| `name` | string | 4 | 155 | Nombre de la caja |
| `height` | number | 1 | - | Altura |
| `width` | number | 1 | - | Anchura |
| `depth` | number | 1 | - | Profundidad |
| `volume` | number | 0.001 | - | Volumen (alternativa a las dimensiones, para `pallet`) |
| `weight` | number | 0.1 | - | Peso del bulto. **Obligatorio cuando `object` es `doc`.** Hasta 3 decimales. Ignorado cuando la caja usa `kit` |
| `price` | number | 0 | - | Valor declarado. **Obligatorio cuando `object` es `doc`.** Ignorado cuando la caja usa `kit` |

### Parámetros — `items[]`

| Campo | Tipo | Mín. | Máx. | Obligatorio en el despacho | Descripción |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU del artículo registrado ([Artículos](api-itens-v1.md)) |
| `ean` | string | - | 50 | - | Código EAN |
| `box_id` | integer | - | - | - | **Posición** de la caja en el array `boxes` (1 = primera) |
| `name` | string | 6 | 35 | Sí, sin `sku` | Nombre del artículo |
| `description` | string | 4 | 155 | - | Descripción del artículo |
| `country_code` | string | 2 | 2 | Sí, sin `sku` | País de origen del artículo |
| `hscode` | string | 6 | 10 | Sí, sin `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Sí, sin `sku` | Peso unitario |
| `quantity` | integer | 1 | 1000 | **Sí** | Cantidad |
| `unit_value` | number | - | - | **Sí** | Valor unitario |
| `selling_unit_value` | number | - | - | Sí cuando `ior` es `true` | Valor de venta unitario |
| `is_groupable` | boolean | - | - | - | Permite agrupar el artículo con otros |
| `height`, `width`, `depth` | number | 1 | - | - | Dimensiones del artículo |
| `is_foldable` | boolean | - | - | - | Artículo plegable (usado en el agrupamiento `smartbox`) |

> `items` pasa a ser **obligatorio en el despacho** cuando `object` es `package`, `pallet` o `not_doc`.

### Parámetros — `payment`

| Campo | Tipo | Obligatorio en el despacho | Valores | Descripción |
|---|---|---|---|---|
| `payment.type` | string | **Sí** | `billed`, `afterward` | El borrador **no acepta** `gateway` |

### Payload mínimo

El menor payload útil — cualquier campo sirve, pero el borrador necesita tener al menos uno:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package"
}
```

Un ejemplo más realista, con lo que normalmente ya se tiene en el momento del pedido:

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
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
  ],
  "payment": { "type": "billed" }
}
```

### Payload máximo

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
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
  "description": "Borrador del pedido del marketplace",
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
    "state_tax_id": "987654321",
    "foreign": true
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
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Camiseta algodon S",
      "description": "Camiseta 100% algodon manga corta blanca",
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
  "payment": { "type": "billed" }
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SU_TOKEN' \
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
    "boxes": [{"height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodon", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Necessário informar pelo menos um parâmetro para criar o rascunho` | Payload vacío |

## Endpoint: listar borradores

- **Método y ruta:** `GET /v1/draft`
- **Ítems por página:** 20

| Campo | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `drafts` | body | object | no | Envelope del filtro |
| `drafts.drafts_ids` | body | array | no | Lista de códigos de borrador. **Vacío o ausente retorna todos** |
| `page` | query | integer | no | Página del listado |

> Este `GET` acepta un cuerpo JSON para el filtro. Si tu cliente HTTP no soporta body en `GET`,
> lista todos y filtra de tu lado.

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

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
          "object": "package",
          "type": "Commercial",
          "tax": "sender",
          "insurance": true,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "payment": { "type": "billed" },
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
          "items": [
            {
              "name": "Camiseta algodon",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9,
              "box_id": 1
            }
          ]
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender.country code é obrigatório. O campo address sender.street é obrigatório.",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `codigo` | integer | **Código del borrador.** Úsalo en `PUT /v1/draft/{code}` y en `drafts.drafts_ids` |
| `data` | object | El payload del borrador, ya fusionado con los preajustes de la cuenta |
| `status` | string | Status actual (ver ciclo de vida) |
| `status_descricao` | string | **Cuando está `incomplete`, lista exactamente lo que falta.** Es el campo más importante para corregir el borrador |
| `created_at`, `updated_at` | string | Fechas de creación y última alteración |

## Endpoint: actualizar borrador

- **Método y ruta:** `PUT /v1/draft/{code}`
- **Restricción:** solo borradores con status `incomplete`.
- **Estructura del payload:** idéntica a la del `POST`.

| Parámetro | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `code` | path | integer | **sí** | Código del borrador (`codigo` en el listado) |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "address_sender": {
      "country_code": "BR",
      "postal_code": "09941070",
      "city": "Diadema",
      "street": "Calle Principal",
      "name": "Tienda Ejemplo LTDA",
      "email": "contato@example.com",
      "phone": "11989677465",
      "type": "pj",
      "federal_tax_id": "12345678000199",
      "state_tax_id": "123456789012"
    }
  }'
```

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso."
}
```

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | El borrador está en otro status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | `code` inválido o de otra cuenta |

## Endpoint: eliminar borradores

- **Método y ruta:** `DELETE /v1/draft`

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `drafts` | object | **sí** | Envelope |
| `drafts.drafts_ids` | array | **sí** | Lista de códigos de borrador a eliminar |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Solo se eliminan los borradores de tu propia cuenta. Si ningún código corresponde, la respuesta es
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Reglas de negocio

### Validación en el despacho
Cuando el borrador se despacha, la plataforma aplica un conjunto **rígido** de reglas. Lo que falta aparece
en `status_descricao`. Resumen de los campos exigidos:

| Grupo | Campos exigidos |
|---|---|
| General | `tax`, `insurance` |
| Remitente | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`state`, `state_code` o `postal_code`) |
| Remitente BR | `federal_tax_id` y `state_tax_id` |
| Destinatario | Los mismos campos del remitente |
| Artículos (`object` ≠ `doc`) | `items` con `quantity` y `unit_value`; y `name`, `country_code`, `hscode`, `weight` cuando no hay `sku` |
| Cajas (`object` = `doc`) | `boxes[].weight` y `boxes[].price` |
| Pago | `payment.type` (`billed` o `afterward`) |

### Fabricante
Si envías `manufacturer` **sin** `address_code`, los campos `name`, `country_code`, `state_code`,
`postal_code`, `city`, `street`, `email`, `phone` y `type` pasan a ser obligatorios en el despacho.

Para artículos con `hscode` que empieza en **`64`** (calzado), el `manufacturer.name` es obligatorio.

### Preajustes de la cuenta
Los preajustes se fusionan con el borrador. **El payload siempre vence al preset**, con estas excepciones:
- Informar `address_sender.country_code` descarta el preset de dirección del remitente.
- Si la cuenta tiene el impuesto bloqueado (`tax locked`), el valor se fuerza a `sender`.
- El preset de facturador solo se aplica cuando envías `biller` vacío.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Borrador vacío, no encontrado o no editable | Enviar al menos un campo, verificar el `code` y el status |
| `403` | Sin permiso para la acción | Solicitar el permiso al soporte |
| `422` | Validación de campos | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- El campo **`status_descricao` es tu guía**: dice exactamente lo que falta para que el borrador avance.
- Informa siempre `channel_identifier_number`: es lo que vincula el borrador a tu pedido y permite
  consultar el envío después.
- El borrador **no acepta** pago vía `gateway`. Para ese escenario usa
  [cotización](api-cotacao-v1.md) + [envío](api-remessa-v1.md).
- Para el flujo opuesto (todo validado de una vez, documentos inmediatos), consulta [Manifiesto](api-manifesto.md).
