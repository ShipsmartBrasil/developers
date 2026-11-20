# Caja

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Mantiene el registro de los embalajes que usas con frecuencia (nombre, dimensiones y peso del embalaje vacío).
- Una vez registrada, la caja puede referenciarse en la cotización por `box_code` o `box_identifier`,
  sin repetir dimensiones y peso en cada solicitud.
- La caja también puede vincularse a un [artículo registrado](api-itens-v1.md) como caja predeterminada del producto.
- El registro/actualización y la eliminación son **asíncronos**: la API valida el formato, confirma la recepción y procesa en segundo plano.
- El listado está **paginado (15 por página)** y se mantiene en caché por hasta **5 minutos**.

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
| `Content-Type` | sí en `POST` y `DELETE` | `application/json` | - |
| `lang` | no | `pt`, `en`, `es` | `pt` |

Cada operación exige un permiso específico en tu perfil (listar, registrar y eliminar cajas).
Sin el permiso, la respuesta es `403` con `Has no permission!`.

## Flujo de uso
1. Registra las cajas con `POST /v1/box`.
2. Espera unos instantes (procesamiento asíncrono). Acompaña el resultado por el informe de importación en el panel.
3. Confirma con `GET /v1/box` y anota el `box_code` generado (recuerda la caché de hasta 5 minutos).
4. Usa `box_code` o `box_identifier` en las [cotizaciones](api-cotacao-v1.md) y en el registro de [artículos](api-itens-v1.md).
5. Elimina las cajas obsoletas con `DELETE /v1/box` (preferentemente por el `box_code`).

## Conceptos esenciales

### Por qué usar el registro de cajas

**Sin registro** — repitiendo las dimensiones en cada cotización:
```json
"boxes": [{ "name": "Caja P", "height": 20, "width": 20, "depth": 20, "weight": 0.3 }]
```

**Con registro** — referenciando la caja:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Ventajas: payload más pequeño, dimensiones estandarizadas y posibilidad de vincular cada artículo registrado a una
caja predeterminada (usado en la agrupación `default` de la cotización `type: items`, exclusiva de la v2).

### `box_code` x `box_identifier`

| Campo | Quién lo define | Tipo en el listado | Uso |
|---|---|---|---|
| `box_code` | **La plataforma** (generado en el registro) | integer | Código numérico de la caja. Forma más confiable de referenciar la caja en cotización, artículo y eliminación |
| `box_identifier` | **Tú** (en el `POST`) | string | Tu propio código de embalaje (ej.: `CX0236`). Debe ser único en tu cuenta |

> **Evita `box_identifier` compuesto solo por números** (ej.: `"123"`). En los filtros del listado y en la eliminación,
> cualquier valor numérico se interpreta como `box_code`. Prefiere identificadores con letras, como `CX0123`.

### Unidades de medida

Las dimensiones y el peso de las cajas registradas siempre se interpretan en **centímetros** y **kilogramos**.
La conversión del sistema `imperial` (pulgadas/libras) hecha en la cotización vale solo para las medidas enviadas
en el propio payload de la cotización — **no** se aplica a los datos provenientes del registro. Registra siempre en cm/kg.

## Endpoint: listar cajas

- **Método y ruta:** `GET /v1/box`
- **Cuándo usar:** para consultar las cajas registradas, descubrir el `box_code` generado y verificar si el registro asíncrono concluyó.

### Parámetros

| Parámetro | Lugar | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array de string/integer | no | - | Lista de `box_code` y/o `box_identifier` (puedes mezclarlos). **Vacío o ausente retorna todas** |
| `page` | query | integer | no | `1` | Página del listado |

- **Elementos por página:** 15 (fijo).
- Los valores numéricos en `boxes[box_ids][]` se buscan como `box_code`; los demás, como `box_identifier`.
- Los identificadores inexistentes simplemente se ignoran (no generan error).

> ⚠️ **Caché de 5 minutos:** el resultado queda en caché por hasta 5 minutos para cada combinación de cuenta y filtro.
> En ese intervalo:
> - las cajas recién registradas, modificadas o eliminadas pueden no reflejarse en el listado;
> - **las solicitudes para páginas diferentes con el mismo filtro pueden retornar la misma página ya consultada**
>   (el número de página no diferencia la caché). Si tienes más de 15 cajas, prefiere consultar por
>   `boxes[box_ids][]` o espera a que expire la caché entre una página y otra.

### Payload mínimo
Sin parámetros (retorna todas las cajas, página 1):
```
GET /v1/box
```

### Payload máximo
```
GET /v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1&page=1
```

### Ejemplo de request

