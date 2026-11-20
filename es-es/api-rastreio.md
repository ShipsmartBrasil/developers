# Seguimiento del Envío

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Consulta el status operativo y el historial de un envío, desde la recepción en la plataforma hasta la
  entrega (o la cancelación).
- La misma respuesta trae **tres lecturas**: la `timeline` cronológica, los eventos **agrupados por status** y el
  **último estado** (`last_update`).
- Hay dos formas de consultar: por el identificador del envío (`{shipping}`) o por el identificador de tu
  canal de venta (`channel_identifier_number` + `channel_code`).
- Funciona para envíos creados en la **v1 y en la v2** (las rutas de seguimiento existen solo en la v1).

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
| `Authorization` | **sí** | `Bearer <token>` | - | Token de tu cuenta. El perfil necesita el permiso de consulta de status de envío |
| `lang` | no | `pt`, `en`, `es` | `pt` | Idioma de los mensajes y de las descripciones generadas por la plataforma |

> El header `lang` traduce el `message` de la respuesta y las descripciones de los eventos generados por la **plataforma**
> (recepción, pago, procesamiento, cancelación y error). Las descripciones de los eventos provenientes del
> **transportista** se devuelven tal como el transportista las envió y **no** cambian con el idioma.
> Los códigos (`status_code`) y las claves de los grupos nunca cambian con el idioma — úsalos en la lógica de la integración.

## Flujo de uso
1. Crea el envío ([Envío v1](api-remessa-v1.md) o [Envío v2](api-remessa-v2.md)) y guarda el `quotation`
   (UUID) usado en la creación.
