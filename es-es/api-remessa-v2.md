# Envío (v2)

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md) y [Cotización (v2)](api-cotacao-v2.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Transforma una **cotización v2** en un envío real.
- La diferencia central respecto al [Envío v1](api-remessa-v1.md): aquí informas la **dirección completa**
  (ciudad, estado y código postal), porque la cotización v2 recibió solo el país (y quizás el estado).
- La dirección completa se **valida en el envío** (CEP en Brasil, dirección internacional en los demás países).
- La v2 expone **solo la creación**. Status, documentos, cancelación, actualización y programación de recogida
  usan los endpoints `/v1/...`, que funcionan normalmente para envíos creados en la v2.

> ⚠️ **El envío es una operación irreversible en la ruta normal.** Una vez creado, entra en procesamiento
> en el transportista.

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

## Flujo de uso

```
1. POST /v2/quotation                  → quotation + carriers[]   (solo país/estado)
2. POST /v2/shipping                   → envío recibido            (dirección completa)
3. GET  /v1/shipping/{shipping}/status → seguir hasta "shipped"
4. GET  /v1/shipping/{shipping}/docs   → descargar AWB, invoice, etc.
5. POST /v1/schedule                   → programar recogida (opcional)
```

1. [Crea la cotización v2](api-cotacao-v2.md) y verifica si es [elegible](#cotizaciones-no-elegibles-para-envío).
2. Envía el envío con el `quotation`, el transportista elegido y la dirección completa.
3. La creación es **asíncrona**: `200` significa "recibimos", no "despachamos".
4. Haz el seguimiento por el [rastreo](api-rastreio.md) y descarga los [documentos](api-documentos.md).

## Conceptos esenciales

### Envío a partir de una cotización v2
- El **país** de origen y destino viene siempre de la cotización (no se informa en el envío).
- El **estado** informado en la cotización (`state_code`) se reaprovecha; si la cotización no tuvo estado, se
  descubre a partir del código postal/dirección enviados en el envío.
- **Ciudad y código postal** se informan aquí y pasan a valer para el envío.
- Una cotización genera **un único** envío.

### Mapa de endpoints por versión

| Operación | Ruta |
|---|---|
| Crear envío | `POST /v2/shipping` |
| Consultar status | `GET /v1/shipping/{shipping}/status` |
| Consultar status por canal | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` |
| Consultar documentos | `GET /v1/shipping/{shipping}/docs` |
| Consultar documentos por canal | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` |
| Actualizar (NF-e, adjuntos, centro de costo) | `PUT /v1/shipping/{shipping}` |
| Eliminar adjuntos | `DELETE /v1/shipping/{shipping}/docs` |
| Cancelar | `DELETE /v1/shipping/{shipping}` |
| Cancelar por canal | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` |
| Programar recogida | `POST /v1/schedule` |

> No existe `GET /v2/shipping/...` ni `DELETE /v2/shipping/...`. Usa las rutas `/v1`, documentadas en
> [Envío (v1)](api-remessa-v1.md).

## Endpoint: crear envío

- **Método y ruta:** `POST /v2/shipping`
- **Cuándo usarlo:** después de obtener una cotización **v2** elegible, dentro del plazo de validez.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `quotation` | string | **sí** | - | - | - | UUID de la cotización. Debe haber sido creada en la **v2** |
| `carrier_code` | integer | **sí** si no se envía `slug` | - | - | - | Código del contrato (`carriers[].code` de la cotización) |
| `slug` | string | **sí** si no se envía `carrier_code` | - | 20 | - | Identificador textual del contrato (`carriers[].slug` de la cotización). Cuando se envía, prevalece sobre `carrier_code` |
| `address_sender` | object | **sí** | - | - | - | Remitente (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | Destinatario (ver tabla abajo) |
| `payment` | object | **sí** | - | - | - | Forma de pago |
| `type` | string | no | - | - | preset de la cuenta o `Commercial` | Finalidad del envío. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | condicional | 44 | 44 | `null` | Clave de la NF-e. **Exactamente 44 caracteres**. Obligatoria en algunos contratos |
| `invoice_number` | string | no | 1 | 50 | código interno | Número de la factura/invoice |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Referencia del envío |
| `cost_center_code` | integer | no | - | - | preset de la cuenta | Código del centro de costo habilitado para tu usuario. Vacío/nulo recibe el predeterminado de la cuenta |
| `payment_notify_shipping_billing` | boolean | no | - | - | preset de la cuenta | Con `payment.type: afterward`, envía el e-mail de cobro |
| `attachments` | array | no | - | - | `null` | Documentos adjuntos al envío |
| `biller` | object | no | - | - | preset de la cuenta | Facturador mostrado en la invoice en lugar del remitente |

### Parámetros — `address_sender` y `address_receiver`

**Esta es la principal diferencia de la v2:** ciudad y código postal (o estado) son obligatorios aquí.

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de la dirección registrada. Cuando se usa, no envíes los demás campos |

> ⚠️ En la v2, usa `address_code` en el envío **solo si el mismo `address_code` se usó en la cotización v2**.
> Una cotización hecha solo con país no tiene ciudad, y la dirección registrada será rechazada con
> `406 A cidade :city é diferente do enviado na cotação!`.

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `city` | string | **sí** | 3 | 100 | - | Ciudad |
| `postal_code` | string | condicional | - | 12 | - | CEP/ZIP. Obligatorio si no se informa `state`. **Recomendado siempre; en Brasil es lo que permite validar la dirección** |
| `state` | string | condicional | - | - | - | Nombre del estado/provincia. Obligatorio si no se informa `postal_code` |
| `state_code` | string | no | - | - | estado de la cotización | Sigla del estado/provincia. Cuando la cotización tuvo `state_code`, se usa en lugar de este |
| `street` | string | **sí** | - | 70 | - | Calle |
| `name` | string | **sí** | 5 | 35 | - | Nombre del contacto/razón social |
| `email` | string | **sí** | - | 60 | - | E-mail válido (recibe notificaciones del transportista) |
| `phone` | string | **sí** | - | 25 | - | Teléfono |
| `type` | string | **sí** | - | - | - | `pf` (persona física) o `pj` (persona jurídica) |
| `number` | string | no | - | 6 | `null` | Número del inmueble |
| `district` | string | no | - | 30 | `null` | Barrio |
| `complement` | string | no | - | 20 | `null` | Complemento |
| `description` | string | no | - | 155 | `null` | Etiqueta interna de la dirección |
| `phone_extension` | string | no | - | 5 | `null` | Extensión |
| `federal_tax_id` | string | condicional | - | 20 | `null` | CPF/CNPJ o documento federal. Obligatorio según el país (ver [Documentos fiscales](api-remessa-v1.md#documentos-fiscales-obligatorios)) |
| `state_tax_id` | string | condicional | - | 20 | `null` | Inscripción estatal. Obligatoria para brasileños (`BR` con `foreign: false`) |
| `foreign` | boolean | no | - | - | `false` si el país es `BR`; `true` en los demás | Indica si la persona es extranjera |

> **Regla práctica:** informa `city` + `postal_code` + `street` + `name` + `email` + `phone` + `type`
> (+ documentos fiscales cuando se exijan). El país viene de la cotización.
> Sin `postal_code`, el campo `state` (nombre del estado) pasa a ser obligatorio — `state_code` solo no basta.

### Parámetros — `payment`

| Campo | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|
| `payment.type` | string | **sí** | `billed`, `gateway`, `afterward` | Modelo financiero del envío |

| Valor | Significado | Requisito de la cuenta |
|---|---|---|
| `billed` | Facturado: el valor entra en tu factura periódica | Opción de pago "facturado" habilitada |
| `gateway` | Pago inmediato vía gateway, en la moneda definida en `currency_payment` | Opción de pago "inmediato" habilitada |
| `afterward` | Pago posterior, acordado fuera del flujo automático | - |

### Parámetros — `attachments[]`

| Campo | Tipo | Obligatorio | Máx. | Descripción |
|---|---|---|---|---|
| `description` | string | **sí** | 100 | Nombre/descripción del documento (usado después para localizar o eliminar el adjunto) |
| `base64` | string | **sí** | - | Contenido del archivo codificado en Base64 |

| Límite | Valor |
|---|---|
| Formatos aceptados | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamaño total de la solicitud | **90 MB** |

> Como en la v1, los adjuntos en formato no reconocido se **descartan en silencio** en la creación. Para adjuntar
> con validación, usa `PUT /v1/shipping/{shipping}` (límite de 5 MB por archivo).

### Parámetros — `biller` (facturador)

Cuando se informa, esta dirección aparece en la invoice en lugar de la dirección del remitente.
Envía `address_code` **o** los campos manuales. **Si no vas a usar facturador, no envíes el campo.**

| Campo | Tipo | Mín. | Máx. | Descripción |
|---|---|---|---|---|
| `address_code` | integer | - | - | Código de dirección registrada |
| `country_code` | string | 2 | 2 | País en ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nombre del estado |
| `state_code` | string | - | 10 | Sigla del estado |
| `postal_code` | string | - | 12 | CEP/ZIP |
| `city` | string | 3 | 100 | Ciudad |
| `street` | string | - | 70 | Calle |
| `number` | string | - | 6 | Número |
| `district` | string | - | 30 | Barrio |
| `complement` | string | - | 20 | Complemento |
| `description` | string | - | 155 | Etiqueta interna |
| `name` | string | 5 | 35 | Nombre/razón social |
| `email` | string | - | 60 | E-mail |
| `phone` | string | - | 25 | Teléfono |
| `phone_extension` | string | - | 5 | Extensión |
| `type` | string | - | - | `pf` o `pj` |
| `federal_tax_id` | string | - | 20 | Documento federal |
| `state_tax_id` | string | - | 20 | Inscripción estatal |
| `foreign` | boolean | - | - | Indica si es extranjero |

### Payload mínimo

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "name": "Tienda Ejemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

### Payload máximo

```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "slug": "dhl-br",
  "type": "Commercial",
  "nfkey": "12345678901234567890123456789012345678901234",
  "invoice_number": "NF-000123",
  "shipment_reference": "REF-2026-0001",
  "cost_center_code": 1,
  "payment_notify_shipping_billing": true,
  "address_sender": {
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
    "phone": "13055550100",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "foreign": true
  },
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
  "payment": {
    "type": "afterward"
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcKJcTl8uXrp..."
    },
    {
      "description": "XML de la Nota Fiscal",
      "base64": "PD94bWwgdmVyc2lvbj0i..."
    }
  ]
}
```

### Ejemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` cuando el envío fue registrado |
| `message` | string | Confirmación de recepción. El identificador del envío es el propio `quotation` |

