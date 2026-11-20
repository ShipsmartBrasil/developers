# Manifiesto

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Ejecuta **cotización + envío + despacho al transportista + generación de la etiqueta en una única llamada sincrónica**.
- La respuesta ya trae el identificador del envío, el número de seguimiento y la etiqueta (AWB) en PDF y/o ZPL.
- Es el flujo indicado para operaciones que necesitan la etiqueta **inmediatamente**, sin esperar
  procesamiento asíncrono.
- Solo considera contratos de transportistas con despacho sincrónico soportado (actualmente **FedEx, UPS y DHL**).
- Solo acepta **pago facturado** (`payment.type: billed`), y la cuenta debe tener esa forma de pago habilitada.

> ⚠️ **El manifiesto es sincrónico e irreversible.** Como cotiza y despacha en la misma solicitud, no hay
> etapa intermedia de revisión. Valida bien el payload antes de enviarlo.

## URL base
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
Content-Type: application/json
Accept: application/json
lang: pt
```

> El header `lang` acepta `pt`, `en` y `es` (predeterminado `pt`). El perfil del token debe tener el permiso de
> **crear manifiesto**; sin él, la respuesta es `403` con `Has no permission!`.

## Flujo de uso
1. Arma el payload completo (direcciones, cajas, artículos, pago). Revisa la sección
   [Dirección manual: envía todas las claves](#dirección-manual-envía-todas-las-claves).
2. `POST /v1/manifest?format=pdf` (o `zpl`, o sin `format` para recibir los dos).
3. Imprime la etiqueta recibida en `data.docs` (usa el `base64` o descárgala por la `url`).
4. Guarda el `data.shipping` y úsalo en [seguimiento](api-rastreio.md), [documentos](api-documentos.md) y
   [programación de recogida](api-agendamento.md).
5. Unos instantes después, consulta [Documentos del Envío](api-documentos.md) para obtener los documentos
   complementarios (invoice, packing list, etc.), generados en segundo plano.

## Conceptos esenciales

### Manifiesto vs. cotización + envío

| Tema | Cotización + Envío | Manifiesto |
|---|---|---|
| Llamadas | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Elección del transportista | Tú eliges en la lista retornada | La plataforma elige (o tú lo indicas con `carrier_code`/`slug`) |
| Transportistas | Todos los contratos de la cuenta | **Solo FedEx, UPS y DHL** |
| Etiqueta (AWB) | Consulta posterior en `/docs` | **Viene en la respuesta** |
| Pago | `billed`, `gateway`, `afterward` | **Solo `billed`** |
| Tipo de respuesta | Asíncrono | **Sincrónico** |
| Revisión antes del despacho | Sí | No |
| Campos obligatorios | Menos (divididos en 2 etapas) | **Más** (todo de una vez) |

Usa el manifiesto cuando: la operación es de alto volumen, el transportista ya está definido por una regla de
negocio y necesitas imprimir la etiqueta en el momento.

### Cómo se elige el transportista

| Lo que envías | Comportamiento |
|---|---|
| Ni `slug` ni `carrier_code` | La plataforma elige según la preferencia configurada en la cuenta: **más rápido** (menor plazo, predeterminado) o **más barato** (menor valor en la moneda de pago) |
| `carrier_code` | Es una **preferencia**. Si el contrato aparece en el resultado de la cotización, se usa. Si no aparece: contrato inexistente o de transportista compatible con el manifiesto → la plataforma recurre a la preferencia de la cuenta; contrato existente de transportista **no** compatible con el manifiesto → `404` |
| `slug` | **Fija** el contrato: la cotización evalúa solo ese contrato y el `slug` **sobrescribe** cualquier `carrier_code` enviado. Si el contrato no está entre los contratos elegibles de tu cuenta para ese envío, la cotización vuelve a evaluar todos y la elección sigue la regla del `carrier_code` |

> No mezcles `slug` y `carrier_code`: usa uno u otro. Si envías los dos, prevalece el `slug`.
> Cuando la cuenta tiene un contrato preferido predefinido y no envías `carrier_code`, la predefinición se usa como `carrier_code`.

### Predefiniciones de la cuenta en el manifiesto

El manifiesto aplica las [predefiniciones de la cuenta](api-primeiros-passos.md#predefiniciones-de-la-cuenta-presets)
**después** de la validación de entrada. Por eso, todos los campos obligatorios deben estar en el payload,
aunque la cuenta tenga predefinición para ellos.

| Predefinición | Cuándo se aplica en el manifiesto |
|---|---|
| Responsable del impuesto (`tax`) | **Siempre que esté configurada: sustituye el `tax` enviado** (consulta la tabla siguiente) |
| Centro de costo (`cost_center_code`) | Cuando el payload no envía el campo o lo envía vacío (`null`, `""` o `0`) |
| Canal de venta (`channel_code`) | Cuando no se envía la clave `channel_code`. Enviar `"channel_code": null` desactiva la predefinición |
| Contrato preferido (`carrier_code`) | Cuando no se envía la clave `carrier_code` |
| Flete declarado (`declared_freight`) | Cuando no se envía la clave `declared_freight`. Enviar `declared_freight` (incluso `null` o `{}`) descarta la predefinición |
| Facturador (`biller`) | Cuando la visualización del facturador está habilitada en la cuenta y el payload no envía `biller`, o lo envía vacío (`null`, `{}` o solo con valores vacíos). Un `biller` completado en el payload siempre prevalece |
| Demás predefiniciones (`insurance`, `measurement`, `type`, monedas, entrega residencial, no apilable, dirección del remitente) | **No tienen efecto**: esos campos son obligatorios en el manifiesto y el valor del payload siempre prevalece |

**Resolución del responsable del impuesto (`tax`):**

| Configuración de la cuenta | `tax` efectivamente usado |
|---|---|
| Sin predefinición de impuesto | El valor enviado en el payload |
| Predeterminado `sender` o `receiver` | El predeterminado de la cuenta, **ignorando** el valor enviado |
| Predeterminado + país de destino en la lista de excepciones de la cuenta | El **opuesto** del predeterminado (`sender` ↔ `receiver`) |
| Modo bloqueado | Siempre `sender`, independientemente del payload y de las excepciones |

> El país de destino usado en las excepciones es el `address_receiver.country_code` del payload. Si el destinatario
> se informa solo por `address_code`, las excepciones por país no se evalúan y vale el predeterminado de la cuenta.
> Incluso cuando la predefinición sustituye el valor, `tax` sigue siendo obligatorio en el payload y debe ser `sender` o `receiver`.

### Etapas del procesamiento y qué ocurre en caso de falla

| Etapa | Si falla |
|---|---|
| 1. Validación de entrada | `422`. No se crea nada |
| 2. Reglas de la cuenta y complementación (pago, canal, `slug`, direcciones, cajas, artículos y kits registrados, estado) | `400`/`404`. No se crea nada |
| 3. Cotización, registro del envío y despacho al transportista | Todo se deshace: ninguna cotización o envío queda registrado en la cuenta |
| 4. Generación de la etiqueta y de los documentos (después de que el transportista acepta el envío) | El envío **permanece creado**, con número de seguimiento, y recibe un evento de error de procesamiento en el seguimiento. Recupera los documentos en [Documentos del Envío](api-documentos.md) y no repitas el manifiesto |

## Endpoint: crear manifiesto

- **Método y ruta:** `POST /v1/manifest`
- **Cuándo usar:** para crear, en una única llamada, un envío ya despachado al transportista y
  recibir la etiqueta en la respuesta.

### Parámetros — query

| Nombre | Lugar | Tipo | Obligatorio | Predeterminado | Valores | Descripción |
|---|---|---|---|---|---|---|
| `format` | query | string | no | vacío (todos) | `pdf`, `zpl` | Formato de la etiqueta en la respuesta. Vacío (o valor no reconocido) retorna PDF **y** ZPL |

> `format` también puede enviarse **en el cuerpo** del payload. Cuando se envía en los dos lugares,
> **prevalece el valor del cuerpo**. En el cuerpo, los valores diferentes de `pdf`/`zpl` se rechazan con `422`.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Valores / Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `object` | string | **sí** | - | - | `doc`, `package`, `pallet`, `not_doc` | Lo que se está enviando. `not_doc` se acepta por compatibilidad y se trata como `package` |
| `type` | string | **sí** | - | - | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalidad del envío (valor exacto, con mayúsculas y espacios) |
| `tax` | string | **sí** | - | - | `sender`, `receiver` | Quién paga los impuestos. Puede ser sustituido por la predefinición de la cuenta (consulta [Predefiniciones](#predefiniciones-de-la-cuenta-en-el-manifiesto)) |
| `insurance` | boolean | **sí** | - | - | - | Contratar seguro. Puede activarse automáticamente si la cuenta tiene contratación automática de seguro y el valor declarado alcanza el mínimo configurado |
| `residential_delivery` | boolean | **sí** | - | - | - | Entrega en dirección residencial |
| `non_stackable` | boolean | **sí** | - | - | - | Bultos no apilables |
| `measurement` | string | **sí** | - | - | `metric`, `imperial` | Sistema de medida de pesos y dimensiones |
| `ior` | boolean | **sí** | - | - | - | Importer of Record |
| `currency_quote` | string | **sí** | 3 | 3 | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda de la cotización (y de los valores de los artículos) |
| `currency_payment` | string | **sí** | 3 | 3 | ver [monedas](api-primeiros-passos.md#monedas-aceptadas) | Moneda del pago |
| `address_sender` | object | **sí** | - | - | - | Remitente (ver tabla) |
| `address_receiver` | object | **sí** | - | - | - | Destinatario (ver tabla) |
| `boxes` | array | **sí** | 1 elemento | - | - | Bultos (ver tabla) |
| `payment` | object | **sí** | - | - | - | Pago (ver tabla) |
| `items` | array | condicional | - | - | - | Obligatorio cuando `object` **no** es `doc` y ninguna caja usa `kit`. **Prohibido (la clave no puede existir) cuando `object` es `doc`** |
| `carrier_code` | integer | no | - | - | - | Contrato preferido (consulta [Cómo se elige el transportista](#cómo-se-elige-el-transportista)) |
| `slug` | string | no | - | 20 | - | Identificador textual que fija el contrato. Debe existir; un contrato eliminado o fuera de vigencia retorna `400` |
| `channel_code` | integer | no | - | - | - | Código del canal de venta registrado **en tu cuenta** |
| `channel_identifier_number` | string | no | - | - | - | Tu identificador comercial (ej.: número del pedido). **Recomendado** |
| `cost_center_code` | integer | no | - | - | Predefinición de la cuenta | Centro de costo registrado para tu cuenta o para cuentas asociadas autorizadas. Vacío/ausente → usa la predefinición de la cuenta |
| `nfkey` | string | no | 44 | 44 | - | Clave de la NF-e (exactamente 44 caracteres) |
| `description` | string | no | 5 | 155 | - | Descripción libre del manifiesto |
| `declared_freight` | object | no | - | - | Predefinición de la cuenta | Flete declarado (ver tabla) |
| `biller` | object | no | - | - | Predefinición de la cuenta | Facturador mostrado en la invoice (ver sección) |
| `manufacturer` | object | no | - | - | - | Fabricante mostrado en la invoice (ver sección) |
| `extra` | object | no | - | - | - | Datos auxiliares en pares clave/valor (ver tabla) |
| `format` | string | no | - | - | `pdf`, `zpl` | Alternativa a la query string (prevalece sobre ella) |
| `size` | string | no | - | - | `label`, `letter` | Aceptado en la validación, pero **actualmente no altera** la etiqueta generada |

### Parámetros — `address_sender` y `address_receiver`

A diferencia del envío v1/v2, aquí la dirección se informa **completa**, porque la cotización y el despacho
ocurren en la misma llamada. Hay dos formas de informar cada dirección — **no mezcles las dos en el mismo objeto**.

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de una dirección registrada y visible para tu cuenta |

> Con `address_code`, **todos** los datos de la dirección provienen del registro; cualquier otro campo enviado en el mismo
> objeto se descarta. Si envías `address_code` junto con `country_code`, `street`, `name`, `email`,
> `phone`, `type` y `foreign`, la solicitud es rechazada con `422`. El remitente y el destinatario no pueden usar el
> mismo `address_code`.

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `country_code` | string | **sí** | 2 | 2 | - | País en ISO 3166-1 alpha-2 |
| `postal_code` | string | condicional | - | 12 | - | CEP/ZIP. Obligatorio si no se informan `state` ni `state_code`. En Brasil, se usa para descubrir el estado cuando no se envía `state_code` |
| `state` | string | condicional | - | 100 | - | Nombre del estado. Obligatorio si no se informan `state_code` ni `postal_code` |
| `state_code` | string | condicional | 1 | 10 | - | Sigla del estado. Obligatoria si no se informan `state` ni `postal_code`. **Recomendada siempre** (consulta las reglas de negocio) |
| `city` | string | condicional | 1 | 100 | - | Ciudad. Obligatoria si no se informa `postal_code`. **Recomendada siempre**: los transportistas usan la ciudad en la etiqueta |
| `street` | string | **sí** | 5 | 70 | - | Calle |
| `number` | string | no | 1 | 6 | - | Número del inmueble |
| `district` | string | no | 1 | 30 | - | Barrio |
| `complement` | string | no | 1 | 20 | - | Complemento |
| `name` | string | **sí** | 1 | 70 | - | Nombre del contacto/razón social |
| `email` | string | **sí** | - | 60 | - | Correo electrónico válido |
| `phone` | string | no | 1 | 25 | - | Teléfono. **Recomendado siempre**: los transportistas exigen un teléfono de contacto |
| `phone_extension` | string | no | - | 5 | - | Extensión |
| `type` | string | **sí** | - | - | - | `pf` (persona física) o `pj` (persona jurídica) |
| `foreign` | boolean | **sí** | - | - | - | Indica si la persona es extranjera (usado para validar el `federal_tax_id`) |
| `federal_tax_id` | string | condicional | - | 20 | - | Documento federal (CPF/CNPJ o equivalente). **Obligatorio cuando `country_code` es `BR` o `AR`** |
| `state_tax_id` | string | no | - | 20 | - | Inscripción estatal |
| `description` | string | no | 1 | 155 | - | Etiqueta interna de la dirección |
| `company_name` | string | no | 5 | 50 | - | **Solo en `address_sender`.** Razón social usada en envíos FedEx (el transportista considera solo los primeros 35 caracteres) |

#### Dirección manual: envía todas las claves

En el manifiesto, las claves siguientes deben **existir** en el objeto de dirección manual, aunque no tengas
valor para ellas — en ese caso, envía `null`. La ausencia de alguna de ellas interrumpe el procesamiento con error
(`500`, o `400` en la etapa del transportista).

| Claves que deben estar presentes (el valor puede ser `null`) |
|---|
| `postal_code`, `city`, `number`, `district`, `complement`, `phone`, `phone_extension`, `federal_tax_id`, `state_tax_id` |

> Tampoco envíes `"address_code": null` en la forma B: la clave `address_code` debe omitirse.

### Parámetros — `boxes[]`

Cada caja puede informarse de tres formas — **no mezcles las formas en la misma caja**:

| Forma | Campos | Comportamiento |
|---|---|---|
| A — caja registrada | `box_code` **o** `box_identifier` | Nombre, dimensiones y peso provienen del [registro de cajas](api-caixa-v1.md) |
| B — caja manual | `name`, `height`, `width`, `depth` (+ `weight`/`price` en `doc`) | Datos informados en el payload |
| C — kit | `kit` | La caja **y** los artículos de esa posición provienen del kit registrado |

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `kit` | string | no | - | 50 | SKU del kit registrado. Completa la caja y los artículos de esa posición; `weight` y `price` de la caja se ignoran |
| `box_code` | integer | condicional | - | - | Código de la caja registrada. Obligatorio cuando la caja no informa ningún otro campo |
| `box_identifier` | string | condicional | - | 50 | Identificador de la caja registrada. Alternativa al `box_code` |
| `name` | string | **sí** sin `kit`/`box_code`/`box_identifier` | 5 | 155 | Nombre de la caja |
| `height` | number | **sí** sin `kit`/`box_code`/`box_identifier` | 1 | - | Alto (cm en `metric`, in en `imperial`) |
| `width` | number | **sí** sin `kit`/`box_code`/`box_identifier` | 1 | - | Ancho |
| `depth` | number | **sí** sin `kit`/`box_code`/`box_identifier` | 1 | - | Profundidad |
| `weight` | number | **sí** cuando `object` es `doc` (incluso con caja registrada) | 0.001 | - | Peso del bulto (kg en `metric`, lb en `imperial`) |
| `price` | number | **sí** cuando `object` es `doc` (incluso con caja registrada) | 0 | - | Valor declarado del bulto. En `doc`, el valor declarado del envío es la suma de los `price` |

> Con caja registrada, los campos enviados en el payload complementan (y sobrescriben) los del registro.
> La caja debe pertenecer a tu cuenta.

### Parámetros — `items[]`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `box_id` | integer | **sí** | - | - | **Posición** de la caja en el array `boxes` (1 = primera) |
| `quantity` | integer | **sí** | 1 | 99 | Cantidad del artículo |
| `sku` | string | condicional | - | 50 | SKU del artículo registrado ([Artículos](api-itens-v1.md)). Obligatorio cuando no se informa ninguno de `name`, `country_code`, `hscode`, `weight` y `unit_value` |
| `name` | string | **sí** sin `sku` | 6 | 50 | Nombre del artículo |
| `country_code` | string | **sí** sin `sku` | 2 | 2 | País de origen del artículo (ISO 3166-1 alpha-2) |
| `hscode` | string | **sí** sin `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **sí** sin `sku` | 0.001 | 99 | Peso unitario (kg en `metric`, lb en `imperial`) |
| `unit_value` | number | **sí** sin `sku` | mayor que 0 | - | Valor unitario, en la moneda `currency_quote`. La validación acepta `0`, pero la cotización rechaza un artículo con valor cero |
| `selling_unit_value` | number | no | 0 | - | Valor de venta unitario, usado cuando `ior` es `true`. Si se omite, proviene del registro del artículo (con `sku`) o, a falta de este, asume el `unit_value` |
| `description` | string | no | 5 | 450 | Descripción detallada del artículo |
| `ean` | string | no | - | 50 | Código EAN (código de barras) del artículo. Usado como identificador estandarizado del fabricante en la información regulatoria para destinos en la Unión Europea (y como identificador del producto cuando no hay `sku`) |
| `nsmp` | string | no | - | 50 | *Non Standard Manufacturer Product Id*: identificador de producto no estandarizado del fabricante. Se imprime en la invoice y se usa en la información regulatoria exigida por algunos transportistas para destinos en la Unión Europea |