2. Consulta el status por `{shipping}` (recomendado) o por el identificador del canal.
3. Cuando aparezca el grupo `shipped`, los documentos ya existen: descárgalos en [Documentos del Envío](api-documentos.md).
4. Sigue consultando en intervalos mayores para acompañar la recogida, el tránsito y la entrega.
5. Deja de consultar cuando `last_update` indique un estado final (consulta [Buenas prácticas de polling](#buenas-prácticas-de-polling)).

```
crear envío ──► GET .../status ──► (shipped?) ──► GET .../docs
                      │                 │
                      │ no              │ sí
                      ▼                 ▼
               esperar y repetir   acompañar recogida → tránsito → entrega
```

## Conceptos esenciales

### Las tres lecturas de la respuesta

| Lectura | Dónde está | Qué contiene | Úsala para |
|---|---|---|---|
| Cronológica | `data.carrier.timeline` | Todos los eventos armados por la plataforma y todos los eventos de seguimiento del transportista, en el orden en que fueron registrados | Mostrar el historial completo al usuario final |
| Agrupada por status | `data.received`, `data.shipped`, `data.in_transit`... | Una clave por status ya alcanzado. Cada clave es una lista con **un único evento**: el más reciente de ese status | Verificar si el envío **ya pasó** por determinada etapa |
| Último estado | `data.last_update` | El estado actual, en el formato `{ "<grupo>": { ...evento } }` | Dashboards y sincronizaciones: una lectura resuelve "dónde está el envío" |

> **Atención:** los grupos no guardan el historial. Si el transportista envía tres eventos `in_transit`, el grupo
> `data.in_transit` traerá solo el último; los tres aparecen en `data.carrier.timeline`.

### Grupos de status y códigos

| Grupo (clave) | `status_code` de la plataforma | Significado | Dónde puede aparecer |
|---|---|---|---|
| `received` | `RCV0001` | El envío fue recibido en la plataforma | Grupo y `timeline` (siempre presente) |
| `processing_payment` | `PPY0001` | Pago aún no aprobado (también es el evento mostrado para pago facturado, `billed`) | Grupo y `timeline` |
| `paid_out` | `POT0001` | Pago aprobado en el medio de pago | Grupo y `timeline` |
| `pending_consolidation` | `PCD0001` | Esperando consolidación (contratos con consolidación) | Reservado (ver [Reglas de negocio](#reglas-de-negocio)) |
| `consolidating` | `CDT0001` | En consolidación | Reservado (ver [Reglas de negocio](#reglas-de-negocio)) |
| `processing_shipment` | `PSH0001` | Pago resuelto, esperando el procesamiento en el transportista | Grupo y `timeline` |
| `shipped` | `SPD0001` | **Procesado en el transportista. Documentos disponibles** | Grupo y `timeline` |
| `collecting` | `CLG0001` | Transportista en camino a la recogida | Grupo, `timeline` y `last_update` |
| `collected` | `CLD0001` | Recogido por el transportista | Grupo, `timeline` y `last_update` |
| `in_transit` | `ITR0001` | En tránsito | Grupo, `timeline` y `last_update` |
| `delivering` | `DLG0001` | Salió para entrega | Grupo, `timeline` y `last_update` |
| `delivered` | `DLD0001` | Entregado | Grupo, `timeline` y `last_update` |
| `error` | `ERR0001` | No fue posible concluir el embarque | Solo `last_update` |
| `cancelling` | `CCG0001` | Cancelación solicitada, en procesamiento | Solo `last_update` |
| `cancelled` | `CCD0001` | Cancelado | Solo `last_update` |
| `foc` | `FOC0001` | Finalizado por el transportista | Solo `last_update` |
| `eol` | `EOL0001` | Fin del ciclo de vida del seguimiento | Solo `last_update` |

> **`status_code` no siempre es un código `XXX0001`.** Trae el código bruto del origen del evento cuando
> ese código existe:
>
> - en el evento de pago: el status informado por el medio de pago (ej.: `approved`);
> - en los eventos de seguimiento: el código propio del transportista.
>
> Los códigos `XXX0001` se usan cuando no hay código externo. Para la lógica de negocio, usa la **clave del
> grupo** (`shipped`, `in_transit`, `delivered`, `cancelled`...) en lugar del `status_code`.

### Descripciones generadas por la plataforma

Mensajes literales en `pt` (cambian según `lang`):

| Grupo | `status_description` |
|---|---|
| `received` | `Remessa foi recebida em nosso sistema` |
| `processing_payment` | `Aguardando a finalização do processamento do pagamento junto ao gateway: :description ` |
| `paid_out` | `Pagamento aprovado junto ao gateway: :description ` |
| `processing_shipment` | `Remessa esta sendo processada no transportador especificado!` |
| `shipped` | `Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!` |
| `cancelling` (en `last_update`) | `Cancelando Remessa` |
| `cancelled` (en `last_update`) | `Remessa Cancelada` |
| `error` (en `last_update`) | `Não foi possível concluir o embarque desta remessa.` |

> En los mensajes de pago, `:description` es la descripción enviada por el medio de pago o, a falta de ella,
> el propio código de la plataforma (`PPY0001` / `POT0001`). Los dos mensajes terminan con un **espacio** —
> tenlo en cuenta si comparas strings (lo recomendado es no comparar descripciones).

### Flujo típico de estados

```
received
   │
   ├── processing_payment ──► paid_out          (pago vía gateway)
   │
   ├── processing_payment                       (pago billed: sigue directo)
   │
   ▼
processing_shipment
   │
   ▼
shipped                 ◄── documentos disponibles a partir de aquí
   │
   ▼
collecting ──► collected ──► in_transit ──► delivering ──► delivered

Estados mostrados solo en last_update:
   error | cancelling ──► cancelled | foc | eol
```

## Endpoint: consultar status por envío

- **Método y ruta:** `GET /v1/shipping/{shipping}/status`
- **Cuándo usar:** es la forma recomendada. Usa el UUID de la cotización que enviaste como `quotation` en la creación del envío.

### Parámetros

| Nombre | Lugar | Tipo | Obligatorio | Tamaño | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | - | **Forma A:** UUID de la cotización usada en el envío. **Forma B:** número de seguimiento (AWB) del envío |

> **Identificadores aceptados en esta ruta:** UUID de la cotización o AWB. Si el AWB está en más de un envío,
> se considera el más reciente. **La clave de la NF-e no se acepta en esta ruta** (la consulta responde
> `400`), aunque sí se acepta en la ruta de documentos.

### Payload mínimo

Endpoint sin cuerpo. Basta con el parámetro de path:

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
```

### Payload máximo

No hay parámetros de query. La forma máxima es la misma que la mínima, variando solo el identificador:

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
GET /v1/shipping/1234567890123/status
```

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK` — envío con pago facturado (`billed`), ya procesado y en tránsito:

```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Contrato Ejemplo Expreso",
      "tracking_number": "1234567890123",
      "timeline": [
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "PPY0001",
          "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
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
          "status_code": "PU",
          "status_description": "Shipment picked up"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "IT",
          "status_description": "In transit"
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
    "processing_payment": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "PPY0001",
        "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
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
        "status_code": "PU",
        "status_description": "Shipment picked up"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    }
  }
}
```

`200 OK` — fragmento de `last_update` de un envío cancelado (sustituye el último evento del transportista):

```json
{
  "last_update": {
    "cancelled": {
      "update_date": "2026-02-24",
      "update_time": "10:00:00",
      "status_code": "CCD0001",
      "status_description": "Remessa Cancelada"
    }
  }
}
```

`400 Bad Request` — envío de otra cuenta:

```json
{
  "status": "warning",
  "message": "Nenhuma remessa encontrada com o código especificado!"
}
```

### Campos de la respuesta

| Campo | Tipo | Presencia | Descripción |
|---|---|---|---|
| `status` | string | siempre | `success`, `warning` o `error` |
| `message` | string | siempre | Mensaje traducido según el header `lang` |
| `data.carrier.name` | string | siempre | Nombre del contrato que procesó el envío o, sin contrato, el nombre del transportista |
| `data.carrier.tracking_number` | string \| null | siempre | Número de seguimiento (AWB) en el transportista. Puede venir `null`/vacío antes del procesamiento |
| `data.carrier.timeline` | array | siempre | Todos los eventos, en el orden de registro: recepción, pago, procesamiento y, después, los eventos de seguimiento del transportista |
| `data.cost_center` | object | opcional | Solo aparece cuando el envío tiene centro de costo |
| `data.cost_center.code` | integer | opcional | Código del centro de costo |
| `data.cost_center.name` | string \| null | opcional | Nombre del centro de costo |
| `data.received` | array | siempre | Evento de recepción (un ítem) |
| `data.processing_payment` o `data.paid_out` | array | siempre uno de los dos | Evento de pago (un ítem) |
| `data.processing_shipment` o `data.shipped` | array | después del pago aprobado o facturado | Evento de procesamiento en el transportista (un ítem) |
| `data.collecting`, `data.collected`, `data.in_transit`, `data.delivering`, `data.delivered` | array | según el seguimiento | Último evento del transportista en ese status (un ítem) |
| `data.last_update` | object | opcional | Estado actual. **Solo aparece** después del primer evento de seguimiento del transportista o cuando el envío está en `cancelling`, `cancelled` o `error` |

Cada evento (en `timeline`, en los grupos y en `last_update`) tiene la misma forma:

| Campo | Tipo | Formato | Descripción |
|---|---|---|---|
| `update_date` | string | `AAAA-MM-DD` | Fecha del evento |
| `update_time` | string | `HH:MM:SS` | Hora del evento. **Excepción:** en el evento `processing_shipment`, este campo viene actualmente con la fecha (`AAAA-MM-DD`) en lugar de la hora |
| `status_code` | string | - | Código de la plataforma (`XXX0001`) o código bruto del medio de pago/transportista |
| `status_description` | string | - | Descripción legible. Traducida según `lang` solo en los eventos generados por la plataforma |

## Endpoint: consultar status por canal

- **Método y ruta:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`
- **Cuándo usar:** cuando guardaste solo el identificador del pedido en tu canal de venta (informado en la cotización),
  y no el UUID de la cotización.

### Parámetros

| Nombre | Lugar | Tipo | Obligatorio | Tamaño | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **sí** | - | - | Identificador del pedido en el canal, exactamente como se informó en la cotización |
| `channel_code` | path | integer | no | - | ausente | Código del canal de venta informado en la cotización. Debe ser un número entero |

> **Cómo funciona la búsqueda:**
>
> - **Con `channel_code`:** busca el envío cuya cotización tiene ese identificador **y** ese código de canal.
> - **Sin `channel_code`:** busca **solo** envíos cuya cotización fue creada **sin** código de canal.
>   Si tu cotización tenía `channel_code`, necesitas enviarlo.
> - Los envíos **cancelados no se consideran**. Si hay más de un envío válido, vale el más reciente.
> - Para omitir el `channel_code`, basta con no enviar el último segmento. El valor `,` también se trata como ausente.

### Payload mínimo

```
GET /v1/shipping/status/PED-2026-0001
```

### Payload máximo

```
GET /v1/shipping/status/PED-2026-0001/1
```

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK` — misma estructura que [consultar status por envío](#endpoint-consultar-status-por-envío).

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

Idénticos a los de [consultar status por envío](#campos-de-la-respuesta).

## Reglas de negocio

- **Solo envíos de tu cuenta.** El envío debe haber sido creado por la cuenta del token. Los envíos de otra
  cuenta responden `400` con `Nenhuma remessa encontrada com o código especificado!`.
- **Todo envío necesita tener un registro de pago** para que se arme el status. Si no lo hay,
  la respuesta es `404` con `Nenhuma entrada de pagamento foi registrada para esta remessa!` (mensaje fijo,
  no cambia con `lang`).
- **Pago y grupo mostrado:**

  | `payment.type` del envío | Evento de pago | Evento de procesamiento |
  |---|---|---|
  | `billed` | `processing_payment` con `PPY0001` | `processing_shipment` → `shipped` |
  | `gateway` | `processing_payment` (código del medio de pago) hasta la aprobación; después `paid_out` | Solo aparece después de `paid_out` |
  | `afterward` | `processing_payment` hasta que el pago sea aprobado | Solo aparece después de la aprobación del pago |

  Con `afterward`, el envío puede ya tener documentos sin que aparezca el grupo `shipped`. En ese caso, consulta
  [Documentos del Envío](api-documentos.md) directamente.
- El grupo cambia de `processing_shipment` a `shipped` en el momento en que **se generan los documentos**; la fecha
  y la hora de `shipped` son las de la generación del primer documento.
- Los grupos de seguimiento (`collecting` a `delivered`) solo aparecen después de que el transportista comienza a enviar eventos.
- Cuando el envío está en `cancelling`, `cancelled` o `error`, el `last_update` usa la descripción propia
  de la plataforma (tabla [Descripciones generadas por la plataforma](#descripciones-generadas-por-la-plataforma)) en lugar del texto
  bruto del transportista.
- **Consolidación:** mientras el envío espera consolidación o está en consolidación, la consulta de status puede
  responder `500`. Espera a que concluya la consolidación y consulta nuevamente.
- **Cancelación automática:** los envíos que, **29 días después de la creación**, aún no tienen número de seguimiento en el
  transportista (y no pasaron de `shipped`) pueden ser cancelados automáticamente por la plataforma.
- **Fin del ciclo de vida:** los envíos con más de **2 meses** que no llegaron a un estado final
  (`delivered`, `cancelled`, `foc` o `eol`) son cerrados por la plataforma y dejan de recibir actualizaciones.
- La búsqueda por AWB en esta ruta considera el envío más reciente con ese número.
- La consulta por canal **ignora los envíos cancelados**: para acompañar una cancelación, consulta por `{shipping}`.

### Buenas prácticas de polling

| Situación | Intervalo recomendado |
|---|---|
| Justo después de crear el envío (esperando `shipped`) | 10 a 30 segundos, con backoff |
| Envío ya en `shipped`/`in_transit` | 1 a 4 horas |
| `last_update` en `delivered`, `cancelled`, `foc` o `eol` | Detén el polling: son estados finales |
| Envío con más de 2 meses | Detén el polling: el envío ya no recibe actualizaciones |

1. Lee `data.last_update` para decidir si hubo cambio — es la lectura más simple. Recuerda que solo
   existe después del primer evento del transportista (o en cancelación/error); antes de eso, usa los grupos.
2. Detén el polling en los estados finales.
3. Usa backoff exponencial en `500`.
4. Trata `400`/`404` como condición de negocio: no repitas la misma solicitud en bucle.

## Errores comunes

| Status | Significado/Mensaje | Cómo corregir |
|---|---|---|
| `400` | `remessa.status.status_not_found: ` — no se encontró ningún envío por el UUID ni por el AWB (consulta por `{shipping}`; el mensaje viene en este formato, sin traducción). También ocurre al enviar una clave de NF-e | Verificar el `{shipping}`. Usar el UUID de la cotización o el AWB |
| `400` | `Nenhuma remessa encontrada com o código especificado!` — envío de otra cuenta (por `{shipping}`) o ningún envío encontrado (por canal) | Verificar el identificador, el `channel_code` y el token usado |
| `403` | `Has no permission!` | Solicitar al soporte el permiso de consulta de status |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Cuenta bloqueada: contactar al soporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificar el token enviado en `Authorization` |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Esperar el registro del pago; si persiste, contactar al soporte |
| `422` | `O campo channel code deve conter um número inteiro.` (en `data.channel_code`) | Enviar un entero en `channel_code` u omitir el segmento |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- `data.carrier.timeline` es la vista más completa; los grupos son un resumen operativo para facilitar la
  lectura programática.
- Guarda el `tracking_number` para ofrecer al cliente final el seguimiento en el sitio del transportista.
- Para descargar la etiqueta y la invoice en cuanto el envío alcance `shipped`, consulta
  [Documentos del Envío](api-documentos.md).
- Para crear y cancelar envíos, consulta [Envío (v1)](api-remessa-v1.md) y [Envío (v2)](api-remessa-v2.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
