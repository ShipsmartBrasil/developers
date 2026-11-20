# Primeros Pasos

- [Volver al README del idioma](README.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
Esta página reúne todo lo que es común a **todos** los endpoints de la API Pública de Shipsmart Global:
entornos, autenticación, idioma, formato de respuesta, versionado y tratamiento de errores.

Lee esta página antes que cualquier otra. Las demás páginas de la guía asumen que ya conoces
estas convenciones y no las repiten en detalle.

## Entornos

| Entorno | URL base | Documentación interactiva |
|---|---|---|
| Producción | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Homologación (staging) | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

- Usa **homologación** para desarrollar y probar la integración. Ningún envío creado allí se despacha de verdad.
- Las nuevas features aparecen primero en homologación. Si un campo descrito aquí todavía no existe en producción,
  valídalo en `https://hmlapi.shipsmart.com.br/doc`.
- Acceder a la raíz (`/`) redirige a `/doc`.

## Autenticación

Toda llamada (excepto la documentación) exige un token de acceso personal en el header `Authorization`,
en el esquema **Bearer**.

```
Authorization: Bearer SU_TOKEN
```

| Ítem | Valor |
|---|---|
| Tipo | Bearer token |
| Header | `Authorization` |
| Dónde obtenerlo | Panel de Shipsmart Global, en el área de configuraciones de tu cuenta |
| Expiración | El token no expira por tiempo; se revoca si la cuenta es bloqueada o si el token se regenera |

### Errores de autenticación

| Status | Respuesta | Significado |
|---|---|---|
| `404` | `{"status":"warning","message":"Nenhum usuário encontrado para o token especificado!"}` | Token ausente, no encontrado o inválido |
| `403` | `{"status":"warning","message":"Usuário especificado encontra-se bloqueado em nosso sistema!"}` | Cuenta bloqueada |
| `403` | `{"status":"warning","message":"Has no permission!"}` | El token es válido, pero tu perfil no tiene permiso para la acción |

> Los permisos son granulares por acción (cotizar, enviar, cancelar, listar documentos, programar recogida, etc.).
> Si recibes `Has no permission!`, solicita al soporte la habilitación del permiso correspondiente.

## Idioma de las respuestas

Los mensajes de éxito, aviso y error se traducen según el header `lang`.

| Header | Valores aceptados | Predeterminado |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> El header es opcional. Si no se envía, la API responde en portugués. Si se envía con un valor
> no soportado (ej.: `fr`), los mensajes vienen en inglés.

> **Sobre los ejemplos de esta guía:** todos los mensajes de respuesta reproducidos en las páginas siguientes
> usan el idioma predeterminado de la API (`pt`). Enviando `lang: en` o `lang: es`, el texto de `message` viene
> traducido, pero el `status` y el código HTTP son siempre los mismos.

## Convenciones de solicitud

| Ítem | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Codificación | UTF-8 |
| Fechas (respuesta) | `AAAA-MM-DD HH:MM:SS` |
| Fechas (programación) | `AAAA-MM-DD` para la fecha y `HH:MM:SS` para la hora |
| Monedas | Código ISO 4217 con 3 letras (ej.: `BRL`, `USD`, `EUR`) |
| Países | Código ISO 3166-1 alpha-2 con 2 letras (ej.: `BR`, `US`, `PT`) |
| Separador decimal | Punto (`.`) — nunca coma |
| Pesos | Kilogramos cuando `measurement` es `metric`; libras cuando es `imperial` |
| Dimensiones | Centímetros cuando `measurement` es `metric`; pulgadas cuando es `imperial` |
| Volumen | m³ cuando `measurement` es `metric`; ft³ cuando es `imperial` |

## Versionado

La versión forma parte del path: `/v1/...` y `/v2/...`.

- `POST /v1/quotation` → cotización en la primera versión
- `POST /v2/quotation` → cotización en la segunda versión

### Llamadas sin prefijo de versión
Si llamas a un recurso **sin** el prefijo de versión, la API responde `308 Permanent Redirect`
apuntando al mismo recurso en `/v1`, preservando la query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> Para integraciones nuevas, informa siempre la versión explícitamente. No dependas de la redirección.

### Compatibilidad entre versiones
Una cotización creada en la v1 **solo** puede generar un envío en la v1, y una cotización creada en la v2 **solo** puede generar
un envío en la v2. Mezclar versiones retorna `422` con el mensaje (en `data.quotation`)
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

Consulta [Cotización y Envío: v1 vs v2](api-comparativo-cotacao-remessa.md) para elegir la versión.

## Formato estándar de las respuestas

Todas las respuestas siguen el mismo envelope:

```json
{
  "status": "success",
  "message": "Mensaje legible para humanos",
  "data": {}
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success`, `warning` o `error` |
| `message` | string | Mensaje ya traducido según el header `lang` |
| `data` | object/array | Contenido de la respuesta. Ausente en respuestas que solo confirman la recepción |

### Relación entre `status` y HTTP

| `status` | HTTP típico | Lectura |
|---|---|---|
| `success` | `200` | La operación fue aceptada/concluida |
| `warning` | `400`, `401`, `403`, `404`, `406`, `422` | Una regla de negocio o validación impidió la operación. **El payload debe corregirse** |
| `error` | `500` (y algunos rechazos de peso/transportista con `400`/`404`) | Fallo inesperado o rechazo del transportista. Lee `message`; los errores `500` pueden repetirse |

## Errores de validación (`422`)

Cuando un campo no pasa la validación, la respuesta trae `data` con el mapa `campo → lista de mensajes`.
El `message` de nivel superior es **genérico** (no describe el error): usa siempre el contenido de `data`.

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "object": ["O campo object é obrigatório."],
    "address_sender.country_code": [
      "O campo address sender.country code é obrigatório quando address sender.address code não está presente."
    ],
    "boxes": ["O campo boxes é obrigatório a menos que type esteja presente em items."]
  }
}
```

> En campos anidados, la clave usa punto (`address_sender.country_code`) y, en arrays,
> usa el índice (`items.0.hscode`). Usa esto para señalar el error en tu formulario/integración.

## Tabla general de status HTTP

| Status | Cuándo ocurre | Qué hacer |
|---|---|---|
| `200` | Solicitud aceptada/procesada | Seguir el flujo |
| `308` | Llamada sin prefijo de versión | Usar `/v1/...` o `/v2/...` |
| `400` | Una regla de negocio bloqueó la operación (ej.: documento fiscal inválido, caja sin artículo, código postal inválido) | Leer `message` y corregir el escenario |
| `401` | Tipo de pago no habilitado para la cuenta | Habilitar la opción de pago o usar otro tipo |
| `403` | Sin permiso, cuenta bloqueada, cotización simple o expirada en el envío | Solicitar la habilitación al soporte / rehacer la cotización |
| `404` | Recurso no encontrado (token, cotización de otra cuenta, envío, contrato no listado en la cotización, dirección) | Verificar el identificador informado |
| `406` | Conflicto de negocio (cotización ya utilizada, dirección registrada incompatible, carga que exige atención personalizada) | Leer `message` y ajustar el flujo |
| `422` | Fallo de validación de campos, o regla de negocio sobre los datos (ej.: HS Code sin tabla de impuesto, cuenta solo DDP) | Corregir los campos listados en `data` / leer `message` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Predefiniciones de la cuenta (presets)

Algunas opciones pueden quedar **preconfiguradas en tu cuenta** en el panel. La API las aplica después de la
validación del payload, con reglas diferentes para cada campo:

| Predefinición | Afecta | Cómo se aplica |
|---|---|---|
| Impuesto (`tax`) | Cotización, Manifiesto, Borrador | **Sustituye** el `tax` enviado. Los países de destino configurados como excepción reciben lo opuesto al valor predeterminado. Si la cuenta está **bloqueada en DDP**, el impuesto es siempre `sender` |
| Seguro automático | Cotización, Manifiesto | Contrata el seguro cuando el valor declarado (convertido a USD) alcanza el mínimo configurado, incluso con `insurance: false` |
| Canal de venta (`channel_code`) | Cotización, Manifiesto, Borrador | Usado cuando el campo no se envía |
| Sistema de medida (`measurement`) | Cotización, Manifiesto, Borrador | Usado cuando el campo no se envía; en su ausencia, `metric` |
| Entrega residencial / no apilable | Cotización, Borrador | Usados cuando los campos no se envían |
| Flete declarado (`declared_freight`) | Cotización, Manifiesto, Borrador | Usado cuando el campo no se envía. Exige el permiso de flete declarado |
| Agrupación de cajas | Cotización v2 (`type: items`) | Define cómo se montan las cajas (ver abajo). No es un campo del payload |
| Centro de costo (`cost_center_code`) | Envío v1/v2, Manifiesto, Borrador | Usado cuando el campo no se envía **o se envía vacío/nulo** |
| Finalidad del envío (`type` del envío) | Envío v1/v2, Borrador | Usada cuando el campo no se envía |
| Facturador (`biller`) | Envío v1/v2, Manifiesto, Borrador | Usado cuando el campo no se envía (o viene vacío) y la cuenta está configurada para mostrar el facturador |
| Notificación de cobro (`payment_notify_shipping_billing`) | Envío v1/v2 | Usada cuando el campo no se envía |
| Recogida automática | Envío | Crea la programación de recogida automáticamente (ver [Programación de Recogida](api-agendamento.md)) |

**Precedencia:** excepto el **impuesto** (que siempre sigue la predefinición, cuando está configurada), lo que envías
en el payload vence a la predefinición. Los campos obligatorios del payload (`object`, `type`, `insurance`, monedas,
`address_sender`) deben enviarse siempre — la predefinición no los dispensa.

> La dirección del remitente preconfigurada en el panel **no** sustituye el `address_sender` de la API.
> Para reutilizar una dirección registrada, envía `address_sender.address_code`.

### Agrupación de cajas
La agrupación define cómo se distribuyen los artículos en las cajas cuando usas `type: items`
(cotización por artículos, **exclusiva de la v2**).
**No** es un campo del payload: es una configuración de tu cuenta.

| Modo | Comportamiento |
|---|---|
| `default` | Cada artículo usa la caja vinculada a su registro (`box_code`/`box_identifier`); los artículos agrupables con la misma caja se reúnen y los artículos no agrupables reciben una caja por unidad |
| `all_in_one` | Consolida los artículos agrupables en una única caja virtual con el volumen total (a partir de las dimensiones registradas); los artículos no agrupables reciben una caja por unidad |
| `smartbox` | Encaja los artículos en las cajas **enviadas en `boxes[]`** (obligatorio), considerando volumen y dimensiones con 10% de holgura y respetando `is_groupable`/`is_foldable` del artículo |

Ejemplos de cada modo en [Cotización de Artículos](api-cotacao-v2.md#cotización-de-artículos).

## Registros hechos en el panel

Algunos códigos usados en los payloads provienen de registros mantenidos **en el panel de Shipsmart Global** (no hay endpoint
público para crearlos). Deben existir, estar activos y pertenecer a tu cuenta (o a las cuentas autorizadas):

| Registro | Campo en el payload | Dónde se usa |
|---|---|---|
| Dirección | `address_sender.address_code`, `address_receiver.address_code`, `manufacturer.address_code`, `biller.address_code` | Cotización, Envío, Manifiesto, Borrador |
| Canal de venta | `channel_code` | Cotización, Manifiesto, Borrador, consultas y cancelación por canal |
| Centro de costo | `cost_center_code` | Envío (creación y actualización), Manifiesto, Borrador |
| Kit | `boxes[].kit` | Cotización, Manifiesto, Borrador |

Las cajas y los artículos también pueden registrarse por la API: consulta [Caja](api-caixa-v1.md) y [Artículos](api-itens-v1.md).

## Monedas aceptadas

Misma lista para `currency_quote` (moneda de la cotización) y `currency_payment` (moneda del pago):

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> `currency_payment` con pago vía gateway tiene disponibilidad dependiente del proveedor:
> `BRL` es procesado por un proveedor y las demás monedas por otro. Si la moneda elegida no está
> habilitada para tu cuenta, el envío retorna `warning` en el momento del pago.

## Primera llamada (prueba de conectividad)

El endpoint más simple para validar token y conectividad:

```bash
curl --request GET \
  --url https://hmlapi.shipsmart.com.br/v1/platform/version \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Accept: application/json'
```

Respuesta:

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "staging",
    "version": "1.0.0"
  }
}
```

## Buenas prácticas de integración

1. **Guarda siempre el `quotation`** retornado en la cotización: es la clave de todo el resto del flujo
   (envío, status, documentos, programación).
2. **Guarda también tu identificador comercial** en `channel_identifier_number`. Esto permite consultar
   status, documentos y cancelar usando el número de tu pedido, sin depender del `quotation`.
3. **Nunca hagas polling agresivo.** Las consultas de status/documentos deben tener un intervalo de algunos
   segundos y backoff exponencial.
4. **Trata `warning` como error de negocio** (no repitas la misma solicitud) y `error` como transitorio
   (puedes repetir).
5. **Respeta los límites de tamaño de campo** descritos en cada página. Los textos más grandes son rechazados
   con `422`, no truncados.
6. **Las cotizaciones expiran.** Crea el envío dentro de la validez informada en `valid_until` de cada transportista.

## Próximos pasos

- [Cotización](api-cotacao-v1.md) o [Cotización V2](api-cotacao-v2.md)
- [Envío](api-remessa-v1.md) o [Envío V2](api-remessa-v2.md)
- [Cotización y Envío: v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Códigos de Error](api-codigos-de-erro.md)
