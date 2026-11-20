# Borrador

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Permite registrar un envío **incompleto**: envías lo que ya tienes, y la plataforma lo valida después.
- En la entrada, **todos los campos son opcionales** (solo se verifica el formato de cada campo). La validación
  estricta ocurre después, en el **despacho**, que se ejecuta automáticamente en ciclos de pocos minutos.
- Cuando el borrador pasa el despacho, la plataforma genera la **cotización + envío** por sí sola y el borrador
  **deja de existir** (se elimina). Mientras falten datos, queda `incomplete` y puede corregirse.
- Es el flujo indicado cuando la integración recibe los datos del pedido por etapas (ej.: el pedido llega antes
  de la dirección final o de la nota fiscal).
- Las **predefiniciones de la cuenta** (presets) se aplican al borrador, incluido el responsable del
  impuesto, que **sustituye** el valor enviado cuando está configurado.

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

El header `lang` acepta `pt`, `en` y `es` (predeterminado `pt`). Cada operación exige un permiso propio
en el perfil (crear, editar, listar y eliminar borradores). Sin él, la respuesta es `403 Has no permission!`.

## Flujo de uso
1. `POST /v1/draft` con los datos que ya tienes. La respuesta solo confirma la recepción
   (**no devuelve el código del borrador**).
2. Espera algunos minutos: el borrador se guarda y después pasa por el despacho automático.
3. `GET /v1/draft` para localizar el borrador (por tu `channel_identifier_number`), obtener el `codigo`
   y ver el `status`. Si está `incomplete`, lee el `status_descricao`.
4. `PUT /v1/draft/{code}` enviando **el borrador completo y corregido** (el `PUT` sustituye todo el contenido).
5. Repite los pasos 2 a 4 hasta que el borrador desaparezca del listado: eso significa que la cotización y el envío fueron creados.
6. Haz el seguimiento del envío por el [rastreo](api-rastreio.md) y descarga los [documentos](api-documentos.md),
   usando el `channel_identifier_number` (y el `channel_code`, si lo hay).
7. `DELETE /v1/draft` para descartar borradores que no se usarán (o que quedaron con status `error`).

## Conceptos esenciales

### Borrador vs. otros flujos

| Tema | Cotización + Envío | Manifiesto | Borrador |
|---|---|---|---|
| Payload | Completo en 2 etapas | Completo en 1 llamada | **Parcial, en tantas etapas como necesites** |
| Validación | Síncrona, por etapa | Síncrona, total | **Formato en la entrada; reglas completas en el despacho (asíncrono)** |
| Respuesta | Cotización con precios | Documentos listos | Confirmación de recepción (sin código) |
| Pago | `billed`, `gateway`, `afterward` | `billed` | **`billed` o `afterward`** |
| Elección del contrato | Tú eliges | Tú eliges | **Automática** (`carrier_code` preferido o preferencia de la cuenta) |
| Corrección posterior | Nueva cotización | No | **Sí, vía `PUT` (solo en `incomplete`)** |

### Ciclo de vida del borrador

```
POST /v1/draft                      PUT /v1/draft/{code}
     │                                       │
     ▼                                       ▼
  received ──► validating ──► incomplete ──► edited ──► validating ──► ...
                    │
                    ▼
                 complete ──► processing ──► cotización + envío creados (borrador eliminado)
                                  │
                                  ├──► incomplete  (una regla de negocio lo rechazó: motivo en status_descricao)
                                  └──► error       (fallo inesperado: motivo en status_descricao)
```

| Status | Significado | ¿Se puede editar (`PUT`)? |
|---|---|---|
| `received` | Borrador recibido, esperando el próximo ciclo de despacho | No |
| `edited` | Borrador actualizado vía `PUT`, esperando el próximo ciclo de despacho | No |
| `validating` | En validación | No |
| `incomplete` | **Faltan datos o alguna regla rechazó el envío.** El motivo está en `status_descricao` | **Sí** |
| `complete` | Pasó la validación, listo para generar cotización y envío | No |
| `processing` | Generando cotización y envío | No |
| `error` | Fallo inesperado en el procesamiento | No (elimínalo y créalo de nuevo) |

| Detalle | Comportamiento |
|---|---|
| Frecuencia del despacho | Los borradores `received` y `edited` se validan automáticamente cada ~3 minutos |
| Éxito | La cotización y el envío se crean y **el borrador se elimina** (ya no aparece en el `GET`) |
| Borrador atascado en `processing` | Tras ~15 minutos sin conclusión, vuelve a `incomplete` con `status_descricao` vacío |
| Tamaño del `status_descricao` | Limitado a **254 caracteres**: la lista de pendientes puede venir cortada |
| Formato del `status_descricao` | En la validación de campos, trae el primer mensaje de cada campo, concatenados sin separador |

> **Solo se pueden editar los borradores con status `incomplete`.** En cualquier otro status la respuesta es
> `400 Apenas rascunhos com o status incomplete podem ser editados`.

