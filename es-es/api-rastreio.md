# Seguimiento del Envío

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Consulta el status operativo y el historial completo de un envío, desde la recepción en la plataforma hasta la
  entrega (o la cancelación).
- La misma respuesta trae **dos formatos**: una `timeline` cronológica y los eventos **agrupados por status**.
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

> Las descripciones de status (`status_description`) se traducen según el header `lang`.
> Los códigos (`status_code`) **no** cambian con el idioma — úsalos para la lógica de integración.

## Endpoints

### Status por envío
- **Método y ruta:** `GET /v1/shipping/{shipping}/status`

### Status por canal
- **Método y ruta:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

## Parámetros

| Nombre | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `shipping` | path | string | **sí** | UUID de la cotización, número del AWB (últimos 90 días) o clave de la NF-e (44 dígitos) |
| `channel_identifier_number` | path | string | **sí** (variante por canal) | Identificador comercial informado en la cotización |
| `channel_code` | path | integer | no | Código del canal de venta |

## Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

Por canal:
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

## Ejemplo de response

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
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:40",
          "status_code": "POT0001",
          "status_description": "Pagamento aprovado junto ao gateway: approved"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:44:38",
          "status_code": "SPD0001",
          "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
        },
        {
          "update_date": "2026-02-22",
          "update_time": "09:10:02",
          "status_code": "CLD0001",
          "status_description": "Remessa coletada pelo transportador"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "ITR0001",
          "status_description": "Remessa em trânsito internacional"
        }
      ]
    },
    "cost_center": {
      "code": 1,
      "name": "E-commerce"
    },
    "received": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "RCV0001",
        "status_description": "Remessa foi recebida em nosso sistema"
      }
    ],
    "paid_out": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:40",
        "status_code": "POT0001",
        "status_description": "Pagamento aprovado junto ao gateway: approved"
      }
    ],
    "shipped": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:44:38",
        "status_code": "SPD0001",
        "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
      }
    ],
    "collected": [
      {
        "update_date": "2026-02-22",
        "update_time": "09:10:02",
        "status_code": "CLD0001",
        "status_description": "Remessa coletada pelo transportador"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    }
  }
}
```

## Estructura de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.carrier.name` | string | Nombre del contrato/transportista que procesó el envío |
| `data.carrier.tracking_number` | string | Número de seguimiento (AWB) en el transportista |
| `data.carrier.timeline` | array | **Todos los eventos en orden cronológico.** Usa este campo para montar una línea de tiempo en tu interfaz |
| `data.cost_center` | object | **Opcional.** Solo aparece cuando el envío tiene centro de costo |
| `data.cost_center.code` | integer | Código del centro de costo |
| `data.cost_center.name` | string | Nombre del centro de costo |
| `data.<grupo_de_status>` | array | Eventos agrupados por status. Las claves existentes varían según el avance |
| `data.last_update` | object | **Último evento**, en el formato `{ "<grupo>": { ...evento } }`. Úsalo para saber el estado actual con una única lectura |

Cada evento (en `timeline`, en los grupos y en `last_update`) tiene la misma forma:

| Campo | Tipo | Descripción |
|---|---|---|
| `update_date` | string | Fecha del evento (`Y-m-d`) |
| `update_time` | string | Hora del evento (`H:i:s`) |
| `status_code` | string | Código interno del status (ver tabla) o el código bruto del gateway/transportista |
| `status_description` | string | Descripción legible, traducida por el header `lang` |

### Los dos formatos, y cuándo usar cada uno

| Formato | Dónde está | Úsalo para |
|---|---|---|
| Cronológico | `data.carrier.timeline` | Mostrar el historial completo al usuario final |
| Agrupado por status | `data.received`, `data.shipped`, `data.in_transit`... | Verificar si el envío **ya pasó** por determinada etapa (`if (isset(data.shipped))`) |
| Último estado | `data.last_update` | Dashboards y sincronizaciones: una lectura resuelve "dónde está el envío" |

## Grupos de status y códigos