> **Artículos con `sku`:** cuando el artículo informa `sku` y falta alguno de los campos `name`, `country_code`, `hscode`,
> `weight`, `unit_value`, `nsmp` (o `selling_unit_value` con `ior: true`), los datos ausentes se completan
> a partir del [registro del artículo](api-itens-v1.md) (incluidos `ean` y `nsmp`). Los valores enviados en el payload
> siempre prevalecen sobre el registro. Si el SKU no está registrado, el artículo solo se acepta si trae
> `name`, `country_code`, `hscode` y `weight` en el propio payload.
>
> La información regulatoria para destinos en la Unión Europea solo se arma cuando el artículo tiene `sku` o `ean`.

> **Atención a las diferencias con respecto a la cotización:** aquí `quantity` tiene un máximo de **99** y
> `description` acepta hasta **450** caracteres.

### Parámetros — `payment`

| Campo | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|
| `payment` | object | **sí** | - | Envoltorio |
| `payment.type` | string | **sí** | **`billed`** | El manifiesto acepta **solo** pago facturado. La cuenta también debe tener el pago facturado habilitado; de lo contrario, la respuesta es `400` |

### Parámetros — `declared_freight`

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `value` | number | no | 0 | - | Valor del flete declarado. Se acepta `0` |
| `currency` | string | **sí** cuando se informa `value` | 3 | 3 | Moneda del flete declarado |
| `declared` | boolean | no | - | - | Aceptado, pero **sin efecto** en el manifiesto: el flete declarado solo se considera cuando se informan `value` **y** `currency` |

