# Artículos

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Mantiene el registro de los productos que envías, con peso, dimensiones, HS Code y valores.
- Con el artículo registrado, la cotización puede referenciarlo solo por el `sku`, sin repetir todos los datos.
- Es lo que habilita la cotización `type: items` y el agrupamiento automático `smartbox`.
- Todas las operaciones de escritura son **asíncronas**.

> ⚠️ **Importante:** `POST`, `PUT` y `DELETE` **no validan los campos de forma sincrónica**.
> La API responde `200` confirmando la recepción y valida cada artículo durante el procesamiento.
> Los artículos inválidos son **descartados silenciosamente** y el motivo queda registrado en el informe de
> importación, en el panel. Confirma siempre el resultado con `GET /v1/item`.

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

## Por qué usar el registro de artículos

**Sin registro** — repitiendo todo en cada cotización:
```json
"items": [
  { "box_id": 1, "name": "Camiseta algodon S", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Con registro** — referenciando el SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Con `type: items`, ni siquiera necesitas indicar la caja: la plataforma elige entre las
[cajas registradas](api-caixa-v1.md).

## Flujo de uso
1. Registra los artículos con `POST /v1/item`.
2. Espera el procesamiento asíncrono.
3. Confirma con `GET /v1/item`.
4. Usa el `sku` en las [cotizaciones](api-cotacao-v1.md).
5. Actualiza con `PUT /v1/item` y elimina con `DELETE /v1/item`.

## Endpoint: listar artículos

- **Método y ruta:** `GET /v1/item`

| Parámetro | Lugar | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `items[skus][]` | query | array | no | - | Lista de SKUs. **Vacío o ausente retorna todos** |
| `page` | query | integer | no | `1` | Página del listado |

- **Ítems por página:** 15

```bash
# Todos los articulos
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'

# Filtrando por SKU
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Zapatillas",
        "description": "Zapatillas de running para hombre 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "TSH-000-S",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "unit_value": 220.25,
        "selling_unit_value": 330.00,
        "hscode": "950450",
        "is_groupable": true,
        "is_foldable": false,
        "box_code": 3,
        "box_identifier": "CX0236"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

## Endpoint: crear artículos

- **Método y ruta:** `POST /v1/item`
- **Procesamiento:** asíncrono, en lote.

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `sku` | string | **sí** | - | - | Código único del producto en tu base. **No puede repetirse** |
| `name` | string | **sí** | - | - | Nombre del producto (aparece en la invoice) |
| `description` | string | **sí** | 6 | 155 | Descripción del producto. **Descripciones distintas entre artículos evitan alertas aduaneros** |
| `category` | string | **sí** | 6 | 6 | Categoría del producto. **Exactamente 6 caracteres** |
| `country_code` | string | **sí** | 2 | 2 | País de origen/fabricación en ISO 3166-1 alpha-2 |
| `hscode` | string | **sí** | - | - | NCM/HS Code. Define la tabla de impuestos aplicada en el destino |
| `height` | number | **sí** | 0.01 | - | Altura del producto |
| `width` | number | **sí** | 0.01 | - | Anchura del producto |
| `depth` | number | **sí** | 0.01 | - | Profundidad del producto |
| `weight` | number | **sí** | 0.01 | - | Peso unitario |
| `is_groupable` | boolean | **sí** | - | - | `true` permite agrupar el artículo con otros en la misma caja. `false` fuerza una caja exclusiva en el `smartbox` |
| `ean` | string | no | - | - | Código EAN/GTIN |
| `unit_value` | number | no | - | - | Valor unitario predeterminado. Si está ausente en el registro **y** en la cotización, la cotización falla |
| `selling_unit_value` | number | no | - | - | Valor de venta unitario. Necesario cuando la cotización usa `ior: true` |
| `box_code` | integer | no | - | - | Caja predeterminada del artículo ([Caja](api-caixa-v1.md)). Debe ser un **entero** |
| `box_identifier` | string | no | - | - | Caja predeterminada del artículo, por el identificador textual |
| `is_foldable` | boolean | no | - | - | Artículo plegable. En el `smartbox`, los plegables se encajan por volumen; los no plegables por dimensiones |

> **Las dimensiones y el peso** siguen el sistema de medida de la cuenta/cotización: cm/kg en `metric`,
> in/lb en `imperial`.