```bash
# Todas las cajas
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'

# Filtrando por identificador y por codigo
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "created_at": "2026-01-27 10:05:05",
        "updated_at": "2026-01-27 10:05:05",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/box?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos de la respuesta

**Caja (`data.data[]`):**

| Campo | Tipo | Descripción |
|---|---|---|
| `box_code` | integer | **Código numérico de la caja**, generado por la plataforma. Úsalo en `boxes[].box_code` en la cotización y en `box_code` del artículo |
| `box_identifier` | string \| null | **Tu identificador textual de la caja**. Úsalo en `boxes[].box_identifier` en la cotización. Puede venir `null` en cajas creadas sin identificador desde el panel |
| `name` | string | Nombre de la caja |
| `height`, `width`, `depth` | number | Dimensiones registradas (cm) |
| `weight` | number | Peso del embalaje vacío (kg) |
| `created_at`, `updated_at` | string | Fechas de creación y última modificación, en el formato `AAAA-MM-DD HH:MM:SS` |
| `active` | boolean | Indica si la caja está activa. Trata también `1`/`0` como verdadero/falso |

**Paginación (`data`):**

| Campo | Tipo | Descripción |
|---|---|---|
| `current_page` | integer | Página actual |
| `from`, `to` | integer \| null | Posición del primer y del último registro de la página |
| `per_page` | integer | Registros por página (15) |
| `total` | integer | Total de registros encontrados |
| `last_page` | integer | Número de la última página |
| `first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url` | string \| null | URL de navegación (mantienen los filtros de la query) |
| `path` | string | URL base del listado |
| `links` | array | Enlaces de navegación listos (`url`, `label`, `active`) |

## Endpoint: crear o actualizar cajas

- **Método y ruta:** `POST /v1/box`
- **Cuándo usar:** para registrar nuevas cajas o modificar cajas existentes, en lote.
- **Comportamiento:** para **actualizar** una caja existente, informa su `box_identifier` (o su `box_code`).
  Si la caja no existe, se **crea**.
- **Procesamiento:** la API valida el formato de todos los campos (error `422` si algo está mal) y, si todo está correcto,
  responde `200` confirmando **solo la recepción**. La grabación ocurre en segundo plano y el resultado queda en el
  informe de importación de cajas, en el panel.

### Parámetros

**Raíz del payload:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `boxes` | array | **sí** | 1 elemento | - | - | Lista de cajas a crear/actualizar. Un array vacío es rechazado |

**Parámetros — `boxes[]`:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `box_identifier` | string | **sí** | - | 155 (usa hasta **50**) | - | Tu identificador único de la caja (ej.: `CX0236`). La validación acepta hasta 155 caracteres, pero los identificadores con más de 50 caracteres **no se graban correctamente** |
| `name` | string | **sí** | - | 155 | - | Nombre descriptivo de la caja |
| `height` | number | **sí** | 1 | - | - | Alto en cm |
| `width` | number | **sí** | 1 | - | - | Ancho en cm |
| `depth` | number | **sí** | 1 | - | - | Profundidad en cm |
| `weight` | number | **sí** | - | 11 dígitos enteros + 3 decimales | - | Peso del embalaje vacío en kg. Formato aceptado: hasta 11 dígitos enteros y hasta **3 decimales**, separador `.`, sin signo (ej.: `0.85`, `2.8`, `12`). Informa un valor mayor que cero |
| `activated` | boolean | **sí** | - | - | - | `true` para caja activa, `false` para inactiva. Acepta también `1`/`0` |
| `box_code` | **string** | no | - | 50 | - | Código de la caja ya existente, para actualizarla por el código. **Envíalo como texto** (ej.: `"1"`): enviado como número, es rechazado con `422` |

> **Atención al tipo de `box_code` en el `POST`:** aquí es **string** (`"1"`). En la cotización, en el registro de artículos y en la
> eliminación, es **integer** (`1`).

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

### Ejemplo de response

**Éxito (`200`):**
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

**Error de validación (`422`):**
```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "boxes.0.height": ["O campo boxes.0.height deve conter um número superior ou igual a 1."],
    "boxes.0.box_code": ["O campo boxes.0.box code deve ser uma string."]
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success`, `warning` (validación/regla) o `error` (inesperado) |
| `message` | string | Mensaje traducido según el header `lang` |
| `data` | object | Solo en errores de validación: mapa `campo` → lista de mensajes. El campo viene con el índice de la caja (`boxes.0.*` = primera caja) |

> **El `200` no garantiza que la caja haya sido grabada.** Confirma el resultado con `GET /v1/box` (recordando la caché de
> hasta 5 minutos) o por el informe de importación en el panel.

## Endpoint: eliminar cajas

- **Método y ruta:** `DELETE /v1/box`
- **Cuándo usar:** para excluir definitivamente cajas que ya no se usan.
- **Procesamiento:** la API valida el formato y responde `200`; la eliminación se hace en segundo plano.
  Al final, recibes **un informe por correo electrónico** (en el correo de la cuenta) con el total de cajas procesadas,
  el total eliminado y la lista de errores.
- La eliminación es **definitiva**. Para solo desactivar, usa `POST /v1/box` con `activated: false`.

### Parámetros

| Campo | Lugar | Tipo | Obligatorio | Mín. | Descripción |
|---|---|---|---|---|---|
| `boxes` | body | object | **sí** | - | Envoltorio |
| `boxes.box_ids` | body | array de integer/string | **sí** | 1 elemento | Lista de cajas a eliminar. **Usa el `box_code` como número entero** (ej.: `1`) |

> ⚠️ **Elimina por el `box_code` (entero).** Aunque la documentación interactiva acepte `box_identifier` en esta lista,
> la eliminación por identificador textual **no es confiable** en este momento:
> - la caja puede ser reportada como `box CX0236 not found!` aunque exista;
> - la verificación de artículos vinculados solo reconoce el vínculo hecho por `box_code` — una caja vinculada a artículos solo
>   por `box_identifier` puede ser eliminada.
>
> Envía también el `box_code` como **número** (`1`) y no como texto (`"1"`): con texto, la verificación de artículos
> vinculados puede no identificar el vínculo. Consulta el `box_code` con `GET /v1/box` antes de eliminar.

### Payload mínimo

```json
{
  "boxes": {
    "box_ids": [1]
  }
}
```

### Payload máximo

```json
{
  "boxes": {
    "box_ids": [1, 2, 3, 15]
  }
}
```

### Ejemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": [1, 2]}}'
```

