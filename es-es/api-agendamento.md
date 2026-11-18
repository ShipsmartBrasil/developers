# Programación de Recogida

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md) y [Envío (v1)](api-remessa-v1.md)

## Visión general
- Solicita al transportista que **retire** el bulto en tu dirección, en lugar de que tú lo lleves a un punto
  de envío.
- Permite consultar el status de la programación y cancelarla con un motivo estandarizado.
- Funciona para envíos creados en la **v1 y en la v2**.

## Base URL
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
Content-Type: application/json
lang: pt
```

## Prerrequisitos

Antes de programar, confirma que:

1. El envío ya está en el status **`shipped`** ([seguimiento](api-rastreio.md)) — es decir, ya fue procesado en el
   transportista y los documentos existen.
2. El transportista del contrato elegido **posee servicio de recogida integrado**.
3. El envío **no tiene** otra programación en curso.

## Flujo de uso

```
1. GET  /v1/shipping/{shipping}/status   → confirmar el status "shipped"
2. POST /v1/schedule                     → crear la programación
3. GET  /v1/schedule/{shipping}          → acompañar hasta el status "success"
4. DELETE /v1/schedule/{shipping}/{reason} → cancelar (si es necesario)
```

## Convenciones
- Fechas en el formato `Y-m-d` (ej.: `2026-03-10`).
- Horarios en el formato `H:i:s` (ej.: `08:00:00`).
- El identificador `{shipping}` acepta el **UUID de la cotización** o el **número del AWB** (envíos de los
  últimos 90 días). A diferencia de los endpoints de envío, **la clave de la NF-e no se acepta aquí**.

## Endpoint: crear programación

- **Método y ruta:** `POST /v1/schedule`
- **Cuándo usarlo:** después de que el envío alcance `shipped`.

### Parámetros

| Campo | Tipo | Obligatorio | Mín. | Máx. | Formato | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | string | **sí** si no se envía `awb_number` | - | - | UUID | Código de la cotización/envío |
| `awb_number` | string | **sí** si no se envía `shipping` | - | 50 | - | Número del AWB (envíos de los últimos 90 días) |
| `date` | string | **sí** | - | - | `Y-m-d` | Fecha de la recogida. Debe ser **mayor que hoy** |
| `initial_time` | string | **sí** | - | - | `H:i:s` | Inicio de la ventana de recogida |
| `final_time` | string | **sí** | - | - | `H:i:s` | Fin de la ventana de recogida |
| `instruction` | string | **sí** | 5 | 80 | - | Instrucción para el conductor (ej.: dónde retirar, con quién hablar) |
| `address` | object | no | - | - | - | Dirección alternativa de recogida (ver tabla abajo) |

> Envía **`shipping` o `awb_number`**, no los dos.

### Parámetros — `address` (dirección alternativa de recogida)

Envíalo solo si la recogida es en una dirección **diferente** a la del remitente del envío.
**La dirección debe estar en el mismo país del remitente.**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `street` | string | **sí** si se envía `address` | 5 | 70 | Calle |
| `city` | string | **sí** si se envía `address` | 5 | 35 | Ciudad |
| `name` | string | **sí** si se envía `address` | 5 | 35 | Nombre del contacto en el lugar |
| `email` | string | **sí** si se envía `address` | - | 100 | E-mail del contacto |
| `phone` | string | **sí** si se envía `address` | 10 | 25 | Teléfono del contacto |
| `postal_code` | string | condicional | - | 12 | Código postal/ZIP. **Obligatorio para recogidas en Brasil** |
| `state_code` | string | no | - | 10 | Sigla del estado. Debe pertenecer al país del remitente |
| `number` | string | condicional | - | 6 | Número del inmueble. **Obligatorio para recogidas en Brasil** |
| `district` | string | condicional | - | 30 | Barrio. **Obligatorio para recogidas en Brasil** |
| `complement` | string | no | - | 20 | Complemento |
| `phone_extension` | string | no | - | 5 | Extensión |
| `federal_tax_id` | string | condicional | - | 20 | CPF/CNPJ. **Obligatorio y validado para recogidas en Brasil** |

> Cuando se omite `address`, la plataforma resuelve la dirección automáticamente: primero el preajuste de
> recogida de tu cuenta y, en su ausencia, la dirección del remitente del envío.
>
> En algunos tipos de cuenta (persona física y PJ mono-CD), la dirección alternativa es **ignorada** y la recogida
> usa siempre la dirección del remitente.

### Payload mínimo

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirar en la recepcion, hablar con Carlos"
}
```

### Payload máximo

