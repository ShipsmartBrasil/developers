# Primeros Pasos

- [Volver al README del idioma](README.md)

## Visión general
Esta página reúne todo lo que es común a **todos** los endpoints de la API Pública de Shipsmart Global:
entornos, autenticación, idioma, formato de respuesta, versionado y tratamiento de errores.

Lee esta página antes que cualquier otra. Las demás páginas de la guía asumen que ya conoces
estas convenciones y no las repiten en detalle.

## Entornos

| Entorno | Base URL | Documentación interactiva |
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
| Expiración | El token no expira por tiempo; se revoca si la cuenta es bloqueada o el token regenerado |

### Errores de autenticación

| Status | Respuesta | Significado |
|---|---|---|
| `404` | `{"status":"warning","message":"..."}` | Token no encontrado/inválido |
| `403` | `{"status":"warning","message":"..."}` | Cuenta bloqueada |
| `403` | `{"status":"warning","message":"Has no permission!"}` | El token es válido, pero tu perfil no tiene permiso para la acción |

> Los permisos son granulares por acción (cotizar, enviar, cancelar, listar documentos, programar recogida, etc.).
> Si recibes `Has no permission!`, solicita la habilitación del permiso correspondiente al soporte.

## Idioma de las respuestas

Los mensajes de éxito, aviso y error se traducen según el header `lang`.

| Header | Valores aceptados | Predeterminado |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> El header es opcional. Si no se envía (o se envía con un valor desconocido), la API responde en portugués.

> **Sobre los ejemplos de esta guía:** todos los mensajes de respuesta reproducidos en las páginas siguientes
> usan el idioma predeterminado de la API (`pt`). Enviando `lang: en` o `lang: es`, el texto de `message` viene
> traducido, pero el `status` y el código HTTP son siempre los mismos.

## Convenciones de solicitud

| Ítem | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Codificación | UTF-8 |
| Fechas (respuesta) | `Y-m-d H:i:s` |
| Fechas (programación) | `Y-m-d` para fecha y `H:i:s` para hora |
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
un envío en la v2. Mezclar versiones retorna `422` con el mensaje
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
| `warning` | `400`, `403`, `404`, `406`, `422` | Una regla de negocio o validación impidió la operación. **El payload debe corregirse** |
| `error` | `500` | Fallo inesperado del lado del servidor. Inténtalo de nuevo; si persiste, contacta al soporte |

## Errores de validación (`422`)

Cuando un campo no pasa la validación, la respuesta trae `data` con el mapa `campo → lista de mensajes`.

```json
{
  "status": "warning",
  "message": "Validation Error",
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
| `400` | Una regla de negocio bloqueó la operación (ej.: tipo de pago inexistente, cotización expirada, ningún transportista elegible) | Leer `message` y corregir el escenario |
| `403` | Sin permiso o cuenta bloqueada | Solicitar la habilitación al soporte |
| `404` | Recurso no encontrado (token, envío, contrato no listado en la cotización) | Verificar el identificador informado |
| `406` | Estructura del payload incompatible con lo esperado | Revisar tipos y anidamiento de los campos |
| `422` | Fallo de validación de campos | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Preajustes de la cuenta (presets)

Algunas opciones pueden quedar **preconfiguradas en tu cuenta** en el panel. Cuando están configuradas, se
aplican automáticamente en la cotización y en el envío, y puedes omitir el campo en el payload.

| Preset | Afecta | Observación |
|---|---|---|
| Canal de venta (`channel_code`) | Cotización | Usado cuando no informas el canal |
| Impuesto (`tax`) | Cotización | `sender` o `receiver` |
| Seguro (`insurance`) | Cotización | - |
| Objeto (`object`) y tipo (`type`) | Cotización | - |
| Monedas (`currency_quote`, `currency_payment`) | Cotización | - |
| Sistema de medida (`measurement`) | Cotización | `metric` o `imperial` |
| Entrega residencial / no apilable | Cotización | - |
| Dirección del remitente | Cotización y envío | Aplicada como `address_code` |
| Flete declarado (`declared_freight`) | Cotización | Valor + moneda |
| Facturador (`biller`) | Envío | Dirección mostrada en la invoice |
| Agrupamiento de cajas | Cotización | `default`, `all_in_one` o `smartbox` |

**Precedencia:** lo que envías en el payload siempre vence al preset. Excepciones:
- Si informas `address_sender.country_code` en la cotización, el preset de dirección del remitente se descarta.
- Si informas `address_sender.street` en el envío, el preset de dirección del remitente se descarta.
- Si informas `declared_freight`, el preset de flete declarado se descarta.

### Agrupamiento de cajas
El agrupamiento define cómo se distribuyen los artículos en las cajas cuando usas `type: items`.
**No** es un campo del payload: es una configuración de tu cuenta.

| Modo | Comportamiento |
|---|---|
| `default` | Agrupamiento estándar de la plataforma |
| `all_in_one` | Consolida todo en una única caja |
| `smartbox` | Encaja los artículos en las cajas registradas considerando volumen y dimensiones (con 10% de holgura) y respetando `is_groupable` del artículo |

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
