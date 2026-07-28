# Documentos de Envío

- [Volver al README del idioma](README.md)

## Visión general
- Consulta documentos generados después del procesamiento del envío.
- Puede retornar PDF, ZPL, anexos y, cuando esté permitido, documentos incorporados.
- Este módulo está publicado en la v1.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- `attachments`, `embedded` y `base64` aceptan booleanos vía query string.
- `documents` recibe lista separada por coma.

## Flujo de uso
1. Cree el envío.
2. Espere a que el status indique procesamiento/conclusión.
3. Consulte los documentos por `shipping` o por canal.
4. Descargue el contenido en `url` o `base64`.

## Endpoints
### Consultar documentos por envío
- Método y ruta: `GET /v1/shipping/{shipping}/docs`
- `shipping` acepta UUID de la cotización, AWB o NF-e.

### Consultar documentos por canal
- Método y ruta: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sí | Identificador del envío | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | También acepta AWB y NF-e |
| `channel_identifier_number` | path | string | sí | Identificador del canal | `334455` | - |
| `channel_code` | path | integer | no | Código del canal | `1` | Opcional |
| `attachments` | query | boolean | no | Incluye anexos | `true` | Default `true` |
| `embedded` | query | boolean | no | Incluye docs incorporados | `true` | Default `true` |
| `base64` | query | boolean | no | Incluye base64 | `true` | Default `true` |
| `documents` | query | string | no | Filtra documentos | `invoice_pdf,awb_pdf` | Cuando está vacío, retorna todo |

### Ejemplo de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response
```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
    "tracking_id": "000000",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "string"
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "string" }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "string" }
    ]
  }
}
```

## Reglas de negocio
- Si no hay documentos ni anexos, la API puede retornar `warning`.
- Los documentos incorporados dependen de permiso específico.
- Los tipos de documentos filtrables son los listados en la respuesta del envío.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Ningún documento encontrado | Confirmar si el envío ya fue procesado |
| `500` | Error inesperado | Repetir y accionar al equipo responsable |

## Observaciones finales
- Para integraciones, la consulta por `shipping` es la forma más directa.
- La consulta por canal es útil cuando el integrador guarda el identificador comercial en el envío.
