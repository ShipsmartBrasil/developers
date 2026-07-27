# Documentos de remesa

- [Volver al README del idioma](README.md)

## Visión general
- Obtiene los documentos generados después del procesamiento de la remesa.
- Puede devolver PDF, ZPL, anexos y documentos embebidos.
- Este módulo está publicado en v1.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma: `lang`.

## Convenciones
- `attachments`, `embedded` y `base64` aceptan booleanos en query.
- `documents` recibe una lista separada por comas.

## Uso
1. Crea la remesa.
2. Espera a que el estado indique procesamiento/conclusión.
3. Consulta los documentos por `shipping` o por canal.
4. Descarga el contenido en `url` o `base64`.

## Endpoints
- `GET /v1/shipping/{shipping}/docs`
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

## Parámetros
| Nombre | Ubicación | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sí | Identificador de la remesa | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | También acepta AWB y NF-e |
| `channel_identifier_number` | path | string | sí | Identificador del canal | `334455` | - |
| `channel_code` | path | integer | no | Código del canal | `1` | Opcional |
| `attachments` | query | boolean | no | Incluye anexos | `true` | Default `true` |
| `embedded` | query | boolean | no | Incluye docs embebidos | `true` | Default `true` |
| `base64` | query | boolean | no | Incluye base64 | `true` | Default `true` |
| `documents` | query | string | no | Filtra documentos | `invoice_pdf,awb_pdf` | Vacío devuelve todo |