| Grupo (clave en `data`) | `status_code` | Significado |
|---|---|---|
| `received` | `RCV0001` | El envío fue recibido en la plataforma |
| `processing_payment` | `PPY0001` | Esperando la confirmación del pago en el gateway |
| `paid_out` | `POT0001` | Pago aprobado |
| `pending_consolidation` | `PCD0001` | Esperando consolidación (contratos con consolidación) |
| `consolidating` | `CDT0001` | En consolidación |
| `processing_shipment` | `PSH0001` | Pago resuelto, esperando el procesamiento en el transportista |
| `shipped` | `SPD0001` | **Procesado en el transportista. Documentos disponibles** |
| `collecting` | `CLG0001` | Recogida en curso |
| `collected` | `CLD0001` | Recogido por el transportista |
| `in_transit` | `ITR0001` | En tránsito |
| `delivering` | `DLG0001` | En ruta de entrega |
| `delivered` | `DLD0001` | Entregado |
| `error` | `ERR0001` | Error en el procesamiento |
| `cancelling` | `CCG0001` | Cancelación solicitada, en procesamiento |
| `cancelled` | `CCD0001` | Cancelado |
| `foc` | `FOC0001` | Finalizado por el transportista |
| `eol` | `EOL0001` | Fin del ciclo de vida del seguimiento |

> **`status_code` puede traer el código bruto del gateway o del transportista** cuando existe
> (ej.: `approved`, `charge.succeeded`, códigos propios de la transportista). Los códigos `XXX0001` son el
> de/para interno de la plataforma, usados cuando no hay código externo. Para la lógica de negocio, prefiere la
> **clave del grupo** (`shipped`, `delivered`, `cancelled`...) en lugar del `status_code`.

## Flujo típico de estados

```
received
   │
   ├── processing_payment ──► paid_out        (pago vía gateway)
   │
   └── (pago billed: sigue directo)
                │
                ├── pending_consolidation ──► consolidating   (cuando hay consolidación)
                │
                ▼
         processing_shipment
                │
                ▼
             shipped          ◄── documentos disponibles a partir de aquí
                │
                ▼
    collecting ──► collected ──► in_transit ──► delivering ──► delivered
                │
                └── error | cancelling ──► cancelled | foc | eol
```

## Reglas de negocio

- **Todo envío necesita tener un registro de pago** para que el status sea montado. Si no lo hay,
  la respuesta es `404` con `Nenhuma entrada de pagamento foi registrada para esta remessa!`.
- Con `payment.type: billed`, el envío no pasa por `processing_payment`: sigue directo a
  `processing_shipment`/`shipped`.
- Con `payment.type: gateway`, el envío queda en `processing_payment` hasta que el gateway apruebe; solo entonces
  avanza a `processing_shipment`.
- El grupo cambia de `processing_shipment` a `shipped` en el momento en que los **documentos son generados**.
  Esa es la señal para llamar a [Documentos del Envío](api-documentos.md).
- Cuando el envío está en `cancelling`, `cancelled` o `error`, el `last_update` usa la descripción propia
  de la plataforma en lugar del texto bruto del transportista.
- Los envíos que permanecen **29 días o más** sin evento de seguimiento en un status elegible pueden ser cancelados
  automáticamente por la plataforma.
- La búsqueda por AWB funciona para envíos creados en los **últimos 90 días**.

## Buenas prácticas de polling

| Situación | Intervalo recomendado |
|---|---|
| Justo después de crear el envío (esperando `shipped`) | 10 a 30 segundos, con backoff |
| Envío ya en `shipped`/`in_transit` | 1 a 4 horas |
| Envío en `delivered`, `cancelled`, `foc` o `eol` | Detén el polling: son estados finales |

1. Lee `data.last_update` para decidir si hubo cambio — es la lectura más barata.
2. Detén el polling en los estados finales.
3. Usa backoff exponencial en `500`.
4. Trata `400`/`404` como condición de negocio: no repitas la misma solicitud en bucle.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Verificar `{shipping}` o el identificador del canal |
| `403` | Sin permiso para consultar el status | Solicitar el permiso al soporte |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Esperar el registro del pago; si persiste, contactar al soporte |
| `422` | `channel_code` no numérico | Enviar un entero en `channel_code` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- `data.carrier.timeline` es la visión más completa; los grupos son un resumen operativo para facilitar la
  lectura programática.
- Guarda el `tracking_number` para ofrecer al cliente final el seguimiento en el sitio del transportista.
- Para descargar la etiqueta y la invoice en cuanto el envío alcance `shipped`, consulta
  [Documentos del Envío](api-documentos.md).
