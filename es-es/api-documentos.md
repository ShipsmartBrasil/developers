# Documentos del Envío

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Recupera los documentos generados por la plataforma y por el transportista después del procesamiento del envío:
  AWB (etiqueta), invoice, packing list, declaración y otros.
- También devuelve los **adjuntos** que enviaste en la creación/actualización del envío.
- Cada documento viene con una `url` para descarga y, opcionalmente, con el contenido en `base64`.
- Las cuentas con permiso específico reciben también los **documentos incorporados** (todos los PDF en un único archivo y todas las etiquetas ZPL en un único contenido).
- Funciona para envíos creados en la **v1 y en la v2** (las rutas de documentos existen solo en la v1).

## URL base
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
lang: pt
```

| Header | Obligatorio | Valores | Predeterminado | Observación |
|---|---|---|---|---|
| `Authorization` | **sí** | `Bearer <token>` | - | Token de tu cuenta. El perfil necesita el permiso de consulta de documentos de envío |
| `lang` | no | `pt`, `en`, `es` | `pt` | Idioma de los mensajes |

## Flujo de uso
1. Crea el envío ([Envío v1](api-remessa-v1.md) o [Envío v2](api-remessa-v2.md)).
2. Consulta el [status](api-rastreio.md) hasta que el envío aparezca en el grupo `shipped` — es en ese momento que
   los documentos pasan a existir.
3. Consulta los documentos por `{shipping}` o por el canal.
4. Descarga el archivo por la `url` **inmediatamente** o decodifica el `base64`, y almacénalo de tu lado.

> **Antes de `shipped` la consulta puede retornar `400 Nenhum documento encontrado com o código especificado!`.**
> Esto es esperado: espera el procesamiento en el transportista. Si el envío tiene adjuntos, la respuesta puede ser
> `200` trayendo solo los adjuntos (consulta [Reglas de negocio](#reglas-de-negocio)).

## Conceptos esenciales

### Tipos de documento filtrables (`documents`)

| Valor en `documents` | Clave en `data.docs` | Contenido |
|---|---|---|
| `awb_pdf` | `AWB` | Etiqueta/AWB en PDF |
| `awb_zpl` | `AWB_ZPL` y `AWB_ZPL_UPS` | Etiqueta/AWB en ZPL (impresora térmica). Para algunos transportistas la clave es `AWB_ZPL_UPS` |
| `awb_prc_pdf` | `AWBPRC` | Etiqueta AWB PRC en PDF, emitida para algunos transportistas |
| `awb_prc_zpl` | `AWBPRCZPL` | Etiqueta AWB PRC en ZPL |
| `awb_prc_fedex_pdf` | `AWBPRCFedex` | Etiqueta AWB PRC FedEx en PDF |
| `awb_prc_fedex_zpl` | `AWBPRCFedexZPL` | Etiqueta AWB PRC FedEx en ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice en PDF |
| `packing_list_pdf` | `PackingList` | Packing list en PDF |
| `declaration_pdf` | `Declaration` | Declaración de contenido en PDF |
| `non_dangerous_pdf` | `NonDangerousObject` | Declaración de mercancía no peligrosa en PDF |

> **Las claves de `data.docs` no tienen espacios** (`PackingList`, `NonDangerousObject`, `AWBPRC`...), aunque la
> documentación interactiva las muestre con espacios (`Packing List`, `Non-Dangerous Object`, `AWB PRC`...).
> Programa la lectura por las claves de esta tabla.

Ejemplo de filtro:
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> No todo documento existe en todo envío: el conjunto depende del transportista, del país de destino y del
> tipo de objeto. Solo los documentos efectivamente generados aparecen en `data.docs`, y cada clave aparece
> **una única vez**.

### Documentos incorporados (`embedded_docs`)

| `type` | Contenido | Condición |
|---|---|---|
| `pdf` | Todos los documentos PDF retornados (después del filtro `documents`) unidos en un único PDF. La cantidad de copias de cada documento y la inclusión de adjuntos PDF siguen la configuración de incorporación de tu cuenta | El envío ya debe haber sido procesado en el transportista |
| `zpl` | Todas las etiquetas ZPL retornadas (después del filtro `documents`) concatenadas, separadas por salto de línea | Que exista al menos un documento ZPL |

Los documentos incorporados solo se generan cuando **todas** las condiciones siguientes son verdaderas:
`embedded=true`, `base64=true`, tu cuenta tiene el permiso de documentos incorporados y el envío tiene
al menos un documento retornado. En caso contrario, `embedded_docs` viene como lista vacía (`[]`).

### Imprimiendo etiquetas

| Formato | Cuándo usar |
|---|---|
| `awb_pdf` | Impresora común (A4/Letter). Abre la `url` o decodifica el `base64` |
| `awb_zpl` | Impresora térmica Zebra. Envía el contenido **puro** (después de decodificar el Base64) directamente a la cola de la impresora |

> El contenido ZPL es texto. No lo conviertas ni lo reproceses: envíalo exactamente como vino después del decode.

### Gestionando adjuntos

| Operación | Ruta | Página |
|---|---|---|
| Enviar adjuntos en la creación | `POST /v1/shipping` (campo `attachments`) | [Envío (v1)](api-remessa-v1.md) |
| Enviar adjuntos después | `PUT /v1/shipping/{shipping}` | [Envío (v1)](api-remessa-v1.md#endpoint-actualizar-envío) |
| Listar adjuntos | `GET /v1/shipping/{shipping}/docs?attachments=true&base64=true` | Esta página |
| Eliminar adjuntos | `DELETE /v1/shipping/{shipping}/docs` | [Envío (v1)](api-remessa-v1.md#endpoint-eliminar-adjuntos) |

Límites de los adjuntos:

| Límite | Valor |
|---|---|
| Formatos aceptados | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamaño por archivo | **5 MB** |
| Tamaño total de la solicitud | **90 MB** |

> Los adjuntos **solo se retornan con `base64=true`**. Con `base64=false`, `data.attachments` viene siempre vacío.

## Endpoint: consultar documentos por envío

- **Método y ruta:** `GET /v1/shipping/{shipping}/docs`
- **Cuándo usar:** es la forma más directa. Acepta el UUID de la cotización, el AWB o la clave de la NF-e.

### Parámetros

| Nombre | Lugar | Tipo | Obligatorio | Tamaño | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | - | Identificador del envío. Consulta las formas aceptadas a continuación |
| `attachments` | query | boolean | no | - | `true` | Incluir los adjuntos enviados por ti. Solo tiene efecto con `base64=true` |
| `embedded` | query | boolean | no | - | `true` | Incluir los documentos incorporados. Solo tiene efecto con `base64=true` y con el permiso específico |
| `base64` | query | boolean | no | - | `true` | Incluir el contenido de los archivos en Base64, además de la `url`. Con `false`, los adjuntos y documentos incorporados no se retornan |
| `documents` | query | string | no | - | todos | Lista de tipos separados por comas (valores de la tabla [Tipos de documento filtrables](#tipos-de-documento-filtrables-documents)). Vacío o ausente: todos |

Formas de informar `{shipping}` (usa solo una):

| Forma | Valor | Cómo se reconoce | Observaciones |
|---|---|---|---|
| A | UUID de la cotización | Contiene guion (`-`) | Forma recomendada |
| B | Clave de la NF-e | Exactamente 44 dígitos numéricos | Considera el envío más reciente con esa clave, sin tener en cuenta los envíos cancelados |
| C | AWB (número de seguimiento) | Cualquier otro valor | Solo envíos creados en los **últimos 90 días**; considera el más reciente con ese AWB |

> Como cualquier valor con guion se trata como UUID, un AWB que contenga `-` no es localizado por esta ruta.
> En ese caso, usa el UUID de la cotización.

Reglas de los parámetros de query:

| Regla | Comportamiento |
|---|---|
| Valores booleanos | Se consideran verdaderos: `true`, `1`, `on`, `yes`. **Cualquier otro valor, incluso vacío** (`attachments=`), se considera `false` |
| `documents` con valor desconocido | El valor se ignora. Si **ningún** valor es válido, el filtro no se aplica y se retornan todos los documentos |
| `documents` con letras mayúsculas | No se reconoce (los valores distinguen mayúsculas y minúsculas) |
| `documents` repetido (`documents=a&documents=b`) | Solo se considera el **último**. Envía todos en un único parámetro, separados por comas |
| `documents[]=...` | No soportado: la solicitud falla con `500` |

> **Consejo de rendimiento:** envía `base64=false` cuando vayas a descargar por la `url`. El payload queda mucho más pequeño
> y la respuesta, bastante más rápida. Recuerda que, así, los adjuntos y documentos incorporados no vienen en la respuesta.

### Payload mínimo

Endpoint sin cuerpo. Basta con el parámetro de path (todos los parámetros de query asumen el valor predeterminado):

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs
```

