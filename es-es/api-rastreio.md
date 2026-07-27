# Rastreo de remesa

- [Volver al README del idioma](README.md)

## Visión general
- Consulta el estado operativo y el historial de una remesa.
- Muestra recepción, pago, procesamiento, transporte y cancelación.

## Formatos de respuesta
- El mismo endpoint devuelve `timeline` y `status`.
- `timeline` lista los eventos en orden de llegada.
- `status` agrupa los eventos por estado.

## Endpoints
- `GET /v1/shipping/{shipping}/status`
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

## Parámetros
| Nombre | Ubicación | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sí | Identificador de la remesa | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Acepta UUID, AWB y NF-e |
| `channel_identifier_number` | path | string | sí | Número del canal | `334455` | - |
| `channel_code` | path | integer | no | Código del canal | `1` | Opcional |

## Ejemplo de request
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: es'
```

## Ejemplo de response
```json
{
  "status": "success",
  "message": "Estado de la remesa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "La remesa fue recibida en nuestro sistema"
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
- El pago debe existir para que la remesa retorne el estado completo.
- Si el pago aún no fue aprobado, el estado puede quedar en `processing_payment`.
- Cuando la remesa está facturada pero aún no procesada por el transportador, puede aparecer `processing_shipment`.
- Cuando hay cancelación, la respuesta puede incluir `cancelling` o `cancelled`.
