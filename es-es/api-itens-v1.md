# Artículos

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Mantiene el registro de los productos que envías, con peso, dimensiones, HS Code, valores e identificadores (SKU, EAN y NSMP).
- Con el artículo registrado, la cotización puede referenciarlo solo por el `sku`, sin repetir todos los datos.
- Es lo que habilita la [cotización por artículos](api-cotacao-v2.md#cotización-de-artículos) (`type: items`, **exclusiva de la v2**) y las agrupaciones automáticas de cajas de la cuenta.
- Todas las operaciones de escritura (`POST`, `PUT`, `DELETE`) son **asíncronas** y se validan solo en el procesamiento.
- El listado está **paginado (20 por página)** y refleja el registro en tiempo real (sin caché).

> ⚠️ **Importante:** `POST`, `PUT` y `DELETE` **no validan los campos de forma síncrona**.
> La API responde `200` confirmando la recepción y valida cada artículo durante el procesamiento.
> Los artículos inválidos se **descartan silenciosamente** (sin error en la respuesta HTTP) y el motivo queda registrado en el
> informe de importación de artículos, en el panel. Confirma siempre el resultado con `GET /v1/item`.

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

| Header | Obligatorio | Valores | Predeterminado |
|---|---|---|---|
| `Authorization` | **sí** | `Bearer SU_TOKEN` | - |
| `Content-Type` | sí en `POST`, `PUT` y `DELETE` | `application/json` | - |
| `lang` | no | `pt`, `en`, `es` | `pt` |

Cada operación exige un permiso específico en tu perfil (listar, registrar/actualizar y eliminar artículos).
Sin el permiso, la respuesta es `403` con `Has no permission!`.

## Flujo de uso
1. (Opcional) Registra las [cajas](api-caixa-v1.md) y anota el `box_code` para vincularlo a los artículos.
2. Registra los artículos con `POST /v1/item`.
3. Espera el procesamiento asíncrono y revisa el informe de importación en el panel.
4. Confirma con `GET /v1/item` que todos los SKU fueron registrados.
5. Usa el `sku` en las [cotizaciones](api-cotacao-v1.md).
6. Actualiza con `PUT /v1/item` y elimina con `DELETE /v1/item`.

## Conceptos esenciales

### Por qué usar el registro de artículos

**Sin registro** — repitiendo todo en cada cotización:
```json
"items": [
  { "box_id": 1, "name": "Camiseta algodon P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Con registro** — referenciando el SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Con `type: items` (**exclusivo de la [Cotización v2](api-cotacao-v2.md#cotización-de-artículos)**), ni siquiera necesitas ubicar el artículo en una caja: la plataforma arma las cajas según la
agrupación configurada en la cuenta (consulta [Cómo usar los artículos en la cotización](#cómo-usar-los-artículos-en-la-cotización)).

### Identificadores del producto

| Campo | Qué es | Uso |
|---|---|---|
| `sku` | Tu código interno del producto | **Clave del registro**: único por cuenta, usado para referenciar, actualizar y eliminar |
| `ean` | Código de barras EAN/GTIN | Impreso en la invoice y enviado como identificador estándar del fabricante en la información regulatoria |
| `nsmp` | *Non Standard Manufacturer Product Id* — identificador de producto **no estandarizado** del fabricante (ej.: el código de pieza/modelo del fabricante) | Impreso en la invoice (línea `Mfr:`) y enviado en la información regulatoria exigida por algunos transportistas para destinos en la **Unión Europea** |

### Unidades de medida

Las dimensiones y el peso del registro siempre se interpretan en **centímetros** y **kilogramos**.
La conversión del sistema `imperial` hecha en la cotización vale solo para las medidas enviadas en el payload de la cotización —
**no** se aplica a los datos completados a partir del registro. Registra siempre en cm/kg.

### Tipos de datos (el procesamiento es estricto)

La validación asíncrona verifica el **tipo JSON** de cada campo. Los valores con el tipo incorrecto descartan el artículo:

| Campo | Envíalo como | Ejemplo correcto | Ejemplo descartado |
|---|---|---|---|
| `sku`, `hscode`, `ean`, `nsmp`, `category` | texto (entre comillas) | `"hscode": "610910"` | `"hscode": 610910` |
| `box_code` | número entero | `"box_code": 3` | `"box_code": "3"` o `3.0` |
| `height`, `width`, `depth`, `weight`, `unit_value`, `selling_unit_value` | número | `"weight": 0.25` | `"weight": "0,25"` |
| `is_groupable`, `is_foldable` | booleano JSON | `"is_groupable": false` | `"is_groupable": "false"` (se graba, pero se trata como `true` en la cotización) |

## Endpoint: listar artículos

- **Método y ruta:** `GET /v1/item`
- **Cuándo usar:** para consultar el registro, verificar si la importación asíncrona concluyó y revisar los datos de un SKU.

### Parámetros

| Parámetro | Lugar | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `items[skus][]` | query | array de string | no | - | Lista de SKU (coincidencia exacta, distingue mayúsculas/minúsculas). **Vacío o ausente retorna todos** |
| `page` | query | integer | no | `1` | Página del listado |

- **Elementos por página:** 20 (fijo).
- Los SKU inexistentes se ignoran (no generan error).
- Sin caché: los cambios procesados aparecen inmediatamente.

### Payload mínimo
Sin parámetros (retorna todos los artículos, página 1):
```
GET /v1/item
```

### Payload máximo
```
GET /v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044&page=1
```

### Ejemplo de request

```bash
# Todos los articulos
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'

# Filtrando por SKU
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

> El `--globoff` evita que curl interprete los corchetes de la query string.

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "sku": "TSH-000-S",
        "name": "Camiseta algodon P",
        "description": "Camiseta 100% algodon manga corta blanca talla P",
        "category": "610910",
        "country_code": "BR",
        "hscode": "610910",
        "ean": "7894900010015",
        "nsmp": "FAB-TSH-P-001",
        "height": 2,
        "width": 20,
        "depth": 25,
        "weight": 0.25,
        "is_groupable": true,
        "is_foldable": true,
        "box_code": 3,
        "box_identifier": "CX0236",
        "unit_value": 29.9,
        "selling_unit_value": 49.9
      },
      {
        "sku": "SNK-044",
        "name": "Zapatilla deportiva 44",
        "description": "Zapatilla de running masculina numero 44 color negro",
        "category": "640411",
        "country_code": "BR",
        "hscode": "640411",
        "height": 15,
        "width": 22,
        "depth": 33,
        "weight": 0.8,
        "is_groupable": false,
        "nsmp": null,
        "unit_value": null,
        "selling_unit_value": null
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/item?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 20,
    "prev_page_url": null,
    "to": 2,
    "total": 2
  }
}
```

### Campos de la respuesta

**Artículo (`data.data[]`):**

| Campo | Tipo | Siempre presente | Descripción |
|---|---|---|---|
| `sku` | string | sí | Código único del producto |
| `name` | string | sí | Nombre del producto |
| `description` | string | sí | Descripción del producto |
| `category` | string | sí | Categoría (6 caracteres) |
| `country_code` | string | sí | País de origen/fabricación |
| `hscode` | string | sí | NCM/HS Code |
| `height`, `width`, `depth` | number | sí | Dimensiones (cm) |
| `weight` | number | sí | Peso unitario (kg) |
| `nsmp` | string \| null | **sí** | Identificador no estandarizado del fabricante. Viene `null` cuando no está registrado |
| `unit_value` | number \| null | **sí** | Valor unitario predeterminado. `null` cuando no está registrado |
| `selling_unit_value` | number \| null | **sí** | Valor de venta unitario. `null` cuando no está registrado |
| `ean` | string | no | Código EAN/GTIN (solo aparece si está registrado) |
| `is_groupable` | boolean | no | Permite agrupar con otros artículos (solo aparece si está registrado) |
| `is_foldable` | boolean | no | Artículo plegable (solo aparece si está registrado) |
| `box_code` | integer | no | Caja predeterminada vinculada (solo aparece si está registrada) |
| `box_identifier` | string | no | Caja predeterminada vinculada, por el identificador (solo aparece si está registrada) |

> Los campos se devuelven tal como fueron grabados (mismo tipo y valor enviados). El orden de las claves puede variar.
> Los campos fuera de la lista documentada que se envíen en el registro también se graban y pueden aparecer en el listado:
> **envía solo los campos documentados**.

**Paginación (`data`):** mismos campos del [listado de cajas](api-caixa-v1.md#campos-de-la-respuesta)
(`current_page`, `from`, `to`, `per_page` = 20, `total`, `last_page`, URL de navegación, `path` y `links`).

## Endpoint: crear artículos

- **Método y ruta:** `POST /v1/item`
- **Cuándo usar:** para registrar nuevos productos, uno o varios por solicitud.
- **Procesamiento:** asíncrono, en lote. Cada artículo se valida individualmente; los válidos se graban y los inválidos
  se descartan y se listan en el informe de importación.
- **Un SKU ya existente no se actualiza:** usa `PUT` para modificarlo.

### Parámetros

**Raíz del payload:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `items` | array | **sí** | Lista de artículos. Sin `items`, no se graba nada (la respuesta sigue siendo `200`) |

**Parámetros — `items[]`:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `sku` | string | **sí** | 1 | 50 (recomendado) | - | Código único del producto en tu cuenta. **No puede repetirse.** La cotización acepta SKU de hasta 50 caracteres |
| `name` | string | **sí** | 1 | - | - | Nombre del producto (aparece en la invoice) |
| `description` | string | **sí** | 1 (6 recomendado) | 155 (recomendado) | - | Descripción del producto. **Descripciones distintas entre artículos evitan alertas aduaneras** |
| `category` | string | **sí** | 6 | 6 | - | Categoría del producto. **Exactamente 6 caracteres** (sin acentos: las letras acentuadas cuentan como 2) |
| `country_code` | string | **sí** | 2 | 2 | - | País de origen/fabricación en ISO 3166-1 alpha-2, en mayúsculas (ej.: `BR`) |
| `hscode` | string | **sí** | 1 | - | - | NCM/HS Code, **como texto**. Define la tabla de impuestos aplicada en el destino |
| `height` | number | **sí** | 0.01 | - | - | Alto del producto (cm) |
| `width` | number | **sí** | 0.01 | - | - | Ancho del producto (cm) |
| `depth` | number | **sí** | 0.01 | - | - | Profundidad del producto (cm) |
| `weight` | number | **sí** | 0.01 | - | - | Peso unitario (kg) |
| `is_groupable` | boolean | no (**recomendado**) | - | - | `true` | `true` permite agrupar el artículo con otros en la misma caja; `false` exige caja exclusiva. En la agrupación `default`, un artículo sin este campo causa error en la cotización |
| `ean` | string | no | - | 50 (recomendado) | - | Código EAN/GTIN, **como texto** |
| `nsmp` | string | no | - | 50 (recomendado) | `null` | *Non Standard Manufacturer Product Id*: identificador no estandarizado del fabricante (ej.: código de modelo/pieza). Impreso en la invoice y usado en la información regulatoria de algunos transportistas para destinos en la Unión Europea. La cotización acepta hasta 50 caracteres |
| `unit_value` | number | no | - | - | - | Valor unitario predeterminado. Si está ausente en el registro **y** en la cotización, la cotización falla |
| `selling_unit_value` | number | no | - | - | - | Valor de venta unitario. Necesario cuando la cotización usa `ior: true` |
| `box_code` | integer | no | - | - | - | Caja predeterminada del artículo ([Caja](api-caixa-v1.md)). Debe ser un **entero** JSON |
| `box_identifier` | string | no | - | 50 | - | Caja predeterminada del artículo, por el identificador textual |
| `is_foldable` | boolean | no | - | - | `false` | Artículo plegable. En el `smartbox`, los plegables se encajan por volumen; los no plegables, por dimensiones |

> **Caja predeterminada (Forma A / Forma B):** vincula la caja por `box_code` (Forma A) **o** por `box_identifier` (Forma B).
> No envíes los dos con valores de cajas diferentes. La existencia de la caja **no** se verifica en el registro del artículo —
> solo se verifica en la cotización. Prefiere `box_code`: es el vínculo reconocido en la protección contra la eliminación de cajas.

> Los valores `null` en `unit_value` y `selling_unit_value` se ignoran (el campo no se graba).

### Payload mínimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta 100% algodon manga corta blanca talla P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25
    }
  ]
}
```

