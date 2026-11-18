# Documentos del Envío

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Recupera los documentos generados por la plataforma y por el transportista después del procesamiento del envío:
  AWB (etiqueta), invoice, packing list, declaración y otros.
- También devuelve los **adjuntos** que enviaste en la creación/actualización del envío.
- Cada documento viene con una `url` para descarga y, opcionalmente, con el contenido en `base64`.
- Funciona para envíos creados en la **v1 y en la v2**.

## Base URL
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
lang: pt
```

## Flujo de uso
1. Crea el [envío](api-remessa-v1.md).
2. Consulta el [status](api-rastreio.md) hasta que el envío aparezca en el grupo `shipped` — es en ese momento que
   los documentos pasan a existir.
3. Consulta los documentos por `{shipping}` o por el canal.
4. Descarga el archivo por la `url` (recomendado) o decodifica el `base64`.

> **Antes de `shipped` la consulta puede retornar `400 Nenhum documento encontrado com o código especificado!`.**
> Esto es esperado: espera el procesamiento en el transportista.

## Endpoints

### Consultar documentos por envío
- **Método y ruta:** `GET /v1/shipping/{shipping}/docs`

### Consultar documentos por canal
- **Método y ruta:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- Útil cuando solo guardaste el número de tu propio pedido.

## Parámetros

| Nombre | Lugar | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | UUID de la cotización, número del AWB (últimos 90 días) o clave de la NF-e (44 dígitos) |
| `channel_identifier_number` | path | string | **sí** (variante por canal) | - | Identificador comercial informado en la cotización |
| `channel_code` | path | integer | no | - | Código del canal de venta |
| `attachments` | query | boolean | no | `true` | Incluir los adjuntos enviados por ti |
| `embedded` | query | boolean | no | `true` | Incluir documentos incorporados (AWB + invoice consolidados en un único archivo). Requiere permiso específico |
| `base64` | query | boolean | no | `true` | Incluir el contenido del archivo en Base64, además de la `url` |
| `documents` | query | string | no | todos | Lista de tipos separados por comas. Cuando está vacío o ausente, retorna todos |

> **Consejo de rendimiento:** envía `base64=false` cuando vayas a descargar por la `url`. El payload queda mucho más pequeño
> y la respuesta, bastante más rápida.

## Tipos de documento filtrables (`documents`)

| Valor en `documents` | Clave en `data.docs` | Contenido |
|---|---|---|
| `awb_pdf` | `AWB` | Etiqueta/AWB en PDF |
| `awb_zpl` | `AWB_ZPL` | Etiqueta/AWB en ZPL (impresora térmica) |
| `awb_prc_pdf` | `AWB PRC` | AWB del proceso de retorno/socio en PDF |
| `awb_prc_zpl` | `AWB PRC ZPL` | AWB del proceso de retorno/socio en ZPL |
| `awb_prc_fedex_pdf` | `AWB PRC Fedex` | AWB FedEx del proceso de retorno/socio en PDF |
| `awb_prc_fedex_zpl` | `AWB PRC Fedex ZPL` | AWB FedEx del proceso de retorno/socio en ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice en PDF |
| `packing_list_pdf` | `Packing List` | Packing list en PDF |
| `declaration_pdf` | `Declaration` | Declaración de contenido en PDF |
| `non_dangerous_pdf` | `Non-Dangerous Object` | Declaración de mercancía no peligrosa en PDF |

Ejemplo de filtro:
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> No todo documento existe en todo envío: el conjunto depende del transportista, del país de destino y del
> tipo de objeto. Solo los documentos efectivamente generados aparecen en `data.docs`.

## Ejemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=false&base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

Por canal:
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

## Ejemplo de response

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
    "tracking_id": "1234567890123",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "Packing List": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Non-Dangerous Object": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML de la Nota Fiscal", "base64": "PD94bWwg..." }
    ]
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.carrier` | string | Nombre del transportista que procesó el envío |
| `data.tracking_id` | string | Número de seguimiento/AWB |
| `data.docs` | object | Mapa `nombre del documento → { url, base64 }`. Las claves son las de la tabla de arriba |
| `data.docs.*.url` | string | URL para descarga directa del archivo |
| `data.docs.*.base64` | string | Contenido en Base64. Presente solo cuando `base64=true` |
| `data.embedded_docs` | array | Documentos incorporados. `null`/ausente cuando `embedded=false` o sin permiso |
| `data.embedded_docs[].type` | string | `pdf` o `zpl` |
| `data.embedded_docs[].base64` | string | Contenido en Base64 |
| `data.attachments` | array | Adjuntos enviados por ti. `null`/ausente cuando `attachments=false` |
| `data.attachments[].id` | integer | **ID del adjunto.** Úsalo para eliminarlo en `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | `pdf`, `xml`, `jpeg` o `png` |
| `data.attachments[].description` | string | Descripción informada en la creación |
| `data.attachments[].base64` | string | Contenido en Base64 |

## Imprimiendo etiquetas

| Formato | Cuándo usarlo |
|---|---|
| `awb_pdf` | Impresora común (A4/Letter). Abre la `url` o decodifica el `base64` |
| `awb_zpl` | Impresora térmica Zebra. Envía el contenido **puro** (después de decodificar el Base64) directo a la cola de la impresora |

> El contenido ZPL es texto. No lo conviertas ni lo reproceses: envíalo exactamente como vino después del decode.

## Gestionando adjuntos

| Operación | Ruta | Página |
|---|---|---|
| Enviar adjuntos en la creación | `POST /v1/shipping` (campo `attachments`) | [Envío (v1)](api-remessa-v1.md) |
| Enviar adjuntos después | `PUT /v1/shipping/{shipping}` | [Envío (v1)](api-remessa-v1.md#endpoint-actualizar-envío) |
| Listar adjuntos | `GET /v1/shipping/{shipping}/docs?attachments=true` | Esta página |
| Eliminar adjuntos | `DELETE /v1/shipping/{shipping}/docs` | [Envío (v1)](api-remessa-v1.md#endpoint-eliminar-adjuntos) |

Límites de los adjuntos:

| Límite | Valor |
|---|---|
| Formatos aceptados | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamaño por archivo | **5 MB** |
| Tamaño total de la solicitud | **90 MB** |

## Reglas de negocio
- Los documentos solo existen **después** del procesamiento en el transportista (grupo `shipped` en el seguimiento).
- `embedded_docs` depende de un **permiso específico** en tu cuenta. Sin él, el campo no se retorna,
  incluso con `embedded=true`. Mensaje relacionado:
  `Para obter os documentos incoporados a remessa precisa ter sido ao menos enviada.`
- Si no hay ningún documento ni adjunto, la respuesta es `400` con
  `Nenhum documento encontrado com o código especificado!`.
- Las `url` apuntan a almacenamiento externo y pueden tener validez. **Descarga y almacena** los archivos
  de tu lado si necesitas mantenerlos por mucho tiempo.
- La consulta por canal considera el `channel_identifier_number` informado en la **cotización**.
  Sin `channel_code`, la búsqueda usa solo el identificador.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Esperar el procesamiento (status `shipped`) o verificar el identificador |
| `403` | Sin permiso para consultar documentos | Solicitar el permiso al soporte |
| `404` | Envío no encontrado | Verificar `{shipping}` o el identificador del canal |
| `422` | `channel_code` no numérico | Enviar un entero en `channel_code` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- La consulta por `{shipping}` es la forma más directa y estable.
- La consulta por canal es la mejor opción cuando quieres trabajar solo con el número de tu pedido.
- Combina `documents` + `base64=false` para respuestas rápidas en integraciones de alto volumen.