### El `PUT` sustituye el borrador entero

El `PUT /v1/draft/{code}` **no** combina con el contenido anterior: el contenido guardado se **reemplaza** por el
payload enviado (ya combinado con los presets de la cuenta). Los campos que no reenvíes **se pierden**.

- Envía siempre el borrador completo, con las correcciones.
- Atención al construir el `PUT` a partir del `GET`: el listado **no devuelve** `measurement`,
  `residential_delivery`, `non_stackable`, `cost_center_code`, `declared_freight`, `biller`,
  `manufacturer` ni `address_sender.address_code`. Reenvíalos a partir de tus propios datos.

### Predefiniciones de la cuenta (presets) en el borrador

Los presets configurados en el panel se combinan con el payload en el `POST` **y** en el `PUT`
(consulta [Predefiniciones de la cuenta](api-primeiros-passos.md#predefiniciones-de-la-cuenta-presets)).

| Campo | Preset usado cuando | Observación |
|---|---|---|
| `channel_code` | El campo **no** se envía | Enviar el campo (incluso `null`) impide el preset |
| `cost_center_code` | El campo no se envía **o** viene vacío/`null` | Único campo en el que un valor vacío también recibe el preset |
| `carrier_code` | El campo no se envía | - |
| `tax` | **Siempre que haya preset de impuesto** | El preset **sustituye** el valor enviado (ver abajo) |
| `insurance` | El campo no se envía | - |
| `measurement` | El campo no se envía o viene `null` | Sin preset, asume `metric` |
| `currency_quote`, `currency_payment` | El campo no se envía | - |
| `residential_delivery`, `non_stackable` | El campo no se envía | - |
| `type` (finalidad del envío) | El campo no se envía | - |
| `address_sender` | El objeto `address_sender` **no** se envía | El preset entra como `address_code`. Enviar `address_sender` con cualquier campo descarta el preset |
| `biller` | El objeto no se envía **o** viene vacío, y la cuenta está configurada para mostrar el facturador | Si envías `biller` rellenado, siempre vale |
| `declared_freight` | El objeto no se envía | Flete declarado transferido a la cotización |

> La combinación se hace **en el primer nivel**: un objeto enviado (`address_sender`, `biller`,
> `declared_freight`) sustituye el objeto del preset por completo, sin combinar campo por campo.

### Responsable del impuesto (`tax`)

Cuando la cuenta tiene preset de impuesto, **el valor enviado en `tax` se ignora** y la plataforma resuelve
el responsable así:

| Configuración del preset de impuesto | Valor guardado en el borrador |
|---|---|
| Sin preset de impuesto | El `tax` enviado. Si no lo envías, el despacho indica que `tax` es obligatorio |
| Predeterminado `sender` | `sender`, aunque envíes `receiver` |
| Predeterminado `receiver` | `receiver`, aunque envíes `sender` |
| Predeterminado definido y país de destino en la lista de excepciones | Lo **opuesto** al predeterminado (`sender` ↔ `receiver`) |
| Modo bloqueado | Siempre `sender` |

- El país de destino considerado es el `address_receiver.country_code` **del mismo payload**. Si no se
  envía, la excepción por país no se evalúa y vale el predeterminado.
- Como el `PUT` vuelve a aplicar los presets, el impuesto se recalcula en cada actualización.

## Endpoint: crear borrador

- **Método y ruta:** `POST /v1/draft`
- **Cuándo usarlo:** para registrar un envío, incluso con datos faltantes.
- **Procesamiento:** asíncrono. La respuesta `200` solo confirma la recepción; el borrador aparece en el
  `GET` algunos instantes después.

### Parámetros — raíz del payload

"Entrada" = regla verificada en la llamada (error `422`). "Despacho" = regla verificada después, que deja
el borrador `incomplete`.

| Campo | Tipo | Entrada | Despacho | Mín. | Máx. | Predeterminado | Valores / descripción |
|---|---|---|---|---|---|---|---|
| `channel_identifier_number` | string | opcional | opcional | - | 100 | - | Tu identificador del pedido. **Recomendado**: es por él que localizas el borrador y el envío |
| `channel_code` | integer | opcional | opcional | - | - | preset de la cuenta | Canal de venta registrado. En el despacho debe pertenecer a tu cuenta |
| `cost_center_code` | integer | opcional | opcional | - | - | preset de la cuenta | Centro de costo registrado de tu cuenta (o de cuentas autorizadas) |
| `carrier_code` | integer | opcional | opcional | - | - | preset de la cuenta | Contrato preferido. Se usa si está disponible en el resultado de la cotización |
| `object` | string | opcional | opcional | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | string | opcional | opcional | - | 30 | `Commercial` (o preset) | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `tax` | string | opcional | **obligatorio** | - | - | preset de la cuenta | `sender`, `receiver`. Ver [Responsable del impuesto](#responsable-del-impuesto-tax) |
| `insurance` | boolean | opcional | **obligatorio** | - | - | preset de la cuenta | Contratar seguro |
| `measurement` | string | opcional | opcional | - | - | preset o `metric` | `metric`, `imperial` |
| `ior` | boolean | opcional | opcional | - | - | `false` | Importer of Record |
| `residential_delivery` | boolean | opcional | opcional | - | - | `false` (o preset) | Entrega residencial |
| `non_stackable` | boolean | opcional | opcional | - | - | `false` (o preset) | Bultos no apilables |
| `currency_quote` | string | opcional | **necesario** (ver nota) | 3 | 3 | preset de la cuenta | Moneda de la cotización. Ver [monedas](api-primeiros-passos.md#monedas-aceptadas) |
| `currency_payment` | string | opcional | **necesario** (ver nota) | 3 | 3 | preset de la cuenta | Moneda del pago. Ver [monedas](api-primeiros-passos.md#monedas-aceptadas) |
| `nfkey` | string | opcional | opcional | 44 | 44 | - | Clave de la NF-e |
| `description` | string | opcional | opcional | - | 155 | - | Descripción libre |
| `address_sender` | object | opcional | **obligatorio** | - | - | preset de la cuenta | Remitente (ver tabla) |
| `address_receiver` | object | opcional | **obligatorio** | - | - | - | Destinatario (ver tabla) |
| `boxes` | array | opcional | ver reglas | - | - | - | Bultos (ver tabla) |
| `items` | array | opcional | ver reglas | - | - | - | Artículos (ver tabla) |
| `payment` | object | opcional | **obligatorio** | - | - | - | Pago (ver tabla) |
| `declared_freight` | object | opcional | opcional | - | - | preset de la cuenta | Flete declarado: `value` (number) y `currency` (3 letras). No se verifica en la entrada; se transfiere a la cotización |
| `biller` | object | opcional | opcional | - | - | preset de la cuenta | Facturador mostrado en la invoice (misma estructura del remitente) |
| `manufacturer` | object | opcional | opcional | - | - | - | Fabricante mostrado en la invoice (ver tabla) |

> **Nota sobre monedas:** la validación del despacho no señala la falta de `currency_quote` y
> `currency_payment`, pero sin ellas (en el payload o en el preset) la generación de la cotización falla y el borrador
> pasa a `error`, que **no puede editarse**. Informa siempre las dos monedas o configura el preset.

### Parámetros — `address_sender` y `address_receiver`

Dos formas de informar el **remitente** (no las mezcles):
- **Forma A — dirección registrada:** envía solo `address_sender.address_code`. En el despacho, la dirección
  registrada se copia al borrador y se valida con las mismas reglas de abajo.
- **Forma B — campos manuales:** envía los campos de abajo y **no** envíes `address_code`.

El destinatario acepta **solo la Forma B** (`address_receiver.address_code` no se usa).

| Campo | Tipo | Mín. | Máx. (entrada) | Máx. (despacho) | Obligatorio en el despacho | Descripción |
|---|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | - | Código de dirección registrada (solo `address_sender`) |
| `country_code` | string | 2 | 2 | 2 | **Sí** | País en ISO 3166-1 alpha-2 |
| `postal_code` | string | - | 12 | 12 | Sí, si no se informa `state` | CEP/ZIP |
| `state` | string | - | - | - | Sí, si no se informan `postal_code` ni `state_code` | Nombre del estado |
| `state_code` | string | - | - | - | Ver nota | Sigla del estado, verificada contra la base de estados. Para un país sin estados, usa el propio código del país |
| `city` | string | - | 100 | 100 | **Sí** | Ciudad |
| `street` | string | - | 70 | 70 | **Sí** | Calle |
| `number` | string | - | 100 | **6** | - | Número del inmueble |
| `district` | string | - | 30 | 30 | - | Barrio |
| `complement` | string | - | 20 | 20 | - | Complemento |
| `description` | string | - | 155 | 155 | - | Etiqueta interna de la dirección |
| `name` | string | 5 | 35 | 35 | **Sí** | Nombre del contacto/razón social |
| `email` | string | - | 60 | 60 | **Sí** | E-mail válido |
| `phone` | string | - | 25 | 25 | **Sí** | Teléfono |
| `phone_extension` | string | - | 5 | 5 | - | Extensión |
| `type` | string | - | - | - | **Sí** (sin predeterminado) | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | 20 | Sí cuando el país es `BR` | Documento federal. Para `BR` sin `foreign: true`, debe ser un CPF o CNPJ válido |
| `state_tax_id` | string | - | 20 | 20 | Sí cuando el país es `BR` | Inscripción estatal |
| `foreign` | boolean | - | - | - | - | Indica persona extranjera (exime de la verificación de CPF/CNPJ) |

> **Localización de la dirección:** informa `postal_code` **o** `state`. `state_code` solo no basta,
> porque, sin `state`, el `postal_code` pasa a ser exigido.
>
> **Atención al `number`:** la entrada acepta hasta 100 caracteres, pero el despacho solo acepta hasta **6**.
> Los valores mayores dejan el borrador `incomplete`.

### Parámetros — `boxes[]`

Cuando **envías `boxes`**, cada caja debe informarse por **una** de las formas de abajo (no las mezcles
en la misma caja):

| Forma | Campos | Qué sucede |
|---|---|---|
| A — kit | `kit` | La caja y los artículos del kit se rellenan a partir del kit registrado |
| B — caja registrada por código | `box_code` | Usa la [caja registrada](api-caixa-v1.md) |
| C — caja registrada por identificador | `box_identifier` | Usa la caja registrada |
| D — caja manual | `name`, `height`, `width`, `depth` | **Los cuatro campos son obligatorios en el despacho** |

| Campo | Tipo | Mín. | Máx. | Descripción |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU del kit registrado |
| `box_code` | integer | - | - | Código de la caja registrada |
| `box_identifier` | string | - | - | Identificador de la caja registrada |
| `name` | string | 4 | 155 | Nombre de la caja |
| `height` | number | 1 | - | Alto |
| `width` | number | 1 | - | Ancho |
| `depth` | number | 1 | - | Profundidad |
| `volume` | number | 0.001 | - | Volumen. Se acepta, pero **no exime** de las dimensiones en el despacho |
| `weight` | number | 0.1 | - | Peso del bulto, hasta 3 decimales. **Obligatorio (ya en la entrada) cuando `object` es `doc`**. Ignorado en caja con `kit` |
| `price` | number | 0 | - | Valor declarado. **Obligatorio (ya en la entrada) cuando `object` es `doc`**. Ignorado en caja con `kit` |

> Con `boxes` y `object` diferente de `doc`, **toda caja que no sea kit necesita al menos un artículo**
> que apunte a ella en `items[].box_id`. Con `object: doc`, los `items` se descartan.

### Parámetros — `items[]`

Dos formas de informar cada artículo:
- **Forma A — artículo registrado:** `sku` (+ `quantity`, `unit_value` y, con `boxes`, `box_id`). Los datos
  faltantes vienen del [registro de artículos](api-itens-v1.md). El SKU debe existir.
- **Forma B — artículo manual:** todos los campos marcados como obligatorios sin `sku`.

| Campo | Tipo | Mín. | Máx. | Obligatorio en el despacho | Descripción |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU del artículo registrado |
| `ean` | string | - | 50 | - | Código EAN |
| `nsmp` | string | - | 50 | - | *Non Standard Manufacturer Product Id*: identificador de producto no estandarizado del fabricante. Se imprime en la invoice y se usa en la información regulatoria exigida por algunos transportistas para destinos en la Unión Europea. Con `sku`, si no se envía, viene del registro del artículo |
| `box_id` | integer | - | - | Sí, cuando hay `boxes` | **Posición** de la caja en el array `boxes` (1 = primera). Sin `boxes`, se ignora |
| `name` | string | 6 | 35 | Sí, sin `sku` | Nombre del artículo |
| `description` | string | 4 | 155 | - | Descripción del artículo (el mínimo de 4 se verifica en el despacho) |
| `country_code` | string | 2 | 2 | Sí, sin `sku` | País de origen del artículo |
| `hscode` | string | 6 | 10 | Sí, sin `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Sí, sin `sku` | Peso unitario (el máximo de 99 se verifica en el despacho) |
| `quantity` | integer | 1 | 1000 | **Sí** | Cantidad |
| `unit_value` | number | - | - | **Sí** | Valor unitario |
| `selling_unit_value` | number | - | - | Exigido por la cotización cuando `ior` es `true` | Valor de venta unitario |
| `height`, `width`, `depth` | number | 1 | - | Sí, cuando **no** hay `boxes` (salvo SKU registrado con dimensiones) | Dimensiones del artículo, usadas para calcular las cajas |
| `is_groupable` | boolean | - | - | - | Predeterminado `true`. Sin `boxes`: los artículos agrupables van juntos en una caja calculada; `false` genera una caja por unidad |
| `is_foldable` | boolean | - | - | - | Se acepta, pero sin efecto en el borrador |

> **Cuándo `items` es obligatorio:** sin `boxes`, `items` se exige siempre (incluso para `doc`). Con
> `boxes`, se exige para `package`, `pallet` y `not_doc`, excepto cuando las cajas usan `kit`.
>
> **Sin `boxes`:** la plataforma arma las cajas a partir de las dimensiones de los artículos. Para `doc`, envía
> siempre `boxes` con `weight` y `price`.

### Parámetros — `payment`

| Campo | Tipo | Obligatorio en el despacho | Valores | Descripción |
|---|---|---|---|---|
| `payment.type` | string | **Sí** | `billed`, `afterward` | El borrador **no acepta** `gateway` (error `422` en la entrada) |

### Parámetros — `biller`

Misma estructura y límites de `address_sender`, incluidos `address_code` (Forma A) o campos manuales
(Forma B). En el despacho, todos los campos del `biller` son opcionales; el `number` acepta hasta 6 caracteres.
Si no envías `biller` (o lo envías vacío), puede aplicarse el facturador del preset de la cuenta.

### Parámetros — `manufacturer`

Envía `address_code` **o** los campos manuales. El fabricante **no se verifica en la entrada**: los límites
de abajo se aplican en el despacho.

| Campo | Tipo | Mín. | Máx. | Obligatorio en el despacho | Descripción |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Código de dirección registrada. Cuando se usa, los demás campos son opcionales |
| `name` | string | 5 | 30 | Sí, sin `address_code` | Razón social/nombre del fabricante |
| `country_code` | string | 2 | 2 | Sí, sin `address_code` | País |
| `state_code` | string | 1 | 10 | Sí, sin `address_code` | Sigla del estado |
| `state` | string | - | 155 | - | Nombre del estado |
| `postal_code` | string | - | 12 | Sí, sin `address_code` | CEP/ZIP |
| `city` | string | - | 100 | Sí, sin `address_code` | Ciudad |
| `street` | string | - | 70 | Sí, sin `address_code` | Calle |
| `number` | string | - | 6 | - | Número |
| `district` | string | - | 30 | - | Barrio |
| `complement` | string | - | 20 | - | Complemento |
| `email` | string | - | 60 | Sí, sin `address_code` | E-mail |
| `phone` | string | - | 25 | Sí, sin `address_code` | Teléfono |
| `phone_extension` | string | - | 6 | - | Extensión |
| `type` | string | - | - | Sí, sin `address_code` | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | - | Documento federal |
| `state_tax_id` | string | - | 20 | - | Inscripción estatal |

### Payload mínimo

El menor payload aceptado es un objeto vacío: crea un borrador que quedará `incomplete` hasta que se complete.

```json
{}
```

En la práctica, envía al menos tu identificador, para poder localizar el borrador después:

```json
{
  "channel_identifier_number": "PED-2026-0001"
}
```

Payload mínimo para que el borrador **pase el despacho** sin depender de presets (remitente y destinatario
fuera de Brasil, caja manual):

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "US",
    "postal_code": "10001",
    "city": "New York",
    "street": "Fifth Avenue",
    "name": "Tienda Ejemplo",
    "email": "contato@example.com",
    "phone": "12125550100",
    "type": "pj"
  },
  "address_receiver": {
    "country_code": "CA",
    "postal_code": "M5V2T6",
    "city": "Toronto",
    "street": "King Street",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "14165550100",
    "type": "pf"
  },
  "boxes": [
    { "name": "Caja P", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Camiseta algodon",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ],
  "payment": { "type": "billed" }
}
```

### Payload máximo

Todos los campos aceptados. Los bultos muestran las formas manual (caja 1), caja registrada (caja 2)
y kit (caja 3); en una integración real usa la forma adecuada para cada bulto. El remitente usa la Forma B
(campos manuales); para la Forma A, envía solo `"address_sender": {"address_code": 1}`.

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 1,
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Borrador del pedido del marketplace",
  "declared_freight": {
    "value": 50,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centro de distribucion SP",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpon 2",
    "name": "Tienda Ejemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Oficina Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": "987654321",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Caja master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "volume": 0.008,
      "weight": 2.5,
      "price": 150
    },
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "NSMP-TSH-000-S",
      "name": "Camiseta algodon S",
      "description": "Camiseta 100% algodon manga corta blanca",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
    },
    {
      "box_id": 2,
      "name": "Bermuda sarga M",
      "description": "Bermuda de sarga beige",
      "country_code": "BR",
      "hscode": "620342",
      "weight": 0.4,
      "quantity": 1,
      "unit_value": 59.9,
      "selling_unit_value": 89.9
    }
  ],
  "payment": { "type": "billed" },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Casa matriz",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Sala 1",
    "name": "Holding Ejemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabrica Ejemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "200",
    "district": "Centro",
    "complement": "Bloque B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000144",
    "state_tax_id": "111222333444"
  }
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Caja P", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodon", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

`422 Unprocessable Entity` (ej.: `payment.type` con `gateway`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "payment.type": ["O campo payment.type não contém um valor válido."]
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success`, `warning` o `error` |
| `message` | string | Mensaje de confirmación o de error |
| `data` | object | Solo en error `422`: mapa campo → lista de mensajes |

> La respuesta de éxito **no trae el código del borrador**. Usa `GET /v1/draft` para obtenerlo.

## Endpoint: listar borradores

- **Método y ruta:** `GET /v1/draft`
- **Cuándo usarlo:** para descubrir el `codigo` de un borrador y hacer el seguimiento del `status`/`status_descricao`.
- **Alcance:** solo borradores de tu cuenta. **Ítems por página:** 20.

### Parámetros

| Campo | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `page` | query | integer | no | Página del listado (predeterminado 1) |
| `drafts` | body | object | no | Envoltorio del filtro |
| `drafts.drafts_ids` | body | array | no | Códigos de borrador (enteros o strings). **Vacío o ausente retorna todos** |

> Este `GET` acepta un cuerpo JSON para el filtro. Si tu cliente HTTP no admite body en `GET`,
> lista todos (paginando) y filtra de tu lado.

### Payload mínimo

Sin cuerpo y sin query: lista la primera página de todos los borradores.

```
GET /v1/draft
```

### Payload máximo

```
GET /v1/draft?page=1
```

```json
{
  "drafts": {
    "drafts_ids": [30, 31]
  }
}
```

### Ejemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "carrier_code": "",
          "nfkey": "",
          "object": "package",
          "type": "Commercial",
          "channel_code": "",
          "channel": "",
          "tax": "sender",
          "insurance": true,
          "ior": false,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "description": "",
          "address_sender": {
            "country_code": "",
            "state": "",
            "state_code": "",
            "postal_code": "",
            "city": "",
            "description": "",
            "street": "",
            "number": "",
            "district": "",
            "complement": "",
            "name": "",
            "email": "",
            "phone": "",
            "type": "",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "address_receiver": {
            "country_code": "US",
            "state": "",
            "state_code": "",
            "postal_code": "33101",
            "city": "Miami",
            "description": "",
            "street": "Main Avenue",
            "number": "",
            "district": "",
            "complement": "",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "items": [
            {
              "box_id": 1,
              "name": "Camiseta algodon",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9
            }
          ],
          "boxes": [
            { "name": "Caja P", "height": 20, "width": 20, "depth": 20 }
          ],
          "payment": { "type": "billed" }
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "links": [
      { "url": null, "label": "« Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/draft?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo »", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.data[].codigo` | integer | **Código del borrador.** Úsalo en `PUT /v1/draft/{code}` y en `drafts.drafts_ids` |
| `data.data[].codigo_usuario` | integer | Código de la cuenta propietaria del borrador |
| `data.data[].data` | object | Contenido del borrador (ya combinado con los presets), en formato fijo: los campos ausentes vienen como `""` |
| `data.data[].data.channel` | string | Nombre del canal de venta (cuando hay `channel_code`) |
| `data.data[].status` | string | Status actual (ver [ciclo de vida](#ciclo-de-vida-del-borrador)) |
| `data.data[].status_descricao` | string/null | **Cuando está `incomplete` o `error`, indica qué impidió el avance** (hasta 254 caracteres) |
| `data.data[].created_at`, `updated_at` | string | Fechas en el formato `AAAA-MM-DD HH:MM:SS` |
| `data.current_page`, `last_page`, `per_page`, `total`, `from`, `to` | integer | Datos de la paginación |
| `data.first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url`, `path`, `links` | string/array | Enlaces de navegación entre páginas |

> El objeto `data` del listado **no incluye** `measurement`, `residential_delivery`, `non_stackable`,
> `cost_center_code`, `declared_freight`, `biller`, `manufacturer` ni `address_sender.address_code`.
> Cuando el remitente se informó por `address_code`, después del primer despacho aparece expandido
> con los datos de la dirección registrada.
>
> Si no hay ningún borrador para los filtros (o la página solicitada está más allá de la última), la respuesta es
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`, y no una lista vacía.

## Endpoint: actualizar borrador

- **Método y ruta:** `PUT /v1/draft/{code}`
- **Cuándo usarlo:** para corregir un borrador `incomplete`.
- **Restricción:** solo borradores de tu cuenta con status `incomplete`.
- **Cuerpo:** mismos campos y reglas del `POST`. **El contenido anterior se sustituye por completo**
  (ver [El PUT sustituye el borrador entero](#el-put-sustituye-el-borrador-entero)).
- **Efecto:** el status pasa a `edited` y el borrador vuelve al próximo ciclo de despacho.
  El `status_descricao` anterior permanece hasta la nueva validación.

### Parámetros

| Parámetro | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `code` | path | integer | **sí** | Código del borrador (`codigo` en el listado) |
| cuerpo | body | object | no | Misma estructura del [`POST`](#endpoint-crear-borrador) |

### Payload mínimo

`PUT /v1/draft/30` con cuerpo `{}`. Se acepta, pero **borra** el contenido del borrador (quedan solo los presets).
Úsalo solo si quieres reiniciar el borrador.

```json
{}
```

### Payload máximo

El mismo [payload máximo del `POST`](#payload-máximo). En la práctica, reenvía el borrador completo con las correcciones.

### Ejemplo de request

Borrador que estaba `incomplete` por falta del remitente: el cuerpo trae el contenido original **más** el remitente.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "number": "100", "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Caja P", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodon", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso.",
  "data": {
    "codigo": 30,
    "codigo_usuario": 1,
    "data": {
      "channel_identifier_number": "PED-2026-0001",
      "object": "package",
      "type": "Commercial",
      "tax": "sender",
      "insurance": true,
      "measurement": "metric",
      "currency_quote": "USD",
      "currency_payment": "BRL",
      "address_sender": {
        "country_code": "BR",
        "postal_code": "09941070",
        "city": "Diadema",
        "street": "Rua Principal",
        "number": "100",
        "name": "Tienda Ejemplo LTDA",
        "email": "contato@example.com",
        "phone": "11989677465",
        "type": "pj",
        "federal_tax_id": "12345678000199",
        "state_tax_id": "123456789012"
      },
      "address_receiver": {
        "country_code": "US",
        "postal_code": "33101",
        "city": "Miami",
        "street": "Main Avenue",
        "name": "John Doe",
        "email": "johndoe@example.com",
        "phone": "11992494597",
        "type": "pf"
      },
      "boxes": [
        { "name": "Caja P", "height": 20, "width": 20, "depth": 20 }
      ],
      "items": [
        {
          "box_id": 1,
          "name": "Camiseta algodon",
          "country_code": "BR",
          "hscode": "610910",
          "weight": 0.25,
          "quantity": 3,
          "unit_value": 29.9
        }
      ],
      "payment": { "type": "billed" }
    },
    "status": "edited",
    "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
    "created_at": "2026-02-20 19:04:00",
    "updated_at": "2026-02-20 19:20:31"
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.codigo` | integer | Código del borrador |
| `data.codigo_usuario` | integer | Código de la cuenta propietaria del borrador |
| `data.data` | object | Contenido guardado: el payload enviado ya combinado con los presets. Puede contener campos adicionales provenientes de las predefiniciones de la cuenta; ignora los que no conozcas |
| `data.status` | string | Siempre `edited` después de la actualización |
| `data.status_descricao` | string/null | Motivo de la validación anterior (se actualizará en el próximo despacho) |
| `data.created_at`, `data.updated_at` | string | Fechas en el formato `AAAA-MM-DD HH:MM:SS` |

## Endpoint: eliminar borradores

- **Método y ruta:** `DELETE /v1/draft`
- **Cuándo usarlo:** para descartar borradores que no se usarán, incluidos los que quedaron en `error`.
- **Alcance:** solo se eliminan los códigos que pertenecen a tu cuenta; los demás se ignoran.
  La eliminación es definitiva.

### Parámetros

| Campo | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `drafts` | body | object | **sí** | Envoltorio |
| `drafts.drafts_ids` | body | array | **sí** (no vacío) | Códigos de borrador a eliminar (enteros o strings) |

### Payload mínimo

```json
{ "drafts": { "drafts_ids": [30] } }
```

### Payload máximo

```json
{ "drafts": { "drafts_ids": [30, 31, 32] } }
```

### Ejemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` cuando al menos un borrador de la lista fue eliminado |
| `message` | string | Mensaje de confirmación |

> Basta con que **un** código de la lista pertenezca a tu cuenta para que la respuesta sea `success`; los demás se
> ignoran sin aviso. Si ningún código corresponde, la respuesta es
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Reglas de negocio

### Validación en el despacho
En el despacho, la plataforma aplica un conjunto **estricto** de reglas. Lo que falta aparece en
`status_descricao` y el borrador queda `incomplete`.

| Grupo | Exigencias |
|---|---|
| General | `tax` e `insurance` presentes (en el payload o vía preset); `channel_identifier_number` texto de hasta 100 caracteres |
| Remitente y destinatario | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`postal_code` o `state`); `number` con hasta 6 caracteres |
| Direcciones en Brasil (`BR`) | `federal_tax_id` y `state_tax_id`; `federal_tax_id` debe ser un CPF o CNPJ válido, salvo `foreign: true` |
| Sin `boxes` | `items` obligatorio (cualquier `object`), con dimensiones para calcular las cajas |
| Con `boxes` | Cada caja: `kit`, `box_code`, `box_identifier` o (`name` + `height` + `width` + `depth`). Para `object` ≠ `doc`, cada caja (excepto kit) necesita al menos un artículo con su `box_id` |
| `object` = `doc` | `boxes[].weight` y `boxes[].price` (verificados ya en la entrada) |
| Artículos | `quantity` (1 a 1000) y `unit_value`; sin `sku`: `name`, `country_code`, `hscode`, `weight` (hasta 99). Con `sku`, el SKU debe estar registrado |
| Pago | `payment.type` (`billed` o `afterward`) |
| Fabricante | Con `manufacturer` sin `address_code`: `name`, `country_code`, `state_code`, `postal_code`, `city`, `street`, `email`, `phone` y `type` |
| Canal | `channel_code`, cuando se informa (o viene del preset), debe pertenecer a tu cuenta |

### Generación de la cotización y del envío
Una vez validado, el borrador se convierte automáticamente en una cotización y un envío:

| Etapa | Comportamiento |
|---|---|
| Kits | Cada `boxes[].kit` se sustituye por la caja y por los artículos del kit registrado |
| Cajas | Con `boxes`, usa las cajas enviadas. Sin `boxes`, calcula las cajas por las dimensiones de los artículos (agrupables juntos; no agrupables, una caja por unidad) |
| Cotización | Tipo simple para `doc` y avanzado para los demás. Usa `declared_freight` del payload o del preset |
| Elección del contrato | Si `carrier_code` está entre los resultados, se usa. Si no, la plataforma elige según la preferencia de la cuenta: el **más barato** o el **más rápido** |
| Envío | Un `type` vacío se convierte en `Commercial`; `nfkey`, `cost_center_code` y `biller` se transfieren cuando se informan |
| Rechazo por regla de negocio | El borrador vuelve a `incomplete`, con el mensaje de la cotización/envío en `status_descricao` |
| Fallo inesperado | El borrador pasa a `error`. Elimínalo y crea uno nuevo |

### Mensajes posibles en `status_descricao`
Además de los mensajes de validación de campos (ej.: `O campo address sender.country code é obrigatório.`),
pueden aparecer:

| Mensaje | Causa | Cómo corregirlo |
|---|---|---|
| `Nenhum item foi enviado para este rascunho` | Sin `items` (y sin kits) | Enviar `items` |
| `O item com o sku :sku não foi encontrado!` | SKU no registrado y artículo sin datos completos | Registrar el artículo o enviar los campos manuales |
| `Nenhuma caixa foi enviado para este rascunho` | `boxes` vacío, o `box_code`/`box_identifier` no encontrado | Verificar las cajas |
| `Parâmetros da caixa não foram prenchidos corretamente por favor certifique-se dos seguintes campos (name,height,width,depth)` | Caja manual sin `name`, `height`, `width` o `depth` | Completar los cuatro campos |
| `Nenhum item foi associado para a caixa :name !` | Caja sin artículo con su `box_id` | Ajustar `items[].box_id` |
| `Não há itens válidos, verifique os dados e/ou o codigo SKU` | Sin `boxes`, ningún artículo utilizable | Verificar los artículos y SKUs |
| `O canal especificado não foi encontrado!` | `channel_code` no pertenece a tu cuenta | Corregir el `channel_code` |
| `O kit :sku é inválido!` | Kit inexistente o inactivo | Verificar el SKU del kit |
| `A caixa do kit :sku é inválida!` | No se encontró la caja del kit | Revisar el registro del kit |
| `Um ou mais itens do kit :sku são inválidos!` | Artículo del kit no encontrado | Revisar el registro del kit |
| Mensajes de la cotización o del envío | Ej.: sin contratos disponibles, restricciones de carga | Ver [Cotización](api-cotacao-v1.md) y [Envío](api-remessa-v1.md) |

## Errores comunes

| Status | Significado / Mensaje | Cómo corregirlo |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | Edita solo borradores `incomplete`; consulta el status en el `GET` |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Verificar el `code`/`drafts_ids` (puede ser de otra cuenta o ya haber sido convertido en envío y eliminado) |
| `403` | `Has no permission!` | Solicitar al soporte el permiso de la acción (crear, editar, listar o eliminar borradores) |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + `data` con los campos | Corregir los campos listados en `data` |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` (en `data.cost_center_code`) | Usar un centro de costo de tu cuenta o de cuentas autorizadas |
| `422` | `O campo drafts é obrigatório.` / `O campo drafts.drafts ids é obrigatório.` (en el `DELETE`) | Enviar `drafts.drafts_ids` con al menos un código |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Repetir con backoff; si persiste, contactar al soporte |

Los errores de autenticación (token inválido, cuenta bloqueada) están en
[Primeros Pasos](api-primeiros-passos.md#errores-de-autenticación).

## Observaciones finales
- El campo **`status_descricao` es tu guía**: indica qué impidió que el borrador avanzara.
- Informa siempre `channel_identifier_number`: es lo que vincula el borrador a tu pedido, permite encontrar el
  `codigo` en el listado y consultar el envío después de que el borrador sea eliminado.
- En el `PUT`, **reenvía el borrador completo**: el contenido anterior se sustituye.
- Si la cuenta tiene preset de impuesto, el `tax` enviado se ignora; verifica el valor guardado en el `GET`.
- El borrador **no acepta** pago vía `gateway`. Para ese escenario usa
  [cotización](api-cotacao-v1.md) + [envío](api-remessa-v1.md).
- Para el flujo opuesto (todo validado de una vez, documentos inmediatos), consulta [Manifiesto](api-manifesto.md).
- Registros usados por el borrador: [Caja](api-caixa-v1.md), [Artículos](api-itens-v1.md), [Contratos](api-contrato-v1.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