> Este es el payload más pequeño aceptado. Para usar el artículo en la cotización `type: items` con agrupación `default`, incluye también
> `is_groupable` y la caja predeterminada (`box_code` o `box_identifier`).

### Payload máximo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta 100% algodon manga corta blanca talla P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-001",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Zapatilla deportiva 44",
      "description": "Zapatilla de running masculina numero 44 color negro",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": "FAB-SNK-44-BLK",
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Camiseta algodon P", "description": "Camiseta 100% algodon manga corta blanca talla P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-001", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true, "box_code": 3}
    ]
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` (recibido) o `error` (falla inesperada) |
| `message` | string | Mensaje traducido según el header `lang` |

> El `200` confirma solo la recepción. Los artículos descartados aparecen en el informe de importación (consulta
> [Validación asíncrona](#validación-asíncrona--tabla-de-referencia)).

## Endpoint: actualizar artículos

- **Método y ruta:** `PUT /v1/item`
- **Cuándo usar:** para modificar datos de productos ya registrados, en lote.
- **Clave de actualización:** el `sku`. El artículo registrado con ese SKU recibe los valores enviados.
- **Procesamiento:** asíncrono, en lote, con **la misma validación del `POST`**.

> ⚠️ **Envía el artículo completo.** La validación del `PUT` es idéntica a la del `POST`: `sku`, `name`, `description`, `category`,
> `country_code`, `hscode`, `height`, `width`, `depth` y `weight` son **obligatorios en toda actualización**. Un artículo
> enviado solo con el SKU y el campo modificado se **descarta** (motivo `name` en el informe).

> ⚠️ **Un SKU inexistente no se crea.** La actualización no hace "upsert": si el SKU no existe, no pasa nada
> (y no se reporta nada). Usa `POST` para crear.

Reglas de actualización de los campos opcionales:

| Situación en el payload del `PUT` | Resultado en el registro |
|---|---|
| Campo opcional **omitido** | Mantiene el valor actual |
| Campo opcional enviado con valor | Sustituye el valor actual |
| `ean`, `nsmp`, `box_code`, `box_identifier`, `is_foldable` enviados como `null` | El valor se borra (pasa a `null`) |
| `unit_value` o `selling_unit_value` enviados como `null` | **Ignorado**: se mantiene el valor actual (no es posible borrar el precio con el `PUT`; envía un nuevo valor) |

### Parámetros

Misma estructura y mismas reglas del [`POST /v1/item`](#endpoint-crear-artículos): raíz `items` (array) y los campos de `items[]`.

### Payload mínimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta premium 100% algodon manga corta blanca P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27
    }
  ]
}
```

