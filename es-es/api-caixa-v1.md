# Caja

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Mantiene el registro de los embalajes que usas con frecuencia.
- Una vez registrada, la caja puede referenciarse en la cotización por `box_code` o `box_identifier`,
  sin repetir dimensiones y peso en cada solicitud.
- El registro y la eliminación son **asíncronos**: la API confirma la recepción y procesa en segundo plano.

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

## Por qué usar el registro de cajas

**Sin registro** — repitiendo las dimensiones en toda cotización:
```json
"boxes": [{ "name": "Caja S", "height": 20, "width": 20, "depth": 20 }]
```

**Con registro** — referenciando la caja:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Ventajas: payload más pequeño, dimensiones estandarizadas y uso del agrupamiento `smartbox`
(que encaja los artículos en las cajas registradas automáticamente).

## Flujo de uso
1. Registra las cajas con `POST /v1/box`.
2. Espera algunos instantes (procesamiento asíncrono).
3. Confirma con `GET /v1/box`.
4. Usa `box_code` o `box_identifier` en las [cotizaciones](api-cotacao-v1.md).
5. Elimina las cajas obsoletas con `DELETE /v1/box`.

## Endpoint: listar cajas

- **Método y ruta:** `GET /v1/box`

| Parámetro | Lugar | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array | no | - | Lista de `box_code` y/o `box_identifier`. **Vacío o ausente retorna todas** |
| `page` | query | integer | no | `1` | Página del listado |

- **Ítems por página:** 15
- El listado está **cacheado por hasta 5 minutos**. Una caja recién registrada puede tardar ese tiempo en aparecer.

```bash
# Todas las cajas
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'

# Filtrando por identificadores
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "width": "65",
        "height": "65",
        "depth": "65",
        "weight": "2.8",
        "created_at": "2026-01-27T10:05:05.000000Z",
        "updated_at": "2026-01-27T10:05:05.000000Z",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `box_code` | integer | **Código numérico de la caja.** Úsalo en `boxes[].box_code` en la cotización |
| `box_identifier` | string | **Identificador textual de la caja.** Úsalo en `boxes[].box_identifier` en la cotización |
| `name` | string | Nombre de la caja |
| `height`, `width`, `depth` | number | Dimensiones registradas |
| `weight` | number | Peso del embalaje vacío |
| `active` | boolean | Caja activa. **Las cajas inactivas no pueden usarse en la cotización** |
| `created_at`, `updated_at` | string | Fechas de creación y última alteración |

## Endpoint: crear o actualizar cajas

- **Método y ruta:** `POST /v1/box`
- **Comportamiento:** si el `box_identifier` (o `box_code`) ya existe, la caja se **actualiza**.
  En caso contrario, se **crea**.
- **Procesamiento:** asíncrono. La respuesta `200` confirma solo la recepción.

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `boxes` | array | **sí** | 1 ítem | - | Lista de cajas a crear/actualizar |
| `boxes[].box_identifier` | string | **sí** | - | 155 | Identificador único de la caja en tu base (ej.: `CX0236`) |
| `boxes[].name` | string | **sí** | - | 155 | Nombre descriptivo |
| `boxes[].height` | number | **sí** | 1 | - | Altura |
| `boxes[].width` | number | **sí** | 1 | - | Anchura |
| `boxes[].depth` | number | **sí** | 1 | - | Profundidad |
| `boxes[].weight` | number | **sí** | - | - | Peso del embalaje vacío. Hasta 11 dígitos enteros y **3 decimales** |
| `boxes[].activated` | boolean | **sí** | - | - | `true` para disponibilizar la caja en las cotizaciones |
| `boxes[].box_code` | string | no | - | 50 | Código de la caja. Infórmalo para actualizar una caja existente por el código |

> Las dimensiones siguen el sistema de medida de tu cuenta/cotización: **centímetros** en `metric`,
> **pulgadas** en `imperial`. El peso es en **kilogramos** (`metric`) o **libras** (`imperial`).

### Payload mínimo

```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload máximo

```json
{
  "boxes": [
    {
      "box_code": "1",
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    },
    {
      "box_identifier": "CX0237",
      "name": "Caja Master",
      "height": 40,
      "width": 30,
      "depth": 25,
      "weight": 0.85,
      "activated": true
    },
    {
      "box_identifier": "CX0100",
      "name": "Caja descontinuada",
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.2,
      "activated": false
    }
  ]
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {"box_identifier": "CX0236", "name": "Cartucho Individual", "height": 65, "width": 65, "depth": 65, "weight": 2.8, "activated": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

> **El procesamiento es asíncrono.** Confirma el resultado con `GET /v1/box` (recordando el caché de 5 minutos)
> o sigue el informe de importación en el panel.

## Endpoint: eliminar cajas

- **Método y ruta:** `DELETE /v1/box`
- **Procesamiento:** asíncrono. Al final, un informe por e-mail indica cuántas cajas se eliminaron
  y cuáles fallaron.

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `boxes` | object | **sí** | Envelope |
| `boxes.box_ids` | array | **sí** | Lista de `box_code` y/o `box_identifier` a eliminar |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": ["CX0236", 1]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Reglas de la eliminación

| Regla | Resultado cuando falla |
|---|---|
| La caja debe existir en tu base | Error en el informe: `box %s not found!` |
| La caja **no puede tener artículos vinculados** | Error en el informe: `box %s cannot be deleted, has related item` |

> Para "desactivar" una caja que ya tiene artículos vinculados, usa `POST /v1/box` con `activated: false`
> en lugar de eliminarla.

## Cómo usar las cajas en la cotización

| Forma | Payload en `boxes[]` | Cuándo usarla |
|---|---|---|
| Por código | `{ "box_code": 1 }` | Guardaste el código numérico |
| Por identificador | `{ "box_identifier": "CX0236" }` | Usas tu propio código de embalaje |
| Manual | `{ "height": 20, "width": 20, "depth": 20 }` | Embalaje suelto, sin registro |
| Kit | `{ "kit": "KIT-001" }` | La caja **y** los artículos vienen del registro de kit |

Consulta la sección `boxes[]` en [Cotización (v1)](api-cotacao-v1.md#parámetros--boxes).

## Reglas de negocio
- El registro y la eliminación se procesan en **cola asíncrona**.
- El listado está **cacheado por 5 minutos** por usuario y página.
- Las cajas con `activated: false` no aparecen como disponibles en las cotizaciones
  (`A caixa :name não existe ou está desativada!`).
- Las cajas son **por cuenta**: cada usuario ve solo las suyas.
- `box_identifier` debe ser único en tu base — es lo que permite actualizar en lugar de duplicar.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Regla de negocio en el procesamiento | Leer `message` y revisar el payload |
| `403` | Sin permiso para la acción | Solicitar el permiso al soporte |
| `422` | Validación de campos | Corregir los campos listados en `data` (dimensiones ≥ 1, `box_identifier` obligatorio, `activated` booleano) |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Registrar las cajas es el prerrequisito para usar el agrupamiento `smartbox` y el tipo de cotización `items`.
- Mantén los `box_identifier` alineados con tu ERP/WMS: eso simplifica mucho la integración.
- Para registrar los productos que van dentro de las cajas, consulta [Artículos](api-itens-v1.md).
