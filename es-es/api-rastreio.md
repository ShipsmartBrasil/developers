# Seguimiento de Envío

- [Volver al README del idioma](README.md)

## Visión general
- Consulta el status operativo y el historial de un envío.
- Muestra recepción, pago, procesamiento, transporte y cancelación.
- Asociado al flujo operativo principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- El campo `shipping` acepta UUID, AWB o NF-e, según el endpoint.
- Los status usan códigos internos como `RCV0001`, `PPY0001`, `SPD0001`, `CCG0001` y `CCD0001`.

## Formatos de respuesta
- El mismo endpoint retorna dos formatos: `timeline` y `status`.
- `timeline` trae los eventos en el orden en que llegan del transportador.
- `status` trae los eventos agrupados por status, usando el mapeo interno de la plataforma para simplificar la lectura.

## Flujo de uso
1. Cree el envío.
2. Consulte el status por `shipping` o por canal.
3. Siga `received`, `processing_payment`, `processing_shipment`, `shipped` y, si corresponde, cancelación.

## Endpoints
### Status por envío
- Método y ruta: `GET /v1/shipping/{shipping}/status`

### Status por canal
- Método y ruta: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sí | Identificador del envío | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Acepta UUID, AWB y NF-e |
| `channel_identifier_number` | path | string | sí | Número del canal | `334455` | - |
| `channel_code` | path | integer | no | Código del canal | `1` | Opcional |

### Ejemplo de request
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response
```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        }
      ]
    },
    "received": [],
    "processing_payment": [],
    "shipped": []
  }
}
```

## Reglas de negocio
- El pago debe existir para que el envío retorne status completo.
- Si el pago aún no fue aprobado, el status puede quedar en `processing_payment`.
- Cuando el envío es facturado y aún no procesado en el transportador, puede aparecer `processing_shipment`.
- Cuando hay cancelación, la respuesta puede incluir `cancelling` o `cancelled`.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Envío no encontrado | Verificar identificador |
| `404` | Sin pago vinculado | Confirmar procesamiento del envío |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- El seguimiento es la forma más completa de monitorear la vida útil del envío.
- La timeline se deriva de los eventos internos y del retorno del pago/transportador.
- La agrupación de status es un resumen operativo para facilitar la comprensión del seguimiento del envío.