### Payload máximo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta premium 100% algodon manga corta blanca P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-002",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27,
      "unit_value": 34.9,
      "selling_unit_value": 54.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Zapatilla deportiva 44",
      "description": "Zapatilla de running masculina numero 44 color negro",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": null,
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 189.9,
      "selling_unit_value": 339.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

> En el segundo artículo, `"nsmp": null` borra el NSMP registrado.

### Ejemplo de request

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Camiseta algodon P", "description": "Camiseta premium 100% algodon manga corta blanca P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-002", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` (recibido) o `error` (falla inesperada) |
| `message` | string | Mensaje traducido según el header `lang` |

## Endpoint: eliminar artículos

- **Método y ruta:** `DELETE /v1/item`
- **Cuándo usar:** para excluir definitivamente productos del registro.
- **Procesamiento:** asíncrono, en lote. Los SKU inexistentes se ignoran sin aviso.

### Parámetros

| Campo | Lugar | Tipo | Obligatorio | Mín. | Descripción |
|---|---|---|---|---|---|
| `items` | body | object | **sí** | - | Envoltorio |
| `items.skus` | body | array de string | **sí** | 1 elemento | Lista de SKU a eliminar. Cada SKU debe ser texto no vacío |

> La ausencia de `items.skus` no genera error en la respuesta HTTP (`200`); el problema se registra en el informe.