### Payload mínimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodon S",
      "description": "Camiseta 100% algodon manga corta blanca talla S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "is_groupable": true
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
      "name": "Camiseta algodon S",
      "description": "Camiseta 100% algodon manga corta blanca talla S",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3,
      "box_identifier": "CX0236"
    },
    {
      "sku": "SNK-044",
      "name": "Zapatillas deportivas 44",
      "description": "Zapatillas de running para hombre numero 44 color negro",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
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
      {"sku": "TSH-000-S", "name": "Camiseta algodon S", "description": "Camiseta 100% algodon manga corta blanca talla S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: actualizar artículos

- **Método y ruta:** `PUT /v1/item`
- **Clave de actualización:** el `sku`. Envía el SKU existente y los campos a modificar.
- **Procesamiento:** asíncrono, en lote.
- **Estructura del payload:** idéntica a la del `POST`.

> ⚠️ **Un SKU inexistente no se crea.** La actualización no hace "upsert": si el SKU no existe,
> no ocurre nada (y nada se reporta en la respuesta HTTP). Usa `POST` para crear.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Camiseta algodon S", "description": "Camiseta premium 100% algodon manga corta blanca S", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: eliminar artículos

- **Método y ruta:** `DELETE /v1/item`
- **Procesamiento:** asíncrono, en lote.

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `items` | object | **sí** | Envelope |
| `items.skus` | array | **sí** | Lista de SKUs a eliminar |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Validación asíncrona — tabla de referencia

Como la validación ocurre en el procesamiento, usa esta tabla para garantizar que los artículos entren.
La primera regla violada descarta el artículo.

| Campo | Regla | Motivo registrado en el informe |
|---|---|---|
| (artículo) | Debe ser un objeto | `not_array` |
| `name` | Obligatorio, texto | `name` |
| `description` | Obligatorio, texto | `description` |
| `category` | Obligatorio, texto, **exactamente 6 caracteres** | `category` |
| `country_code` | Obligatorio, texto, **exactamente 2 caracteres** | `country_code` |
| `sku` | Obligatorio, texto | `sku` |
| `ean` | Si se envía, debe ser texto | `ean` |
| `height` | Obligatorio, numérico, **≥ 0.01** | `height` |
| `width` | Obligatorio, numérico, **≥ 0.01** | `width` |
| `depth` | Obligatorio, numérico, **≥ 0.01** | `depth` |
| `weight` | Obligatorio, numérico, **≥ 0.01** | `weight` |
| `hscode` | Obligatorio, texto | `hscode` |
| `is_groupable` | Si se envía, debe ser booleano | `is_groupable` |
| `box_code` | Si se envía, debe ser **entero** (no string) | `box_code` |
| `box_identifier` | Si se envía, debe ser texto | `box_identifier` |
| `unit_value` | Si se envía, debe ser numérico | `unit_value` |
| `selling_unit_value` | Si se envía, debe ser numérico | `selling_unit_value` |

Otras situaciones registradas en el informe:

| Situación | Comportamiento |
|---|---|
| `items` vacío o no es un array | Informe: `Itens isn't an array or is empty` |
| SKU duplicado en el registro | El registro es rechazado (el SKU es índice único) |
| SKU inexistente en el `PUT` | Nada se modifica y nada se crea |
| SKU inválido en el `DELETE` | Informe: `sku_invalid` |

## Cómo usar los artículos en la cotización

| `type` de la cotización | Payload en `items[]` | Comportamiento |
|---|---|---|
| `advanced` | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Tú defines en qué caja va el artículo |
| `items` | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | La plataforma elige las cajas (agrupamiento de la cuenta) |
| cualquiera | `{ "box_id": 1, "name": "...", "country_code": "BR", "hscode": "...", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Artículo suelto, sin registro |

Consulta la sección `items[]` en [Cotización (v1)](api-cotacao-v1.md#parámetros--items).

## Reglas de negocio
- Todas las operaciones de escritura corren en **cola asíncrona**.
- El `sku` es **índice único** por cuenta.
- Los artículos son **por cuenta**: cada usuario ve solo los suyos.
- El registro debe estar completo para que la cotización funcione. Mensajes relacionados:
  - `Nenhum item foi encontrado com o SKU :sku especificado !`
  - `Por favor verifique o cadastro do item :sku, pois poder haver informações imcompletas!`
  - `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).`
  - `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...`
- `is_groupable: false` garantiza una caja exclusiva en el agrupamiento `smartbox`.
- Un `hscode` incorrecto genera un error de impuesto en la cotización:
  `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item).`

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `200` con artículos ausentes | Artículo descartado en la validación asíncrona | Verificar el informe en el panel y la tabla de validación de arriba |
| `403` | Sin permiso para la acción | Solicitar el permiso al soporte |
| `422` | `items` ausente | Enviar el array `items` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Como la validación no es sincrónica, **confirma siempre con `GET /v1/item`** después de registrar o actualizar.
- Mantén descripciones distintas entre productos diferentes: las descripciones repetidas generan el warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` en la cotización.
- Para estandarizar los embalajes usados con estos artículos, consulta [Caja](api-caixa-v1.md).