### Ejemplo de response

**Recibido (`200`):**
```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> A pesar del texto, el mensaje anterior confirma solo la **recepción**. El resultado real llega en el informe por correo electrónico.

**Error inesperado (`500`):**
```json
{
  "status": "error",
  "message": "Algo inesperado aconteceu. Tente novamente mais tarde!"
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success`, `warning` o `error` |
| `message` | string | Mensaje traducido según el header `lang` |
| `data` | object | Solo en errores de validación: mapa `campo` → lista de mensajes |

### Reglas de la eliminación (informe por correo electrónico)

Cada caja de `box_ids` se evalúa individualmente; una falla no impide la eliminación de las demás.

| Regla | Mensaje en el informe cuando falla |
|---|---|
| La caja debe existir en tu cuenta | `Box Error: box %s not found!` (`%s` = valor enviado) |
| La caja **no puede tener artículos vinculados** (`box_code` del artículo) | `Box Error: box %s cannot be deleted, has related item` |

> Para dejar de usar una caja que ya tiene artículos vinculados, cambia la caja predeterminada de esos artículos
> ([`PUT /v1/item`](api-itens-v1.md#endpoint-actualizar-artículos)) antes de eliminarla, o simplemente desactiva la caja
> con `POST /v1/box` y `activated: false`.

## Cómo usar las cajas en la cotización

| Forma | Payload en `boxes[]` | Cuándo usar |
|---|---|---|
| Por código | `{ "box_code": 1 }` | Guardaste el código numérico (integer) |
| Por identificador | `{ "box_identifier": "CX0236" }` | Usas tu propio código de embalaje |
| Manual | `{ "height": 20, "width": 20, "depth": 20, "weight": 0.3 }` | Embalaje suelto, sin registro |
| Kit | `{ "kit": "KIT-001" }` | La caja **y** los artículos provienen del registro de kit |

Usa **una** forma por caja (no mezcles `box_code` con `box_identifier` o con dimensiones manuales en la misma posición).
Una caja referenciada que no existe genera error en la cotización:

| Situación | Status | Mensaje |
|---|---|---|
| `box_code`/`box_identifier` inexistente en la plataforma | `422` | `O valor selecionado para o campo boxes.0.box code é inválido.` (o `box identifier`) |
| La caja no pertenece a tu cuenta | `400` | `A caixa :name não foi encontrada para o usuário especificado!` |

Uso de las cajas registradas según la agrupación configurada en la cuenta (cotización `type: items`, exclusiva de la v2):

| Agrupación | Cómo se usan las cajas registradas |
|---|---|
| `default` | Cada artículo registrado debe tener una caja predeterminada (`box_code` o `box_identifier` en el [registro del artículo](api-itens-v1.md)). Sin caja vinculada: `A caixa com o box_code/box_identifier :name não existe ou está desativada` (HTTP `404`) |
| `smartbox` | Usa las cajas informadas con dimensiones en `boxes[]` en el payload de la cotización. Sin caja válida: `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (HTTP `400`) |
| `all_in_one` | Consolida los artículos; no depende de la caja predeterminada del artículo |

Consulta la sección `boxes[]` en [Cotización (v1)](api-cotacao-v1.md#parámetros--boxes).

## Reglas de negocio
- La creación, actualización y eliminación se procesan en **segundo plano**; el `200` confirma solo la recepción.
- El listado queda en **caché por hasta 5 minutos** por cuenta y filtro (consulta la alerta sobre paginación más arriba).
- Las cajas son **por cuenta**: cada usuario lista, usa y elimina solo las suyas.
- `box_identifier` debe ser único en tu cuenta — es lo que permite actualizar en lugar de duplicar. Usa como máximo 50 caracteres y evita valores solo numéricos.
- Las dimensiones y el peso registrados siempre están en **cm** y **kg**, independientemente del `measurement` de la cotización.
- El indicador `active` es informativo en el listado: **no referencies cajas inactivas** en cotizaciones. En este momento, la cotización
  no bloquea de forma garantizada una caja inactiva referenciada por `box_code`/`box_identifier`.
- Una caja con artículos vinculados (por el `box_code` del artículo) no se elimina.
- La eliminación es definitiva y se reporta por correo electrónico.

## Errores comunes

| Status | Significado/Mensaje | Cómo corregir |
|---|---|---|
| `403` | `Has no permission!` — el perfil no tiene permiso para listar, registrar o eliminar cajas | Solicitar el permiso al soporte |
| `422` | `O campo boxes é obrigatório.` — `boxes` ausente o vacío (`POST`/`DELETE`) | Enviar al menos una caja (`POST`) o el envoltorio `boxes` (`DELETE`) |
| `422` | `O campo boxes deve conter um array.` — `boxes` no es lista/objeto | Corregir la estructura del payload |
| `422` | `O campo boxes.0.box identifier é obrigatório.` / `O campo boxes.0.name é obrigatório.` | Informar `box_identifier` y `name` en todas las cajas |
| `422` | `O campo boxes.0.box identifier não pode conter mais de 155 caracteres.` / `O campo boxes.0.name não pode conter mais de 155 caracteres.` | Acortar el valor (para `box_identifier`, usa hasta 50) |
| `422` | `O campo boxes.0.height deve conter um número superior ou igual a 1.` (ídem `width`/`depth`) | Informar dimensiones ≥ 1 cm |
| `422` | `O campo boxes.0.height deve conter um valor numérico.` (ídem `width`/`depth`/`weight`) | Enviar números, sin unidad en el valor |
| `422` | `O formato do valor informado no campo boxes.0.weight é inválido.` | Usar hasta 3 decimales, separador `.` y sin signo |
| `422` | `O campo boxes.0.activated deve conter o valor verdadeiro ou falso.` | Enviar `true`/`false` |
| `422` | `O campo boxes.0.box code deve ser uma string.` | En el `POST`, enviar `box_code` como texto (`"1"`) |
| `422` | `O campo boxes.0.box code não pode conter mais de 50 caracteres.` | Verificar el `box_code` |
| `422` | `O campo boxes.box ids é obrigatório.` / `O campo boxes.box ids deve conter um array.` (`DELETE`; en el `GET`, solo la segunda) | Enviar `box_ids` como lista no vacía |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Repetir con intervalo creciente; si persiste, contactar al soporte |
| Informe (correo) | `Box Error: box %s not found!` | Verificar el `box_code` con `GET /v1/box` |
| Informe (correo) | `Box Error: box %s cannot be deleted, has related item` | Desvincular la caja de los artículos o simplemente desactivarla |

> En los mensajes de validación, el índice (`boxes.0`, `boxes.1`...) indica la posición de la caja en el array, comenzando en `0`,
> y el `_` del nombre del campo aparece como espacio (`box identifier`).

## Observaciones finales
- Registrar las cajas y vincularlas a los artículos es el requisito previo para la [cotización por artículos de la v2](api-cotacao-v2.md#cotización-de-artículos) (`type: items`) con agrupación `default`.
- Mantén los `box_identifier` alineados con tu ERP/WMS: esto simplifica mucho la integración.
- Guarda el `box_code` retornado en el listado: es la referencia más confiable para cotización, artículos y eliminación.
- Para registrar los productos que van dentro de las cajas, consulta [Artículos](api-itens-v1.md).
- Para referenciar las cajas en la cotización, consulta [Cotización (v1)](api-cotacao-v1.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