### Payload mínimo

```json
{
  "items": {
    "skus": ["TSH-000-S"]
  }
}
```

### Payload máximo

```json
{
  "items": {
    "skus": ["TSH-000-S", "TSH-000-M", "SNK-044"]
  }
}
```

### Ejemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` (recibido) o `error` (falla inesperada) |
| `message` | string | Mensaje traducido según el header `lang` |

## Validación asíncrona — tabla de referencia

Como la validación ocurre en el procesamiento (`POST` y `PUT`), usa esta tabla para garantizar que los artículos sean registrados.
Las reglas se verifican **en este orden** y la **primera** regla violada descarta el artículo.
En el informe de importación, cada artículo descartado aparece como:

```
sku => TSH-000-S, status => invalid, reason => category
```

| Orden | Campo | Regla | `reason` en el informe |
|---|---|---|---|
| 1 | (artículo) | Debe ser un objeto | `not_array` |
| 2 | `name` | Obligatorio, texto no vacío | `name` |
| 3 | `description` | Obligatorio, texto no vacío | `description` |
| 4 | `category` | Obligatorio, texto, **exactamente 6 caracteres** | `category` |
| 5 | `country_code` | Obligatorio, texto, **exactamente 2 caracteres** | `country_code` |
| 6 | `sku` | Obligatorio, texto no vacío | `sku` |
| 7 | `ean` | Si se envía (no nulo), debe ser texto | `ean` |
| 8 | `nsmp` | Si se envía (no nulo), debe ser texto | `nsmp` |
| 9 | `height` | Obligatorio, numérico, **≥ 0.01** | `height` |
| 10 | `width` | Obligatorio, numérico, **≥ 0.01** | `width` |
| 11 | `depth` | Obligatorio, numérico, **≥ 0.01** | `depth` |
| 12 | `weight` | Obligatorio, numérico, **≥ 0.01** | `weight` |
| 13 | `hscode` | Obligatorio, texto no vacío | `hscode` |
| 14 | `box_code` | Si se envía (no nulo), debe ser **entero** (no string, no decimal) | `box_code` |
| 15 | `box_identifier` | Si se envía (no nulo), debe ser texto | `box_identifier` |
| 16 | `unit_value` | Si se envía (no nulo), debe ser numérico | `unit_value` |
| 17 | `selling_unit_value` | Si se envía (no nulo), debe ser numérico | `selling_unit_value` |