### Payload máximo

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Ejemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf,awb_zpl' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

Solo las URL, sin contenido Base64:

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK` — con `base64=true`, adjuntos y permiso de documentos incorporados:

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "Invoice": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "PackingList": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "NonDangerousObject": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Factura Fiscal", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML de la Factura Fiscal", "base64": "PD94bWwg..." }
    ],
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "carrier": "Contrato Ejemplo Expreso",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — con `base64=false`:

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": null
      }
    },
    "attachments": [],
    "embedded_docs": [],
    "carrier": "Contrato Ejemplo Expreso",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — envío aún sin documentos, pero con adjuntos (`base64=true`):

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": [],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Factura Fiscal", "base64": "JVBERi0xLjcK..." }
    ],
    "embedded_docs": []
  }
}
```

`400 Bad Request` — ningún documento ni adjunto:

```json
{
  "status": "warning",
  "message": "Nenhum documento encontrado com o código especificado!"
}
```

### Campos de la respuesta

| Campo | Tipo | Presencia | Descripción |
|---|---|---|---|
| `status` | string | siempre | `success`, `warning` o `error` |
| `message` | string | siempre | Mensaje traducido según el header `lang` |
| `data.docs` | object \| array | siempre | Mapa `clave del documento → { url, base64 }`. Las claves son las de la tabla [Tipos de documento filtrables](#tipos-de-documento-filtrables-documents). **Cuando no hay documentos, viene como lista vacía (`[]`)** |
| `data.docs.*.url` | string | siempre | URL para descargar el archivo. Puede ser temporal (consulta [Reglas de negocio](#reglas-de-negocio)) |
| `data.docs.*.base64` | string \| null | siempre | Contenido en Base64. `null` cuando `base64=false` |
| `data.attachments` | array | siempre | Adjuntos enviados por ti. Lista vacía cuando `attachments=false`, `base64=false` o sin adjuntos |
| `data.attachments[].id` | integer | - | **ID del adjunto.** Úsalo para eliminarlo en `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | - | Extensión del archivo: `pdf`, `xml`, `jpg` o `png` |
| `data.attachments[].description` | string | - | Descripción informada en el envío del adjunto |
| `data.attachments[].base64` | string | - | Contenido en Base64 |
| `data.embedded_docs` | array | siempre | Documentos incorporados. Lista vacía cuando no aplican (consulta [Documentos incorporados](#documentos-incorporados-embedded_docs)) |
| `data.embedded_docs[].type` | string | - | `pdf` o `zpl` |
| `data.embedded_docs[].base64` | string | - | Contenido en Base64 |
| `data.carrier` | string | cuando hay documentos | Nombre del contrato que procesó el envío o, sin contrato, el nombre del transportista |
| `data.tracking_id` | string \| null | cuando hay documentos | Número de seguimiento/AWB |

## Endpoint: consultar documentos por canal

- **Método y ruta:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- **Cuándo usar:** cuando solo guardaste el número de tu propio pedido (informado en la cotización).

### Parámetros

| Nombre | Lugar | Tipo | Obligatorio | Tamaño | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **sí** | - | - | Identificador del pedido en el canal, exactamente como se informó en la cotización |
| `channel_code` | path | integer | no | - | ausente | Código del canal de venta informado en la cotización. Debe ser un número entero |
| `attachments` | query | boolean | no | - | `true` | Igual que en [consultar documentos por envío](#endpoint-consultar-documentos-por-envío) |
| `embedded` | query | boolean | no | - | `true` | Igual que en [consultar documentos por envío](#endpoint-consultar-documentos-por-envío) |
| `base64` | query | boolean | no | - | `true` | Igual que en [consultar documentos por envío](#endpoint-consultar-documentos-por-envío) |
| `documents` | query | string | no | - | todos | Igual que en [consultar documentos por envío](#endpoint-consultar-documentos-por-envío) |

> **Cómo funciona la búsqueda:**
>
> - **Con `channel_code`:** busca la cotización más reciente con ese identificador **y** ese código de canal.
> - **Sin `channel_code`:** busca la cotización más reciente con ese identificador, **con o sin** código de canal.
>   Si usas el mismo identificador en canales diferentes, envía siempre el `channel_code`.
> - A diferencia de la [consulta de status por canal](api-rastreio.md#endpoint-consultar-status-por-canal), esta búsqueda
>   **también considera los envíos cancelados**.
> - Para omitir el `channel_code`, basta con no enviar el segmento. El valor `,` también se trata como ausente.

### Payload mínimo

```
GET /v1/shipping/docs/PED-2026-0001
```

### Payload máximo

```
GET /v1/shipping/docs/PED-2026-0001/1?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Ejemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK` — misma estructura que [consultar documentos por envío](#endpoint-consultar-documentos-por-envío).

`422 Unprocessable Entity` — `channel_code` no numérico:

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "channel_code": [
      "O campo channel code deve conter um número inteiro."
    ]
  }
}
```

### Campos de la respuesta

Idénticos a los de [consultar documentos por envío](#campos-de-la-respuesta).

## Reglas de negocio
- Los documentos solo existen **después** del procesamiento en el transportista (grupo `shipped` en el seguimiento).
- **Orden de decisión de la respuesta:**

  | Documentos encontrados (después del filtro) | Adjuntos (`attachments=true` y `base64=true`) | Respuesta |
  |---|---|---|
  | Sí | Sí o no | `200` con `docs`, `attachments`, `embedded_docs`, `carrier` y `tracking_id` |
  | No | Sí | `200` con `docs: []`, `attachments` y `embedded_docs: []` (sin `carrier` ni `tracking_id`) |
  | No | No | `400` `Nenhum documento encontrado com o código especificado!` |

- El filtro `documents` también vale para la decisión anterior: si los tipos filtrados aún no existen y no hay
  adjuntos, la respuesta es `400`.
- Los documentos cuyo archivo no está disponible se omiten en silencio. Si no queda ninguno y no hay
  adjuntos, la respuesta es `400`.
- `embedded_docs` depende de un **permiso específico** en tu cuenta. Sin él, la lista viene vacía, incluso con
  `embedded=true`.
- **Validez de las URL:** cuando tu cuenta está configurada para proteger los enlaces de los documentos, cada `url`
  es **temporal y vale 5 minutos**. En los demás casos el enlace es público, pero también puede ser temporal.
  **Descarga y almacena** los archivos de tu lado justo después de la consulta; para descargarlos de nuevo más tarde, consulta el
  endpoint nuevamente.
- Solo se retornan documentos de envíos de tu cuenta. Los envíos de otra cuenta responden
  `400 Nenhum documento encontrado com o código especificado!` (por UUID y por canal) o
  `404 Esta remessa não foi encontrada!` (por AWB y por NF-e).
- La búsqueda por AWB considera solo los envíos creados en los **últimos 90 días**.
- La búsqueda por clave de NF-e no tiene en cuenta los envíos cancelados.

## Errores comunes

| Status | Significado/Mensaje | Cómo corregir |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Esperar el procesamiento (status `shipped`), revisar el filtro `documents` o verificar el identificador. En la consulta por canal, también indica que no se encontró ningún envío |
| `400` | `Remessa não encontrada!` — ningún envío con el UUID informado (consulta por `{shipping}`) | Verificar el UUID de la cotización |
| `403` | `Has no permission!` | Solicitar al soporte el permiso de consulta de documentos |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Cuenta bloqueada: contactar al soporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificar el token enviado en `Authorization` |
| `404` | `Esta remessa não foi encontrada!` — AWB o clave de NF-e sin envío correspondiente en tu cuenta (AWB con más de 90 días, NF-e de envío cancelado) | Verificar el AWB/NF-e o usar el UUID de la cotización |
| `422` | `O campo channel code deve conter um número inteiro.` (en `data.channel_code`) | Enviar un entero en `channel_code` u omitir el segmento |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- La consulta por `{shipping}` con el UUID de la cotización es la forma más directa y estable.
- La consulta por canal es la mejor opción cuando quieres trabajar solo con el número de tu pedido.
- Combina `documents` + `base64=false` para respuestas rápidas en integraciones de alto volumen.
- Para saber cuándo los documentos quedan disponibles, consulta [Seguimiento del Envío](api-rastreio.md).
- Para enviar y eliminar adjuntos, consulta [Envío (v1)](api-remessa-v1.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
