# Programación de Recogida

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md) y [Envío (v1)](api-remessa-v1.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Solicita al transportista que **retire** el bulto en tu dirección, en lugar de que tú lo lleves a un punto
  de entrega.
- Permite consultar el avance de la programación y cancelarla con un motivo estandarizado.
- Funciona para envíos creados en la **v1 y en la v2** (los endpoints de programación existen solo en `/v1`).
- El pedido es **asíncrono**: la API registra la solicitud y el envío al transportista ocurre en segundo
  plano. La confirmación llega después, en la consulta de la programación.
- Cada envío acepta **una única programación** por la API.

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

> `Content-Type` solo es necesario en el `POST`. Cada operación exige un permiso propio en tu perfil
> (crear, consultar y cancelar programación). Sin el permiso, la respuesta es `403` con `Has no permission!`.

## Flujo de uso

```
1. GET    /v1/shipping/{shipping}/status    → confirmar que el status ACTUAL es "shipped"
2. POST   /v1/schedule                      → registrar la programación (status "processing")
3. GET    /v1/schedule/{shipping}           → acompañar hasta el status "success"
4. DELETE /v1/schedule/{shipping}/{reason}  → cancelar, si es necesario (solo con status "success")
```

### Antes de programar, confirma

| # | Condición | Cómo verificar |
|---|---|---|
| 1 | El status **actual** del envío es **`shipped`** (procesado en el transportista, documentos disponibles). Después de que el envío avanza (ej.: en tránsito), la programación deja de ser aceptada | [Seguimiento](api-rastreio.md) |
| 2 | El transportista del contrato del envío **tiene recogida integrada** (ver [Transportistas con recogida integrada](#transportistas-con-recogida-integrada)) | Contrato elegido en el envío |
| 3 | El envío **aún no tiene** ninguna programación — ni siquiera una cancelada o con error | `GET /v1/schedule/{shipping}` debe responder `404` |
| 4 | El envío fue creado por el **mismo usuario** del token | - |
| 5 | Tu cuenta **no** tiene la recogida automática habilitada en las predefiniciones (ver [Recogida automática por las predefiniciones de la cuenta](#recogida-automática-por-las-predefiniciones-de-la-cuenta)) | Panel de la cuenta |

## Conceptos esenciales

### Convenciones

| Ítem | Formato | Ejemplo |
|---|---|---|
| Fecha de la recogida | `AAAA-MM-DD` (exactamente ese formato) | `2026-03-10` |
| Horarios de la ventana | `HH:MM:SS` (con segundos) | `08:00:00` |
| Fecha "actual" | Considera el huso horario de la plataforma (horario de Brasilia) | - |
| Identificador `{shipping}` | **UUID de la cotización** o **número del AWB** (envíos creados en los últimos 90 días) | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` |

> A diferencia de los endpoints de envío, **la clave de la NF-e no se acepta** como identificador en la programación.
> Un identificador que contiene guion (`-`) siempre se trata como UUID; sin guion, se trata como AWB.

### Transportistas con recogida integrada

| Transportista | Programación por la API | Cancelación por la API |
|---|---|---|
| DHL | sí | sí |
| FedEx | sí | sí |
| UPS | sí | sí |
| SkyPostal | sí | sí |
| CTBI Log | sí | **no** (el pedido de cancelación termina en `error`; contacta al soporte) |
| GLS | sí | **no** (el pedido de cancelación termina en `error`; contacta al soporte) |

> Lista vigente en la fecha de esta versión de la guía; puede ampliarse. Para cualquier otro transportista,
> el `POST` responde `400` con `Esta transportadora não possui serviço de coleta integrado`.

### Dirección de la recogida

La dirección de retiro se define de una de estas formas. **No las mezcles**: o envías `address`, o dejas
que la plataforma la resuelva.

| Forma | Cómo enviar | Dirección usada |
|---|---|---|
| **Forma A — automática** | Omite `address` (o envía `null` / `{}`) | 1º la predefinición de recogida de tu cuenta para el país del remitente (si está habilitada y con dirección registrada); 2º la dirección del remitente del envío |
| **Forma B — dirección alternativa** | Envía el objeto `address` completo | La dirección informada, que debe estar **en el mismo país del remitente** |

> En cuentas del tipo **persona física** y **PJ mono-CD**, el objeto `address` se **ignora** y la recogida usa
> siempre la dirección del remitente del envío. Solo se verifica el formato de los campos (`422`); las reglas de
> dirección (país, código postal, documento) no se aplican.

### Status de la programación

| `code` | Significado | Próximo paso |
|---|---|---|
| `processing` | Programación registrada, esperando envío/confirmación del transportista | Esperar y consultar nuevamente |
| `success` | **Recogida confirmada por el transportista.** `confirmation_code` disponible | Dejar el bulto listo en la ventana informada |
| `canceling` | Cancelación solicitada, esperando envío al transportista | Esperar |
| `terminating` | Cancelación enviada al transportista, esperando confirmación | Esperar |
| `canceled` | Cancelación concluida | - |
| `error` | Falla en la programación o en la cancelación. El motivo viene en `status.description` | Leer la descripción y contactar al soporte |

```
POST                         DELETE
 │                             │
 ▼                             ▼
processing ──► success ──► canceling ──► terminating ──► canceled
 │                             │              │
 └──► error                    └──► error     └──► error
```

### Tipo de procesamiento

La plataforma clasifica la programación automáticamente según la fecha informada. No existe un campo en el payload.

| Tipo | Cuándo | Comportamiento |
|---|---|---|
| Inmediato | `date` = mañana | Enviado al transportista en el próximo ciclo de procesamiento |
| Programado | `date` pasado mañana o más tarde | Enviado al transportista **1 día antes** de la fecha de la recogida |

> Por eso una programación del tipo programado permanece en `processing` durante algunos días. Esto es esperado.
> Ejemplo: un pedido hecho el viernes para el lunes siguiente es **programado** (enviado el domingo).

### Recogida automática por las predefiniciones de la cuenta

Si tu cuenta tiene la **recogida automática** habilitada en las predefiniciones (configurada en el panel), la
plataforma crea la programación por sí sola en cuanto se generan los documentos del envío, sin llamada a la API:

| Ítem | Valor aplicado |
|---|---|
| Fecha | Próximo día hábil (para remitente en Brasil, salta también un feriado nacional) |
| Ventana | `08:00:00` a `18:00:00` |
| Instrucción | La instrucción registrada en la predefinición |
| Dirección | Misma regla de la Forma A (predefinición de recogida y, a falta de ella, el remitente) |

> En ese caso, un `POST /v1/schedule` para el mismo envío responde
> `400 Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso`.
> Usa solo el `GET /v1/schedule/{shipping}` para acompañar.

### Motivos de cancelación

| Código | Motivo |
|---|---|
| `001` | Package not ready (el bulto no está listo) |
| `002` | Rates too high (tarifas muy altas) |
| `003` | Transit time too slow (plazo de tránsito muy lento) |
| `004` | Take to service center or drop box (se llevará a un punto de entrega) |
| `005` | Commitment time not met (plazo acordado no cumplido) |
| `006` | Reason not given (motivo no informado) |
| `007` | Other (otro) |
| `008` | Pickup modified (recogida modificada) |

> Estos códigos están estandarizados por los transportistas y se les transmiten sin traducción.
> Usa `006` o `007` cuando ninguno de los específicos aplique. Envía siempre los **3 dígitos** (`001`, no `1`).

## Endpoint: crear programación

- **Método y ruta:** `POST /v1/schedule`
- **Cuándo usar:** con el envío en el status actual `shipped`, para pedir el retiro del bulto.

### Parámetros

| Campo | Tipo | Obligatorio | Mín. | Máx. | Formato / valores | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | string | **sí** si no se envía `awb_number` | - | - | UUID de una cotización existente | Código de la cotización/envío |
| `awb_number` | string | **sí** si no se envía `shipping` | - | 50 | - | Número del AWB (envíos creados en los últimos 90 días) |
| `date` | string | **sí** | - | - | `AAAA-MM-DD` | Fecha de la recogida. Ver [Reglas de fecha](#reglas-de-fecha) |
| `initial_time` | string | **sí** | - | - | `HH:MM:SS` | Inicio de la ventana de recogida |
| `final_time` | string | **sí** | - | - | `HH:MM:SS` | Fin de la ventana de recogida |
| `instruction` | string | **sí** | 5 | 80 | - | Instrucción para el conductor (dónde retirar, con quién hablar) |
| `address` | object \| null | no | - | - | - | Dirección alternativa de recogida (Forma B). Ver la tabla siguiente |

> Envía **`shipping` o `awb_number`**, no los dos. Si se envían los dos, prevalece el `awb_number`.
>
> La API **no** compara `initial_time` con `final_time` ni valida el horario comercial. Asegúrate de que el fin sea
> posterior al inicio y de que la ventana sea aceptada por el transportista; de lo contrario, la programación puede
> terminar en `error`.

### Parámetros — `address` (dirección alternativa de recogida)

Envíala solo si la recogida es en una dirección **diferente** de la automática.

**Atención: cuando se envía `address`, las 12 claves siguientes deben estar presentes en el objeto.**
Una clave ausente hace que la solicitud falle con `500`. Cuando no tengas el dato, usa `null` en los campos que
aceptan nulo y `""` (texto vacío) en `complement` y `phone_extension`.

| Campo | Tipo | Acepta `null` | Mín. | Máx. | Regla para remitente en Brasil | Descripción |
|---|---|---|---|---|---|---|
| `street` | string | no | 5 | 70 | - | Calle |
| `number` | string | sí | - | 6 | **Obligatorio** (no puede estar vacío) | Número del inmueble |
| `district` | string | sí | - | 30 | **Obligatorio** (no puede estar vacío) | Barrio |
| `complement` | string | no (usa `""`) | - | 20 | - | Complemento |
| `city` | string | no | 5 | 35 | - | Ciudad |
| `state_code` | string | sí | - | 10 | - | Sigla del estado. Debe pertenecer al país del remitente |
| `postal_code` | string | sí | - | 12 | - | CEP/ZIP. Si `state_code` viene vacío, el estado se obtiene a partir de él |
| `name` | string | no | 5 | 35 | - | Nombre del contacto en el lugar |
| `email` | string | no | - | 100 | - | Correo electrónico del contacto (formato de correo válido) |
| `phone` | string | no | 10 | 25 | - | Teléfono del contacto |
| `phone_extension` | string | no (usa `""`) | - | 5 | - | Extensión |
| `federal_tax_id` | string | sí | - | 20 | **Obligatorio: CPF o CNPJ válido** | Documento del responsable del lugar |

Cómo se verifican el país y el estado:

| Situación | Comportamiento |
|---|---|
| `state_code` informado | Debe ser un estado del país del remitente (o el propio código del país). Si no: `Endereço enviado não pertence ao pais da remessa` |
| `state_code` vacío y `postal_code` informado (remitente BR) | El estado se obtiene por el CEP. CEP no localizado: `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| `state_code` vacío y `postal_code` informado (otros países) | El estado se obtiene por el código postal. No localizado: `Informações de endereço de coleta insuficientes ou inválidas` |
| `state_code` y `postal_code` vacíos | No hay verificación. Recomendamos enviar siempre los dos para evitar el rechazo del transportista |

### Payload mínimo

Forma A con `shipping` (UUID):

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirar en la recepcion"
}
```

Forma A con `awb_number`:

```json
{
  "awb_number": "1234567890",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirar en la recepcion"
}
```

### Payload máximo

Forma B (dirección alternativa, remitente en Brasil), con todos los campos aceptados:

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Retirar en el galpon 2, puerta lateral, hablar con Carlos",
  "address": {
    "street": "Calle Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpon 2",
    "city": "Ciudad Ejemplo",
    "state_code": "SP",
    "postal_code": "01000000",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

> Para usar el AWB, cambia `shipping` por `awb_number`. El `federal_tax_id` del ejemplo es ficticio: en
> envíos de Brasil, la API exige un CPF o CNPJ válido.

### Ejemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2026-03-10",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirar en la recepcion"
  }'
```

### Ejemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> La respuesta confirma solo el **registro** del pedido. La confirmación del transportista es asíncrona:
> consulta la programación hasta que el status pase a `success` y aparezca el `confirmation_code`.

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` cuando la programación fue registrada |
| `message` | string | Mensaje traducido según el header `lang` |

> No hay `data` en esta respuesta. Para obtener detalles, usa [consultar programación](#endpoint-consultar-programación).

### Reglas de fecha

Las reglas se verifican en este orden; la primera que falle define el mensaje (`400`).

| Regla | Detalle | Mensaje cuando falla |
|---|---|---|
| Fecha futura | `date` debe ser **posterior a hoy** (hoy no se acepta) | `A data de agendamento não pode ser menor que a data Atual` |
| Sin fin de semana | Sábado y domingo no se aceptan, para cualquier país | `Coleta não pode ser realizada em um final de semana` |
| Sin feriado nacional | Solo cuando el remitente está en Brasil | `Coleta não pode ser realizada durante um feriado nacional` |
| Límite de 7 días hábiles | La fecha no puede pasar del 7º día hábil después de hoy (para remitente en Brasil, los feriados nacionales no cuentan como día hábil) | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Feriados nacionales brasileños considerados (fechas fijas): **01/01, 21/04, 01/05, 07/09, 12/10, 02/11,
15/11, 20/11 y 25/12**. Los feriados móviles (ej.: Carnaval, Viernes Santo, Corpus Christi) y los feriados
estatales/municipales **no** son bloqueados por la API — evita esas fechas por tu cuenta.

### Orden de las validaciones

| # | Verificación | Status | Mensaje |
|---|---|---|---|
| 1 | Formato de los campos | `422` | Mensajes por campo en `data` |
| 2 | AWB localizado (si se envió `awb_number`) | `400` | `Agendamento não foi encontrado para esta remessa` |
| 3 | Fecha posterior a hoy | `400` | `A data de agendamento não pode ser menor que a data Atual` |
| 4 | Envío sin programación anterior | `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` |
| 5 | Envío existente y con status actual `shipped` | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 6 | Transportista con recogida integrada | `400` | `Esta transportadora não possui serviço de coleta integrado` |
| 7 | Reglas de fecha (fin de semana, feriado, 7 días hábiles) | `400` | Ver [Reglas de fecha](#reglas-de-fecha) |
| 8 | Envío creado por el usuario del token | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 9 | Dirección alternativa (si se envió) | `400` | Ver [Errores comunes](#errores-comunes) |

## Endpoint: consultar programación

- **Método y ruta:** `GET /v1/schedule/{shipping}`
- **Cuándo usar:** para acompañar la programación después del `POST` y antes de cancelar.

### Parámetros

| Parámetro | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `shipping` | path | string | **sí** | UUID de la cotización o número del AWB (envíos de los últimos 90 días) |

### Payload mínimo

Sin cuerpo. Solo el identificador en el path:

```
GET /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944
```

### Payload máximo

No hay otros parámetros. La alternativa es usar el AWB en lugar del UUID:

```
GET /v1/schedule/1234567890
```

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirar en la recepcion",
    "status": {
      "code": "processing",
      "description": "Agendamento da Remessa Recebida. Em breve será processada!",
      "reason": null
    }
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.confirmation_code` | string \| null | Número de confirmación de la recogida en el transportista. Solo es definitivo cuando `status.code` es `success`; antes de eso puede venir `null` o un valor provisional (ej.: `000000`) |
| `data.date` | string | Fecha de la recogida (`AAAA-MM-DD`) |
| `data.inicial_time` | string | Inicio de la ventana (`HH:MM:SS`). ⚠️ **La clave es `inicial_time`** en la respuesta, mientras que en el request el campo es `initial_time` |
| `data.final_time` | string | Fin de la ventana (`HH:MM:SS`) |
| `data.instruction` | string | Instrucción informada en la creación (o la de la predefinición, en la recogida automática) |
| `data.status.code` | string | Código del status de la programación (ver [Status de la programación](#status-de-la-programación)). Usa este campo en tu lógica |
| `data.status.description` | string | Texto informativo del último evento. **No se traduce en la consulta**: puede venir en el idioma en que se hizo el pedido, en inglés (ej.: `Pickup Scheduled Successfully!`, `Pickup Cancelled Successfully!`) o con el mensaje devuelto por el transportista, en caso de `error` |
| `data.status.reason` | string \| null | Código del motivo de cancelación (`001`–`008`). `null` mientras no haya pedido de cancelación |

## Endpoint: cancelar programación

- **Método y ruta:** `DELETE /v1/schedule/{shipping}/{reason}`
- **Cuándo usar:** cuando la recogida ya fue **confirmada** (status `success`) y necesita ser desprogramada.

### Parámetros

| Parámetro | Lugar | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | UUID de la cotización o número del AWB (envíos de los últimos 90 días) |
| `reason` | path | string | **sí** | `001` a `008` | Código del motivo (ver [Motivos de cancelación](#motivos-de-cancelación)). Se transmite al transportista |

### Payload mínimo

Sin cuerpo. Identificador y motivo en el path:

```
DELETE /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/006
```

### Payload máximo

No hay otros parámetros. La alternativa es usar el AWB en lugar del UUID:

```
DELETE /v1/schedule/1234567890/001
```

### Ejemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` cuando el pedido de cancelación fue registrado |
| `message` | string | Mensaje traducido según el header `lang` |

> Después del pedido, el status pasa a `canceling` con la descripción
> `Solicitação de cancelamento recebida! Em breve será processada!`, luego a `terminating` y, por último, a
> `canceled` (o `error`). Consulta la programación para acompañar.

### Reglas de la cancelación

| Regla | Status | Mensaje cuando falla |
|---|---|---|
| El `reason` debe ser uno de los códigos `001` a `008` | `422` | `O campo reason não contém um valor válido.` (en `data.reason`) |
| El AWB debe ser localizado (si se usa en el path) | `400` | `Agendamento não foi encontrado para esta remessa` |
| La programación debe existir | `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| El envío debe continuar con status actual `shipped` | `400` | ídem |
| La programación debe estar en `success` (ya confirmada). En `processing`, `canceling`, `terminating`, `canceled` o `error`, la cancelación es rechazada | `400` | ídem |

## Reglas de negocio

| Regla | Detalle |
|---|---|
| Status del envío | El status **actual** debe ser `shipped`, tanto para programar como para cancelar |
| Programación única | Cada envío acepta una única programación por la API. Una programación cancelada o con error **también** impide un nuevo `POST`; para reprogramar, contacta al soporte |
| Transportista | El servicio del envío debe tener recogida integrada. Cancelación integrada solo para DHL, FedEx, UPS y SkyPostal |
| Fecha | Posterior a hoy, día hábil, sin feriado nacional fijo (remitente BR), hasta 7 días hábiles |
| Ventana | `initial_time` y `final_time` en el formato `HH:MM:SS`; no hay validación de orden ni de horario comercial |
| Dirección alternativa | Las 12 claves presentes; en Brasil exige `number`, `district` y `federal_tax_id` válido (CPF o CNPJ) |
| País de la dirección | Siempre el país del remitente del envío. El `state_code` debe pertenecer a ese país |
| Tipo de cuenta | Persona física y PJ mono-CD: `address` ignorado; recogida en la dirección del remitente |
| Recogida automática | Con la predefinición de recogida automática activa, la programación es creada por la plataforma y el `POST` manual es rechazado |
| Propiedad | Programa y consulta solo envíos creados por el usuario del token |
| AWB | Resuelve solo envíos creados en los **últimos 90 días** (el más reciente con ese número) |
| Procesamiento | Inmediato para recogida mañana; programado (enviado 1 día antes) para fechas posteriores |

## Errores comunes

| Status | Significado/Mensaje | Cómo corregir |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Ya existe una programación (manual, automática, cancelada o con error). Consulta con `GET`; para reprogramar, contacta al soporte |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Confirmar que el status actual es `shipped` y que el envío es del usuario del token |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Llevar el bulto a un punto de entrega |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Usar una fecha a partir de mañana |
| `400` | `Coleta não pode ser realizada em um final de semana` | Elegir un día hábil |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Elegir otro día hábil |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Reducir la antelación |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Remitente BR: completar `number` y `district`. Otros países: verificar `postal_code` o enviar `state_code` |
| `400` | `O atributo federal_tax_id não é válido` | Remitente BR: informar un CPF o CNPJ válido |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Usar un `state_code` del país del remitente |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Verificar el `postal_code` o informar el `state_code` |
| `400` | `Agendamento não foi encontrado para esta remessa` | En el `POST`/`DELETE` con AWB: verificar el número, la antigüedad del envío (90 días) y si es de tu cuenta |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Cancela solo programaciones en `success` de envíos en `shipped` |
| `403` | `Has no permission!` | Solicitar al soporte el permiso de la operación (crear, consultar o cancelar programación) |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contactar al soporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificar el header `Authorization: Bearer ...` |
| `404` | `Agendamento não foi encontrado para esta remessa` | En el `GET` con UUID: el envío no tiene programación o no es de tu cuenta |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + mapa de campos en `data` | Corregir los campos listados (ej.: `O campo data é obrigatório.`, `A data informada para o campo :attribute não respeita o formato :format.`, `O valor selecionado para o campo shipping é inválido.`, `O campo reason não contém um valor válido.`) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | `POST`: verificar si se enviaron todas las claves de `address`. `GET` con AWB: el AWB no fue localizado (inexistente, con más de 90 días o de otra cuenta). En los demás casos, repetir con backoff y contactar al soporte |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Falla inesperada en el `DELETE`. Repetir con backoff; si persiste, contactar al soporte |

> Atención: en la **consulta por AWB** (`GET /v1/schedule/{awb}`), un AWB no localizado responde `500`, y no
> `404`. Prefiere consultar por el UUID de la cotización, que responde `404` cuando no hay programación.

## Observaciones finales
- Consulta siempre la programación antes de crear o cancelar: esto evita solicitudes redundantes y el error
  `já tem um agendamento em progresso`.
- Atención a la diferencia de nomenclatura: **`initial_time` en el request** e **`inicial_time` en la respuesta**.
- Las programaciones del tipo programado permanecen en `processing` hasta 1 día antes de la recogida. No es un error. Si el status
  no cambia después de la fecha de la recogida, contacta al soporte.
- Usa `status.code` (y no `status.description`) para la lógica de tu integración.
- Status del envío: [Seguimiento](api-rastreio.md). Documentos para la recogida: [Documentos](api-documentos.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