Reglas aplicadas silenciosamente (sin descarte y sin aviso):

| Situación | Comportamiento |
|---|---|
| `is_groupable` con cualquier valor | Aceptado y grabado tal como se envió. Usa booleano JSON: `"false"` en texto se trata como `true` en la cotización |
| `is_foldable` con cualquier valor | No se valida; se graba tal como se envió |
| `name`, `description`, `sku`, `ean`, `nsmp` por encima de los límites recomendados | Grabados, pero pueden generar error cuando el artículo se use en la cotización/envío |
| `country_code` con 2 caracteres que no es un país válido | Grabado; el error aparece solo en la cotización |
| `box_code`/`box_identifier` de caja inexistente | Grabado; el error aparece solo en la cotización |
| Campos no documentados | Grabados tal como se enviaron |

Otras situaciones registradas en el informe:

| Operación | Situación | Comportamiento |
|---|---|---|
| `POST`/`PUT` | Cuerpo vacío (`{}`) | Informe: `Itens isn't an array or is empty` |
| `POST`/`PUT` | `items` ausente con otros campos en el cuerpo, o `items` no es lista | El procesamiento falla y se registra como falla en el informe; no se graba nada |
| `POST`/`PUT` | `items: []` | No se graba nada y no se reporta nada |
| `POST` | SKU ya registrado, o repetido en el mismo lote | El artículo duplicado es rechazado; los demás artículos válidos del lote se graban y el lote se marca como falla en el informe |
| `PUT` | SKU inexistente | No se modifica nada, no se crea nada y no se reporta nada |
| `DELETE` | `items.skus` ausente o vacío | Informe: `SKUs isn't an array or is empty` |
| `DELETE` | SKU vacío o que no es texto (ej.: número) | Informe: `sku => ..., status => invalid, reason => sku_invalid` |
| `DELETE` | SKU inexistente | Ignorado sin aviso |

## Cómo usar los artículos en la cotización