## Diferencias respecto a la v1

| Tema | v1 | v2 |
|---|---|---|
| Dirección en el envío | Calle, nombre, e‑mail, teléfono, tipo (+ docs fiscales) | Los mismos campos **+ `city`, `postal_code` (o `state`)** |
| `city` | Ignorado (viene de la cotización) | **Obligatorio**, mín. 3 y máx. 100 caracteres |
| Validación de dirección | Hecha en la cotización | **Hecha en el envío** |
| `address_code` | Debe coincidir con la dirección de la cotización | Solo funciona si la cotización v2 también usó `address_code` |
| `biller` | Campos de localidad aceptados | Campos de localidad **validados** (`city` mín. 3) |
| Endpoints del ciclo | Creación, status, documentos, actualización, adjuntos, cancelación | **Solo creación.** El resto usa `/v1` |
| Origen de la cotización | `POST /v1/quotation` | `POST /v2/quotation` |

## Cotizaciones no elegibles para envío

Valen las mismas verificaciones de la v1, en el mismo orden — consulta la tabla completa en
[Envío (v1)](api-remessa-v1.md#cotizaciones-no-elegibles-para-envío). La única diferencia es la versión:

| Verificación | Status | Mensaje |
|---|---|---|
| `quotation` existe y fue creado en la **v2** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| Cotización simple de paquete/pallet | `403` | `Cotação Simples é inelegível para remessa!` |
| Cotización ya usada | `406` | `Cotação já foi utilizada em outra remessa!` |
| Cotización expirada | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| Contrato fuera de la lista de la cotización | `404` | `Contrato especificado não foi listado para esta cotação!` |

## Reglas de negocio

### Validación de la dirección completa
Después de las verificaciones de la cotización, se valida la dirección manual (Forma B):

| Caso | Status | Mensaje |
|---|---|---|
| Brasil, sin estado en la cotización: CEP inexistente o de otro estado | `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| Fuera de Brasil: dirección no localizada | `404` | `Endereço internacional enviado está errado ou não existe!` |
| Fuera de Brasil: estado/provincia no reconocido | `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` |

> Envía siempre `postal_code` y, cuando lo sepas, `state_code` (preferiblemente ya en la cotización): es lo que da la mayor
> tasa de acierto en la validación.

### Coherencia entre cotización y envío
- El país es siempre el de la cotización.
- El `state_code` de la cotización, cuando existe, prevalece sobre el enviado en el envío.
- Con `address_code`, la dirección registrada debe tener el mismo país, estado y ciudad registrados en la cotización:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documentos fiscales, transportista y predefiniciones
Idénticos a la v1:
[Documentos fiscales obligatorios](api-remessa-v1.md#documentos-fiscales-obligatorios),
[Restricciones de transportista](api-remessa-v1.md#restricciones-de-transportista) y
[Predefiniciones aplicadas en el envío](api-remessa-v1.md#predefiniciones-aplicadas-en-el-envío).

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Documento fiscal, CEP brasileño inválido, NF-e obligatoria | Leer `message` y corregir el escenario |
| `401` | Pago no autorizado | Habilitar el tipo de pago en la cuenta |
| `403` | Cotización simple/expirada, o sin permiso | Rehacer la cotización / solicitar el permiso al soporte |
| `404` | Contrato fuera de la cotización, dirección internacional no localizada | Verificar `carrier_code`/`slug` y la dirección |
| `406` | Cotización ya usada o `address_code` incompatible | Hacer una nueva cotización / usar dirección manual |
| `422` | Validación de campos, cotización de otra versión, cuenta solo DDP | Corregir los campos listados en `data` |
| `500` | Error inesperado o estado/provincia no reconocido | Revisar la dirección; repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Usa la v2 cuando la dirección completa solo exista en el momento del envío.
- Todo el ciclo operativo posterior a la creación se realiza por los endpoints `/v1` — no distinguen la versión
  en que se creó el envío.
- Para el detalle de cada operación posterior a la creación, consulta [Envío (v1)](api-remessa-v1.md),
  [Rastreo](api-rastreio.md), [Documentos](api-documentos.md) y [Programación](api-agendamento.md).
- Diferencias entre las versiones en [Cotización & Envío: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