```json
{
  "awb_number": "271249639538",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Retirar en el galpon 2, porton lateral, hablar con Carlos",
  "address": {
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Calle Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpon 2",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

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
    "instruction": "Retirar en la recepcion, hablar con Carlos"
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> La respuesta confirma la **recepción**. La confirmación del transportista es asíncrona: consulta la
> programación hasta que el status cambie a `success` y aparezca el `confirmation_code`.

### Reglas de fecha y ventana

| Regla | Detalle | Mensaje cuando falla |
|---|---|---|
| Fecha futura | `date` debe ser **mayor que la fecha actual** | `A data de agendamento não pode ser menor que a data Atual` |
| Sin fin de semana | Sábado y domingo no se aceptan | `Coleta não pode ser realizada em um final de semana` |
| Sin feriado nacional | Para recogidas en Brasil | `Coleta não pode ser realizada durante um feriado nacional` |
| Límite de 7 días hábiles | La fecha no puede superar 7 días hábiles a partir de hoy | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Feriados nacionales brasileños considerados: **01/01, 21/04, 01/05, 07/09, 12/10, 02/11, 15/11, 20/11 y 25/12**.

## Endpoint: consultar programación

- **Método y ruta:** `GET /v1/schedule/{shipping}`

| Parámetro | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `shipping` | path | string | **sí** | UUID de la cotización o número del AWB (últimos 90 días) |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirar en la recepcion, hablar con Carlos",
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
| `data.confirmation_code` | string | Número de confirmación de la recogida en el transportista. Se completa cuando el status llega a `success` |
| `data.date` | string | Fecha de la recogida (`Y-m-d`) |
| `data.inicial_time` | string | Inicio de la ventana (`H:i:s`). ⚠️ **La clave es `inicial_time`** en la respuesta, mientras que en el request el campo es `initial_time` |
| `data.final_time` | string | Fin de la ventana (`H:i:s`) |
| `data.instruction` | string | Instrucción informada en la creación |
| `data.status.code` | string | Código del status de la programación (ver tabla) |
| `data.status.description` | string | Descripción traducida según el header `lang` |
| `data.status.reason` | string \| null | Código del motivo de cancelación (`001`–`008`). `null` mientras no haya cancelación |

### Status de la programación

| `code` | Significado |
|---|---|
| `processing` | Programación recibida, esperando el envío al transportista |
| `success` | **Recogida confirmada por el transportista.** `confirmation_code` disponible |
| `canceling` | Cancelación solicitada, esperando procesamiento |
| `terminating` | Cancelación en procesamiento en el transportista |
| `canceled` | Cancelación concluida |
| `error` | Fallo en la programación o en la cancelación |

### Tipo de procesamiento

La plataforma clasifica la programación automáticamente, sin campo en el payload:

| Tipo | Cuándo | Comportamiento |
|---|---|---|
| `automatico` | Recogida para el día siguiente (hasta 1 día) | Enviado al transportista inmediatamente |
| `agendado` | Recogida con más de 1 día de antelación | Enviado al transportista 1 día antes de la recogida |

> Por eso una programación futura puede permanecer en `processing` durante días. Esto es esperado.

## Endpoint: cancelar programación

- **Método y ruta:** `DELETE /v1/schedule/{shipping}/{reason}`
- **Cuándo usarlo:** cuando la recogida ya fue **confirmada** (status `success`) y necesita ser desprogramada.

| Parámetro | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `shipping` | path | string | **sí** | UUID de la cotización o número del AWB (últimos 90 días) |
| `reason` | path | string | **sí** | Código del motivo (ver tabla). Enviado al transportista |

### Motivos de cancelación

| Código | Motivo |
|---|---|
| `001` | Package not ready (el bulto no está listo) |
| `002` | Rates too high (tarifas muy altas) |
| `003` | Transit time too slow (plazo de tránsito muy lento) |
| `004` | Take to service center or drop box (será llevado a un punto de envío) |
| `005` | Commitment time not met (plazo acordado no cumplido) |
| `006` | Reason not given (motivo no informado) |
| `007` | Other (otro) |
| `008` | Pickup modified (recogida modificada) |

> Esos códigos están estandarizados por los transportistas y se les transmiten sin traducción.
> Usa `006` o `007` cuando ninguno de los específicos se aplique.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Reglas de la cancelación

| Regla | Mensaje cuando falla |
|---|---|
| El `reason` debe estar entre `001` y `008` | `Motivo do cancelamento não é compativel` |
| La programación debe existir | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| El envío debe estar en `shipped` | ídem |
| La programación debe estar en `success` (ya confirmada por el transportista) | ídem |

> Después de la solicitud, el status pasa a `canceling` con la descripción
> `Solicitação de cancelamento recebida! Em breve será processada!`, luego `terminating` y, por último,
> `canceled` (o `error`).

## Reglas de negocio consolidadas

| Regla | Detalle |
|---|---|
| Status del envío | El envío debe estar en `shipped` |
| Programación única | No es posible tener dos programaciones activas en el mismo envío |
| Transportista | El servicio debe soportar recogida integrada |
| Fecha | Futura, día hábil, sin feriado nacional (BR), hasta 7 días hábiles |
| Dirección BR | Exige `postal_code`, `number`, `district` y `federal_tax_id` válido (CPF o CNPJ) |
| País de la dirección | El `state_code` debe pertenecer al país del remitente del envío |
| Propiedad | Solo es posible programar/consultar/cancelar envíos de tu propia cuenta |
| AWB | Resuelve solo envíos de los **últimos 90 días** |

## Errores comunes

| Status | Mensaje | Cómo corregirlo |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Consultar la programación existente antes de crear otra |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Esperar el status `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Llevar el bulto a un punto de envío |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Usar una fecha futura |
| `400` | `Coleta não pode ser realizada em um final de semana` | Elegir un día hábil |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Elegir otro día hábil |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Reducir la antelación |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Completar `number` y `district` (recogidas en Brasil) |
| `400` | `O atributo federal_tax_id não é válido` | Informar un CPF o CNPJ válido |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Usar un `state_code` del país del remitente |
| `400` | `Motivo do cancelamento não é compativel` | Usar un código entre `001` y `008` |
| `403` | Sin permiso | Solicitar el permiso de programación al soporte |
| `404` | `Agendamento não foi encontrado para esta remessa` | Verificar el identificador o crear la programación |
| `422` | Validación | Corregir fecha/hora/instrucción conforme a los formatos |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Consulta siempre la programación antes de crear o cancelar: eso evita solicitudes redundantes y el error
  `já tem um agendamento em progresso`.
- Atención a la diferencia de nomenclatura: **`initial_time` en el request** e **`inicial_time` en la respuesta**.
- Las programaciones futuras permanecen en `processing` hasta 1 día antes de la recogida. No es un error.