| Cotización `type` | Payload en `items[]` | Comportamiento |
|---|---|---|
| `advanced` (v1 y v2) | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Tú defines en qué caja (posición en `boxes[]`) va el artículo |
| `items` (**solo v2**) | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | La plataforma arma las cajas según la agrupación de la cuenta |
| cualquiera | `{ "box_id": 1, "name": "Camiseta algodon P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Artículo suelto, sin registro |

Cuando el artículo de la cotización tiene `sku` y falta algún dato (nombre, país, HS Code, dimensiones, peso, valores o `nsmp`),
los datos ausentes se completan a partir del registro. Los campos enviados en la cotización tienen prioridad sobre el registro.

Uso del registro según la agrupación de la cuenta (cotización `type: items`, v2):

| Agrupación | Requisitos del registro | `is_groupable` |
|---|---|---|
| `default` | Cada artículo debe tener caja predeterminada (`box_code` o `box_identifier`) e `is_groupable` registrados | `true`: agrupado con otros artículos de la misma caja predeterminada. `false`: una caja exclusiva por unidad |
| `smartbox` | Las cajas candidatas provienen de `boxes[]` en el payload de la cotización | Ausente = `true`. `false`: caja exclusiva. `is_foldable` (ausente = `false`) define el encaje por volumen o por dimensiones |
| `all_in_one` | Solo los datos del artículo | Ausente = `true`. `false`: el artículo queda fuera de la consolidación, en caja propia |

Consulta la sección `items[]` en [Cotización (v1)](api-cotacao-v1.md#parámetros--items).

## Reglas de negocio
- Todas las operaciones de escritura se ejecutan en **segundo plano**; el `200` confirma solo la recepción.
- El `sku` es **único** por cuenta: el `POST` no sobrescribe un SKU existente.
- Los artículos son **por cuenta**: cada usuario ve y modifica solo los suyos.
- El `PUT` exige el artículo completo (misma validación del `POST`) y no crea SKU nuevos.
- Las dimensiones y el peso del registro siempre están en **cm** y **kg**.
- `nsmp` es opcional; cuando se completa, se imprime en la invoice y se envía en la información regulatoria de algunos
  transportistas para destinos en la Unión Europea. Sin `nsmp`, ese dato sigue vacío.
- El registro debe estar completo para que la cotización funcione. Mensajes relacionados en la cotización:

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | SKU no registrado y artículo sin los datos mínimos en el payload |
| `404` | `O item com o sku :name não existe!` | SKU no registrado en la cotización `type: items` |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Artículo sin caja predeterminada válida en la agrupación `default` |
| `400` | `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).` | Sin `unit_value` en el registro ni en la cotización |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | Valor en cero |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | `hscode` incorrecto para el destino |

> Los códigos HTTP de los mensajes de cotización anteriores siguen la página [Cotización (v1)](api-cotacao-v1.md#errores-comunes).

## Errores comunes

| Status | Significado/Mensaje | Cómo corregir |
|---|---|---|
| `200` con artículos ausentes | Artículo descartado en la validación asíncrona (`reason => campo` en el informe) | Revisar el informe en el panel y la [tabla de validación](#validación-asíncrona--tabla-de-referencia) |
| `200` sin efecto en el `PUT` | SKU inexistente o artículo incompleto | Crear con `POST` o enviar el artículo completo |
| `403` | `Has no permission!` — perfil sin permiso para listar, registrar/actualizar o eliminar artículos | Solicitar el permiso al soporte |
| `422` | `O campo items.skus deve conter um array.` (solo `GET`) | Enviar `items[skus][]` como lista en la query string |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Repetir con intervalo creciente; si persiste, contactar al soporte |
| Informe | `Itens isn't an array or is empty` | Enviar `items` con al menos un artículo |
| Informe | `SKUs isn't an array or is empty` | En el `DELETE`, enviar `items.skus` con al menos un SKU |
| Informe | `sku => ..., status => invalid, reason => sku_invalid` | En el `DELETE`, enviar los SKU como texto no vacío |

> `POST`, `PUT` y `DELETE` **no retornan `422`**: los problemas de campos aparecen solo en el informe de importación.

## Observaciones finales
- Como la validación no es síncrona, **confirma siempre con `GET /v1/item`** después de registrar, actualizar o eliminar.
- Envía los tipos JSON correctos: `sku`, `hscode`, `ean` y `nsmp` como texto; `box_code` como entero; booleanos sin comillas.
- Mantén descripciones distintas entre productos diferentes: las descripciones repetidas generan el warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` en la cotización.
- Completa `nsmp` para productos enviados a la Unión Europea siempre que el fabricante tenga un código propio del producto.
- Para estandarizar los embalajes usados con estos artículos, consulta [Caja](api-caixa-v1.md).
- Para usar los artículos en la cotización, consulta [Cotización (v1)](api-cotacao-v1.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