> Para declarar flete cero, envía `"value": 0` con la `currency`. Sin `value` y `currency`, el flete declarado
> no se aplica (ni el de la predefinición, si enviaste la clave `declared_freight`).

### Parámetros — `extra`

Pares clave/valor auxiliares, grabados en el envío.

| Regla | Detalle |
|---|---|
| Estructura | Objeto **plano** (un nivel, sin objetos ni listas anidados) |
| Claves | Solo letras minúsculas, dígitos y `_` |
| Valores | Texto, entero o decimal (booleano y `null` son rechazados) |

| Clave conocida | Tipo | Regla | Descripción |
|---|---|---|---|
| `federal_tax_provision` | number | - | Provisión de impuesto federal |
| `state_tax_provision` | number | - | Provisión de impuesto estatal |
| `usd_rate_provision` | number | mayor que 0 | Provisión de tipo de cambio del dólar. **Solo se mantiene** para cuentas del programa Remessa Conforme con destino `BR`; en los demás casos se descarta silenciosamente |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parámetros — `biller` y `manufacturer`

- `biller`: mismo formato descrito en [Envío (v1)](api-remessa-v1.md#parámetros--biller-facturador)
  (`address_code` **o** campos manuales). En el manifiesto, `biller.name` tiene de 5 a **355** caracteres.
  Los campos no enviados asumen `type: pj` y `foreign: false`. Sin `biller` en el payload, vale la
  predefinición de facturador de la cuenta (cuando está habilitada).
- `manufacturer`: mismo formato descrito en [Cotización (v1)](api-cotacao-v1.md#parámetros--manufacturer)
  (`address_code` **o** campos manuales). Con `address_code` de una dirección existente, los datos provienen del registro.

### Payload mínimo

Manifiesto de documento (`object: doc`) con direcciones manuales — `items` está prohibido en este caso:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Calle Principal",
    "number": null,
    "district": null,
    "complement": null,
    "name": "Tienda Ejemplo LTDA",
    "email": "contato@example.com",
    "phone": "11999990000",
    "phone_extension": null,
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": null,
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "number": null,
    "district": null,
    "complement": null,
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550000",
    "phone_extension": null,
    "type": "pf",
    "federal_tax_id": null,
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Sobre documentos",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

**Variación mínima con direcciones registradas y kit** (`object: package`; `items` puede omitirse porque la caja usa `kit`):

```json
{
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": { "address_code": 1234 },
  "address_receiver": { "address_code": 5678 },
  "boxes": [
    { "kit": "KIT-001" }
  ],
  "payment": { "type": "billed" }
}
```

### Payload máximo

Todos los campos aceptados. `slug` y `carrier_code` aparecen juntos solo para ilustrar: en la práctica usa uno
u otro (prevalece el `slug`). Las cajas muestran la forma manual y la forma registrada.

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 12,
  "carrier_code": 45,
  "slug": "fedex-br",
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
  "description": "Manifiesto del lote de marzo",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centro de distribucion SP",
    "street": "Calle Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpon 2",
    "company_name": "Tienda Ejemplo Comercio LTDA",
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
    "phone": "13055550000",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Caja master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 1.5,
      "price": 90
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7891234567895",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodon P",
      "description": "Camiseta 100% algodon, manga corta, color blanco, talla P",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Zapatilla deportiva",
      "description": "Zapatilla de running masculina numero 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Casa matriz",
    "street": "Calle Principal",
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
    "street": "Calle Principal",
    "number": "200",
    "district": "Centro",
    "complement": "Bloque B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000181",
    "state_tax_id": "111222333444"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": false,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Calle Principal", "number": null, "district": null, "complement": null, "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "phone": "11999990000", "phone_extension": null, "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": null, "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "number": null, "district": null, "complement": null, "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550000", "phone_extension": null, "type": "pf", "federal_tax_id": null, "state_tax_id": null, "foreign": true},
    "boxes": [{"name": "Sobre documentos", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response

`200 OK` (sin `format`, la respuesta trae `AWB` y `AWB_ZPL`; con `format=pdf`, solo `AWB`; con `format=zpl`, solo `AWB_ZPL`):

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "271249639538",
    "tracking_packages_ids": [
      "271249639538"
    ],
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Presencia | Descripción |
|---|---|---|---|
| `data.shipping` | string (UUID) | siempre | **Identificador del envío.** Úsalo en seguimiento, documentos y programación de recogida |
| `data.carrier` | string | siempre | Nombre del contrato/transportista que procesó el envío |
| `data.tracking_id` | string | siempre | Número de seguimiento principal (AWB) |
| `data.tracking_packages_ids` | array de string | cuando el transportista lo informa | Número de seguimiento de cada bulto, en el orden de las cajas |
| `data.docs` | object | siempre | Etiquetas generadas, en el formato `nombre → { url, base64 }` |
| `data.docs.AWB` | object | `format` vacío o `pdf` | Etiqueta en PDF |
| `data.docs.AWB_ZPL` | object | `format` vacío o `zpl` | Etiqueta en ZPL (impresoras térmicas) |
| `data.docs.AWBPRC` / `data.docs.AWBPRCZPL` | object | cuentas del programa Remessa Conforme, en importaciones (origen fuera de Brasil y destino `BR`) | Etiqueta específica del programa, en PDF / ZPL |
| `data.docs.*.url` | string | siempre | Enlace del archivo. Si la cuenta protege los enlaces de documentos, el enlace es **temporal** (válido por unos 5 minutos) |
| `data.docs.*.base64` | string | siempre | Contenido del archivo en Base64 |
| `data.valorProvCBS`, `data.valorProvIBSEstadual`, `data.valorProvIBSMunicipal` | number | cuando el cálculo de impuestos trae esas provisiones | Provisiones de CBS e IBS (estatal y municipal) |
| `data.serviceType` | string | envíos FedEx | Servicio FedEx utilizado |
| `data.operationalDetail` | object | envíos FedEx | Datos operativos del transportista (`ursaPrefixCode`, `ursaSuffixCode`, `originLocationId`, `originLocationNumber`, `originServiceArea`, `destinationLocationId`, `destinationLocationNumber`, `destinationServiceArea`, `destinationLocationStateOrProvinceCode`, `deliveryDate`, `deliveryDay`, `commitDate`, `commitDay`, `ineligibleForMoneyBackGuarantee`, `astraPlannedServiceLevel`, `astraDescription`, `postalCode`, `stateOrProvinceCode`, `countryCode`, `airportId`, `serviceCode`, `packagingCode`, `publishedDeliveryTime`, `scac`). Los campos sin valor vienen `null` |

> Los documentos complementarios — invoice (envíos que no son `doc`), declaración de contenido (cuando el
> remitente es `pf`), packing list y declaración de productos no peligrosos — se generan en segundo plano.
> Consulta [Documentos del Envío](api-documentos.md) unos instantes después para obtener la lista completa.

### Respuesta de error del transportista

Cuando el transportista rechaza el envío, la respuesta es `400` con `status: "error"` y trae su respuesta bruta:

```json
{
  "status": "error",
  "message": "[1234] Fedex - Teste - Mensaje de error del transportista",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Campo | Descripción |
|---|---|
| `message` | `[código del contrato] nombre del transportista - mensaje` retornado por el transportista. Si el transportista responde sin un mensaje reconocible, viene `CARRIER SYNC - SHIPPING RETURN UNEXPECTED RESPONSE!` |
| `data` | Payload bruto del transportista. **La estructura varía según el transportista** y puede venir `null` |

> Registra `message` y `data` en tus logs: es la única fuente del motivo técnico del rechazo. Un rechazo del
> transportista deshace la cotización y el envío; corrige el payload y envía un nuevo manifiesto.

## Reglas de negocio
- **Pago:** solo `billed`, y la cuenta debe tener el pago facturado habilitado (solo o junto
  con el pago inmediato). Otro valor en `payment.type` → `422`; cuenta sin facturación → `400`.
- **Transportistas:** el manifiesto cotiza solo contratos FedEx, UPS y DHL de tu cuenta. La elección sigue
  [Cómo se elige el transportista](#cómo-se-elige-el-transportista).
- **Predefiniciones:** aplicadas después de la validación, según
  [Predefiniciones de la cuenta en el manifiesto](#predefiniciones-de-la-cuenta-en-el-manifiesto). Destacados: el `tax` de la
  predefinición **sustituye** el enviado; el `cost_center_code` de la predefinición completa el campo cuando viene vacío.
- **`items` y `object: doc`:** con `object: doc`, la clave `items` **no puede** enviarse (ni vacía).
  Con cualquier otro `object`, `items` es obligatorio — excepto cuando alguna caja usa `kit`.
- **Kits:** cuando una caja informa `kit`, la caja y los artículos de esa posición provienen del registro,
  y `boxes[].weight`/`boxes[].price` se ignoran.
- **Valor declarado:** en `doc`, es la suma de `boxes[].price`; en los demás, la suma de `unit_value × quantity`
  de los artículos. Un artículo con valor unitario vacío o cero es rechazado en la cotización (`400`).
- **`imperial`:** los pesos y dimensiones informados en libras/pulgadas se convierten a kg/cm antes de la cotización.
- **Estado (`state_code`):** debe ser una sigla de estado reconocida por la plataforma; para países sin
  división en estados, envía el propio `country_code`. Sin `state_code`, la plataforma intenta descubrirlo
  (por el CEP en Brasil, por el nombre en `state` o por el código postal en el exterior). Si no lo consigue para el
  remitente **o** para el destinatario, el manifiesto es rechazado con `400` antes de cualquier registro.
- **`federal_tax_id`:** obligatorio cuando el país de la dirección es `BR` o `AR`.
- **`address_code`:** la dirección debe estar visible para tu cuenta; con él, los demás campos del
  objeto se ignoran. El remitente y el destinatario no pueden tener el mismo `address_code`.
- **Dirección manual:** envía todas las claves listadas en
  [Dirección manual: envía todas las claves](#dirección-manual-envía-todas-las-claves) (usa `null` cuando no haya valor).
- **`channel_code`:** debe existir y pertenecer a tu cuenta (incluso cuando proviene de la predefinición).
- **`cost_center_code`:** debe existir y pertenecer a tu cuenta, a la cuenta principal o a cuentas
  asociadas autorizadas.
- **`slug`:** debe existir (`422`) y el contrato debe estar activo y dentro de la vigencia (`400`).
- **Seguro automático:** si la cuenta contrata seguro automáticamente a partir de un valor mínimo y el valor
  declarado (convertido a USD) alcanza ese mínimo, el seguro se contrata incluso con `insurance: false`.
- **`extra.usd_rate_provision`:** se mantiene solo para cuentas del programa Remessa Conforme con destino `BR`.
- **`not_doc`** se acepta por compatibilidad y se trata como `package`.

## Errores comunes

Los errores de validación (`422`) tienen `status: "warning"`, `message` igual a
`Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!`
y `data` con el mapa `campo → mensajes`. Los errores de regla de negocio tienen `status: "warning"` y no traen `data`.

| Status | Significado / Mensaje | Cómo corregir |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificar el token enviado en `Authorization` |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contactar al soporte |
| `403` | `Has no permission!` | Solicitar al soporte el permiso de crear manifiesto |
| `422` | Validación de campos (mensajes en `data`, ej.: `O campo :attribute é obrigatório.`) | Corregir los campos listados en `data` |
| `422` | `The :attribute field must be missing when :other is :value.` (en `items`) | Eliminar la clave `items` cuando `object` es `doc` |
| `422` | `The :attribute field must be missing when :values are present.` (en `address_code`) | Usar solo `address_code` **o** solo los campos manuales |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Usar un `cost_center_code` de tu cuenta |
| `400` | `Tipo de pagamento não autorizado!` | Solicitar al soporte la habilitación del pago facturado |
| `400` | `O canal especificado não foi encontrado!` | Usar un `channel_code` registrado en tu cuenta (revisa también la predefinición) |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | Usar un `slug` de contrato activo |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Usar `address_code` diferentes |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | Verificar el `address_code` |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Corregir el `postal_code` o enviar `state_code` |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Corregir `state` o enviar `state_code`/`postal_code` |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Enviar `state_code` en las dos direcciones |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | Verificar `box_code`/`box_identifier` |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | Registrar el artículo o enviar los campos manuales |
| `400` | `O kit :sku é inválido!` / `A caixa do kit :sku é inválida!` / `Um ou mais itens do kit :sku são inválidos!` | Revisar el registro del kit |
| `400` | Mensaje de la cotización (ej.: `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. ...`, `Nenhum transportador encontrado para este usuário!`) | En el manifiesto, todo error de la etapa de cotización vuelve como `400`. Leer `message` y ajustar el payload |
| `400` | Mensaje del registro del envío | Leer `message` y ajustar el payload |
| `404` | `A transportadora informada não está disponível para esta remessa!` | Usar un `carrier_code` de un contrato FedEx, UPS o DHL, u omitir el campo |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | El contrato elegido no tiene despacho sincrónico; contactar al soporte |
| `400` | `[código] transportador - mensagem` (`status: "error"`, `data` con la respuesta del transportista) | Ver [Respuesta de error del transportista](#respuesta-de-error-del-transportista) |
| `500` | Error inesperado (`status: "error"`) | **Antes de repetir**, verifica en [Seguimiento](api-rastreio.md) si el envío fue creado |

> ⚠️ **Nunca repitas un manifiesto automáticamente después de un `500` o de una falla de red/timeout.** Si la falla
> ocurre después de que el transportista aceptó el envío, el envío existe de verdad y una repetición genera un
> envío duplicado. Verifica primero por el `channel_identifier_number` + `channel_code` en [Seguimiento](api-rastreio.md).

## Observaciones finales
- Informa siempre `channel_identifier_number` (con `channel_code`): es tu protección contra duplicidad y la
  forma de localizar el envío si la respuesta se pierde.
- El manifiesto concentra mucha responsabilidad en una llamada. Para integraciones que pueden esperar,
  el flujo de [cotización](api-cotacao-v1.md) + [envío](api-remessa-v1.md) es más seguro.
- Para el flujo opuesto (payload parcial, validado después), consulta [Borrador](api-rascunho.md).
- Registros usados por el manifiesto: [Caja](api-caixa-v1.md) y [Artículos](api-itens-v1.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
