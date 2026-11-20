# Envío (v1)

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md) y [Cotización (v1)](api-cotacao-v1.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Transforma una **cotización v1** en un envío real: genera AWB, invoice, declaración y los demás documentos.
- Es el módulo más completo del ciclo: creación, consulta de status, consulta de documentos, actualización,
  eliminación de adjuntos y cancelación.
- Los endpoints de status, documentos, actualización y cancelación atienden **también** envíos creados en el
  [Envío V2](api-remessa-v2.md).
- La creación es **asíncrona**: la API registra el envío y el despacho en el transportista ocurre a continuación.

> ⚠️ **El envío es una operación irreversible en la ruta normal.** Una vez creado, entra en procesamiento
> en el transportista. Las únicas modificaciones posteriores admitidas son excepcionales: vincular la NF-e cuando
> no se informó, adjuntar documentos que no se enviaron en la creación e informar/cambiar el centro de costo.

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
1. POST /v1/quotation                  → quotation + carriers[]
2. POST /v1/shipping                   → envío recibido (asíncrono)
3. GET  /v1/shipping/{shipping}/status → seguir hasta "shipped"
4. GET  /v1/shipping/{shipping}/docs   → descargar AWB, invoice, etc.
5. POST /v1/schedule                   → programar recogida (opcional)
```

1. [Crea la cotización v1](api-cotacao-v1.md) y verifica si es [elegible](#cotizaciones-no-elegibles-para-envío).
2. Envía el envío con el `quotation` y el transportista elegido (`carrier_code` **o** `slug`).
3. La creación es **asíncrona**: la respuesta `200` significa "recibimos", no "despachamos".
4. Consulta el [status](api-rastreio.md) hasta que aparezca en `shipped`.
5. Descarga los [documentos](api-documentos.md).
6. Si es necesario, [programa la recogida](api-agendamento.md).

## Conceptos esenciales

### Envío a partir de una cotización v1
- La cotización v1 ya trajo **país, ciudad, estado y código postal** de origen y destino. Por eso, en el envío v1
  informas solo **quién** envía y **quién** recibe (nombre, contacto, calle, documentos fiscales).
- Los campos de localidad (`city`, `state`, `state_code`, `postal_code`, `country_code`) enviados en
  `address_sender`/`address_receiver` se **ignoran** en la v1: la localidad es siempre la de la cotización.
- Una cotización genera **un único** envío.

### El identificador `{shipping}`

En los endpoints de documentos, actualización, eliminación de adjuntos y cancelación, el parámetro de path `{shipping}`
acepta **tres formatos**:

| Formato | Ejemplo | Observación |
|---|---|---|
| UUID de la cotización | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forma recomendada. Cualquier valor con guion se trata como UUID |
| Clave de la NF-e | `12345678901234567890123456789012345678901234` | Exactamente 44 dígitos numéricos |
| Número del AWB | `271249639538` | Envíos creados en los **últimos 90 días** |

> El endpoint de **status** acepta solo UUID o AWB. Detalles en [Rastreo de Envío](api-rastreio.md).

Para consultar sin guardar ninguno de esos valores, usa los endpoints **por canal**, basados en el
`channel_identifier_number` que informaste en la cotización.

## Endpoint: crear envío

- **Método y ruta:** `POST /v1/shipping`
- **Cuándo usarlo:** después de obtener una cotización v1 elegible, dentro del plazo de validez.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `quotation` | string | **sí** | - | - | - | UUID de la cotización. Debe haber sido creada en la **v1** |
| `carrier_code` | integer | **sí** si no se envía `slug` | - | - | - | Código del contrato (`carriers[].code` de la cotización) |
| `slug` | string | **sí** si no se envía `carrier_code` | - | 20 | - | Identificador textual del contrato (`carriers[].slug` de la cotización). Cuando se envía, prevalece sobre `carrier_code` |
| `address_sender` | object | **sí** | - | - | - | Remitente (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | Destinatario (ver tabla abajo) |
| `payment` | object | **sí** | - | - | - | Forma de pago (ver tabla abajo) |
| `type` | string | no | - | - | preset de la cuenta o `Commercial` | Finalidad del envío. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | condicional | 44 | 44 | `null` | Clave de la NF-e. Cuando se informa, debe tener **exactamente 44 caracteres**. Obligatoria en algunos contratos (ver [Reglas de negocio](#restricciones-de-transportista)) |
| `invoice_number` | string | no | 1 | 50 | código interno | Número de la factura/invoice. Si se omite, usa el código interno del envío |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Referencia del envío. Si se omite, usa el ID interno del envío |
| `cost_center_code` | integer | no | - | - | preset de la cuenta | Código del centro de costo. Debe existir y estar habilitado para tu usuario. Vacío/nulo recibe el centro de costo predeterminado de la cuenta |
| `payment_notify_shipping_billing` | boolean | no | - | - | preset de la cuenta | Con `payment.type: afterward`, envía el e-mail de cobro del envío |
| `attachments` | array | no | - | - | `null` | Documentos adjuntos al envío (ver tabla abajo) |
| `biller` | object | no | - | - | preset de la cuenta | Facturador mostrado en la invoice en lugar del remitente (ver tabla abajo) |

> **`carrier_code` o `slug`:** envía **uno** de los dos. El contrato informado debe estar en la lista
> `carriers[]` retornada por la cotización; de lo contrario, la API responde `404` con
> `Contrato especificado não foi listado para esta cotação!`.

### Parámetros — `address_sender` y `address_receiver`

Dos formas, no las mezcles:

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de la dirección registrada. **Debe tener el mismo país, estado y ciudad de la cotización** |

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
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
| `federal_tax_id` | string | condicional | - | 20 | `null` | CPF/CNPJ o documento federal. Obligatorio según el país (ver [Documentos fiscales](#documentos-fiscales-obligatorios)) |
| `state_tax_id` | string | condicional | - | 20 | `null` | Inscripción estatal. Obligatoria para brasileños (`BR` con `foreign: false`) |
| `foreign` | boolean | no | - | - | `false` si el país es `BR`; `true` en los demás | Indica si la persona es extranjera. Define cómo se valida el `federal_tax_id` en Brasil |

> El país, el estado, la ciudad y el código postal **no** se informan aquí: vienen de la cotización.
> Si necesitas informarlos en el envío, usa el [Envío V2](api-remessa-v2.md).

### Parámetros — `payment`

| Campo | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|
| `payment.type` | string | **sí** | `billed`, `gateway`, `afterward` | Modelo financiero del envío |

| Valor | Significado | Requisito de la cuenta |
|---|---|---|
| `billed` | Facturado: el valor entra en tu factura periódica | Opción de pago "facturado" habilitada |
| `gateway` | Pago inmediato vía gateway, en la moneda definida en `currency_payment` | Opción de pago "inmediato" habilitada |
| `afterward` | Pago posterior, acordado fuera del flujo automático | - |

> Un valor fuera de la lista retorna `422`. Un tipo no habilitado para tu cuenta retorna
> `401` con `Opção de pagamento não autorizado!`.

### Parámetros — `attachments[]`

| Campo | Tipo | Obligatorio | Máx. | Descripción |
|---|---|---|---|---|
| `description` | string | **sí** | 100 | Nombre/descripción del documento (usado después para localizar o eliminar el adjunto) |
| `base64` | string | **sí** | - | Contenido del archivo codificado en Base64 |

| Límite | Valor |
|---|---|
| Formatos aceptados | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamaño total de la solicitud | **90 MB** |

> ⚠️ **En la creación, los adjuntos inválidos se descartan en silencio:** un archivo en formato no reconocido o mayor
> que el límite no genera error — el envío se crea sin él. Verifica los adjuntos guardados en
> [Documentos](api-documentos.md). Para adjuntar después, con validación y mensajes de error, usa
> [actualizar envío](#endpoint-actualizar-envío) (límite de **5 MB** por archivo).

### Parámetros — `biller` (facturador)

Cuando se informa, esta dirección aparece en la invoice en lugar de la dirección del remitente.
Envía `address_code` **o** los campos manuales. **Si no vas a usar facturador, no envíes el campo** (o envíalo vacío:
en ese caso vale el facturador preconfigurado en la cuenta, si lo hay).

| Campo | Tipo | Mín. | Máx. | Descripción |
|---|---|---|---|---|
| `address_code` | integer | - | - | Código de dirección registrada |
| `country_code` | string | 2 | 2 | País en ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nombre del estado |
| `state_code` | string | - | 10 | Sigla del estado |
| `postal_code` | string | - | 12 | CEP/ZIP |
| `city` | string | - | 100 | Ciudad |
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
    "street": "Rua Principal",
    "name": "Tienda Ejemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012"
  },
  "address_receiver": {
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

> `federal_tax_id` (CNPJ válido) y `state_tax_id` aparecen en el payload mínimo porque el remitente es brasileño
> persona jurídica. Para un remitente de otro país (excepto `MX` y `GR`), esos campos pueden omitirse.

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
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Rua Principal", "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550100", "type": "pf"},
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
| `message` | string | Confirmación de recepción |

> La respuesta **no** trae AWB ni documentos: el envío solo fue recibido. Usa el
> [rastreo](api-rastreio.md) para hacer el seguimiento y los [documentos](api-documentos.md) para descargar los archivos.
> El identificador del envío es el propio `quotation`.

### Errores específicos de la creación

| Status | Mensaje | Causa / corrección |
|---|---|---|
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Documento federal ausente donde es obligatorio (ver [Documentos fiscales](#documentos-fiscales-obligatorios)) |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brasileño sin `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ con dígito verificador incorrecto |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF con dígito verificador incorrecto |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` en Brasil con `state_tax_id` exento |
| `400` | `Numero da nota fiscal é requerido` | El contrato exige `nfkey` para este escenario |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` de contrato eliminado/inactivo |
| `401` | `Opção de pagamento não autorizado!` | Tipo de pago no habilitado en tu cuenta |
| `403` | `Cotação Simples é inelegível para remessa!` | Usa `type: advanced` o `items` en la cotización |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` superado — rehaz la cotización |
| `403` | `Has no permission!` | Tu perfil no tiene permiso para crear envíos |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | `quotation` de otra cuenta |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fuera de la lista de la cotización |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` inválido o de otra cuenta |
| `406` | `Cotação já foi utilizada em outra remessa!` | Una cotización genera **un** envío |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` con país diferente del cotizado |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` con estado diferente del cotizado |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` con ciudad diferente de la cotizada |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inexistente o creado en la v2 (viene en `data.quotation`) |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente)...` | Cuenta restringida a DDP con cotización DDU no pagada |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` no habilitado para ti |

> Cuando hay más de un error de documento fiscal, los mensajes vienen juntos en `message`, separados por ` | `.

## Endpoint: consultar status del envío

- **Método y ruta:** `GET /v1/shipping/{shipping}/status`
- **Por canal:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

### Parámetros

| Parámetro | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `shipping` | path | string | **sí** | UUID de la cotización o AWB |
| `channel_identifier_number` | path | string | **sí** (variante por canal) | Identificador comercial informado en la cotización |
| `channel_code` | path | integer | no | Código del canal de venta |

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

> Formato de la respuesta, grupos de status y códigos en [Rastreo de Envío](api-rastreio.md).

## Endpoint: consultar documentos

- **Método y ruta:** `GET /v1/shipping/{shipping}/docs`
- **Por canal:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

### Parámetros

| Parámetro | Ubicación | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | UUID, clave NF-e o AWB |
| `attachments` | query | boolean | no | `true` | Incluir los adjuntos enviados por ti (exige `base64=true`) |
| `embedded` | query | boolean | no | `true` | Incluir documentos incorporados (requiere permiso específico y `base64=true`) |
| `base64` | query | boolean | no | `true` | Incluir el contenido en Base64 además de la `url` |
| `documents` | query | string | no | todos | Lista separada por comas con los tipos deseados |

Tipos aceptados en `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

### Ejemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

> Formato de la respuesta y detalles de cada documento en [Documentos de Envío](api-documentos.md).

## Endpoint: actualizar envío

- **Método y ruta:** `PUT /v1/shipping/{shipping}`
- **Cuándo usarlo:** solo en casos excepcionales — vincular la NF-e, adjuntar documentos faltantes
  o informar/cambiar/eliminar el centro de costo.
- **Regla:** envía `nfkey` y/o `attachments` y/o `cost_center_code`. **Al menos uno** debe estar presente.

### Parámetros

| Campo | Ubicación | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | - | UUID, clave NF-e o AWB |
| `nfkey` | body | string | condicional | 44 | 44 | Clave de la NF-e. **Solo se guarda si el envío aún no tiene NF-e**; si ya la tiene, se ignora |
| `cost_center_code` | body | integer | condicional | - | - | Código del centro de costo habilitado para tu usuario. Enviar `null` **elimina** el centro de costo |
| `attachments` | body | array | condicional | - | - | Archivos a adjuntar |
| `attachments[].description` | body | string | **sí** si se envía `attachments` | - | 100 | Descripción del documento |
| `attachments[].base64` | body | string | **sí** si se envía `attachments` | - | - | Contenido en Base64. Formatos: `PDF`, `XML`, `JPEG`, `PNG`. Máx. **5 MB** por archivo |

### Payload mínimo

```json
{
  "cost_center_code": 1
}
```

### Payload máximo

```json
{
  "nfkey": "12345678901234567890123456789012345678901234",
  "cost_center_code": 1,
  "attachments": [
    { "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
    { "description": "Foto del producto", "base64": "iVBORw0KGgoAAAANSUhEUg..." }
  ]
}
```

### Ejemplo de request

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` cuando la actualización fue aplicada |
| `message` | string | Confirmación |

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | No se envió ninguno de los tres campos |
| `400` | `Status da remessa não permite atualização na NFE.` | El envío ya pasó de `shipped` (o está cancelado/con error) |
| `400` | `Erro ao salvar arquivo!` | `base64` inválido |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Adjunto superior a 5 MB |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` inválido o de otra cuenta |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Envío cancelado o con ciclo finalizado |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Formato del adjunto no aceptado |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML mal formado |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Centro de costo no habilitado |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Error inesperado |

> Las modificaciones se aplican en el orden NF-e → centro de costo → adjuntos. Si un adjunto falla, la NF-e y el centro
> de costo enviados en la misma llamada **ya fueron guardados**.

## Endpoint: eliminar adjuntos

- **Método y ruta:** `DELETE /v1/shipping/{shipping}/docs`
- **Cuándo usarlo:** para borrar un adjunto enviado por error.

### Parámetros

| Campo | Ubicación | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | - | UUID, clave NF-e o AWB |
| `attachments` | body | array | **sí** | 1 ítem | - | Lista de los adjuntos a eliminar |
| `attachments[].id` | body | integer | **sí** si no se envía `name` | 1 | - | ID del adjunto (retornado en `GET .../docs`) |
| `attachments[].name` | body | string | **sí** si no se envía `id` | - | 100 | Descripción exacta del adjunto |

> Envía **`id` O `name`** por ítem, nunca los dos.

### Payload mínimo

```json
{
  "attachments": [
    { "id": 123456 }
  ]
}
```

### Payload máximo

```json
{
  "attachments": [
    { "id": 123456 },
    { "name": "Nota Fiscal" }
  ]
}
```

### Ejemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!",
  "data": {
    "deleted": [
      { "id": 123456, "name": "Nota Fiscal" }
    ],
    "failed": [
      { "id": 789, "name": null, "reason": "Nenhum documento encontrado!" }
    ]
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `data.deleted[]` | array | Adjuntos eliminados (`id`, `name`) |
| `data.failed[]` | array | Adjuntos no eliminados, con el motivo en `reason` |

> La operación es **parcial**: lo que se eliminó aparece en `deleted` y lo que falló en `failed`,
> con el motivo. Verifica siempre el array `failed`.

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Remessa não encontrada!` | UUID sin envío |
| `404` | `Esta remessa não foi encontrada!` | AWB/NF-e no localizado o de otra cuenta |
| `422` | Validación | `attachments` vacío, o ítem sin `id`/`name` |

## Endpoint: cancelar envío

- **Método y ruta:** `DELETE /v1/shipping/{shipping}`
- **Por canal:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Cuándo usarlo:** para solicitar la cancelación. La respuesta confirma la solicitud; la conclusión puede ser asíncrona.

### Parámetros

| Campo | Ubicación | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | - | UUID, clave NF-e, AWB **o** `channel_identifier_number` |
| `channel_identifier_number` | path | string | **sí** (variante por canal) | - | - | Identificador comercial |
| `channel_code` | path | integer | no | - | - | Código del canal. Solo los valores **numéricos** se interpretan como canal |
| `reason` | body | string | no | 10 | 255 | Motivo de la cancelación, en texto libre |

> **Cómo se interpreta la ruta:**
> - `DELETE /v1/shipping/{valor}/{numero}` es siempre **cancelación por canal** (`channel_identifier_number` + `channel_code`).
> - `DELETE /v1/shipping/{valor}` busca primero un envío cuyo `channel_identifier_number` sea `{valor}`;
>   si no lo encuentra, trata `{valor}` como UUID, clave NF-e o AWB.

### Payload mínimo

Sin cuerpo:

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SU_TOKEN'
```

### Payload máximo

```json
{
  "reason": "Cliente solicito la cancelacion del pedido"
}
```

### Ejemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/PED-2026-0001/1 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"reason":"Cliente solicito la cancelacion del pedido"}'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Solicitação de cancelamento da remessa 9c06fd75-d5f4-40eb-a1a8-f12086f49944 recebida. Em breve será processada; consulte o status para acompanhar."
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | `success` cuando la solicitud de cancelación fue aceptada |
| `message` | string | Confirmación, con el UUID del envío |

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Esta remessa não foi encontrada!` | Identificador inválido o envío de otra cuenta |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | El status actual no permite la cancelación (ver tabla abajo) |
| `422` | Validación | `reason` con menos de 10 o más de 255 caracteres; `channel_code` inexistente |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Error inesperado |

**Cuándo se acepta la cancelación:**

| Status actual del envío | Resultado |
|---|---|
| `received`, `processing_payment`, `paid_out` | Cancelado al instante (`cancelled`) |
| `shipped` | Pasa a `cancelling` y se cancela en el transportista; algunos contratos cancelan directamente (`cancelled`) |
| `error` sin envío registrado en el transportista | Cancelado al instante (`cancelled`) |
| `processing_shipment` | Aceptado solo en contratos específicos; en los demás, rechazado |
| Demás (`collecting`, `in_transit`, `delivered`, `cancelled`...) | Rechazado con `Remessa não atende requisitos para cancelamento :id !` |

> Después de solicitar la cancelación, consulta el [status](api-rastreio.md): pasa por `cancelling`
> y, cuando lo confirma el transportista, pasa a `cancelled`.

## Cotizaciones no elegibles para envío

Antes de registrar el envío, la API verifica la cotización en este orden. La primera condición que falle interrumpe
la creación:

| # | Verificación | Status | Mensaje |
|---|---|---|---|
| 1 | `quotation` existe y fue creado en la **v1** | `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| 2 | La cotización pertenece a tu cuenta | `404` | `Nenhuma cotação foi encontrada com os dados especificados!` |
| 3 | Cuenta restringida a DDP: la cotización DDU debe estar pagada | `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` |
| 4 | Documentos fiscales de remitente y destinatario | `400` | Ver [Documentos fiscales obligatorios](#documentos-fiscales-obligatorios) |
| 5 | Cotización aún no usada | `406` | `Cotação já foi utilizada em outra remessa!` |
| 6 | No es `simple` de `package`/`pallet` | `403` | `Cotação Simples é inelegível para remessa!` |
| 7 | Cotización dentro de la validez | `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` |
| 8 | El contrato está en la lista de la cotización | `404` | `Contrato especificado não foi listado para esta cotação!` |
| 9 | Tipo de pago habilitado para la cuenta | `401` | `Opção de pagamento não autorizado!` |
| 10 | NF-e informada cuando el contrato la exige | `400` | `Numero da nota fiscal é requerido` |

> Para saber, **antes** de enviar, si una cotización ya fue usada, consulta
> `GET /v1/quotation/{quotation}` y mira `data.finished`. Detalles de los escenarios elegibles en
> [Cotización (v1)](api-cotacao-v1.md#cotizaciones-no-elegibles-para-envío).

## Reglas de negocio

### Coherencia entre cotización y envío
- Con dirección manual (Forma B), la localidad es siempre la de la cotización.
- Con `address_code` (Forma A), la dirección registrada debe tener **el mismo país, estado y ciudad** de la cotización.
  Las divergencias retornan `406`:
  - `O país :country é diferente do enviado na cotação!`
  - `O Estado :state é diferente do enviado na cotação!`
  - `A cidade :city é diferente do enviado na cotação!`

### Documentos fiscales obligatorios
| País de la dirección | `type` | `foreign` | `federal_tax_id` | `state_tax_id` |
|---|---|---|---|---|
| `BR` | `pf` | `false` (predeterminado) | **Obligatorio** — CPF válido | **Obligatorio** |
| `BR` | `pj` | `false` (predeterminado) | **Obligatorio** — CNPJ válido | **Obligatorio** |
| `BR` | cualquiera | `true` | **Obligatorio** (documento extranjero, sin validación de CPF/CNPJ) | Opcional |
| `MX`, `GR` | cualquiera | - | **Obligatorio** | Opcional |
| Otros | cualquiera | - | Opcional | Opcional |

- Para `type: pf` en Brasil, la inscripción estatal no puede ser `ISENTO`/`ISENTA`.
- Para `type: pj` en cualquier país, un `federal_tax_id` con formato de CNPJ se valida como CNPJ.
- Con `address_code`, los documentos fiscales se leen del registro de la dirección y pasan por las mismas reglas.

### Restricciones de transportista
- Para contratos DHL con remitente **persona jurídica en Brasil** y objeto diferente de documento, la `nfkey` es
  obligatoria. Su ausencia retorna `400` con `Numero da nota fiscal é requerido`.

### Predefiniciones aplicadas en el envío
| Predefinición | Efecto |
|---|---|
| Centro de costo | Rellena `cost_center_code` cuando no se envía, o se envía vacío/nulo |
| Finalidad del envío | Rellena `type` cuando no se envía |
| Facturador | Rellena `biller` cuando no se envía (o se envía vacío) y la cuenta muestra facturador |
| Notificación de cobro | Rellena `payment_notify_shipping_billing` cuando no se envía |

### Cancelación automática
Los envíos que permanecen **29 días** sin número de rastreo, en un status anterior al despacho, pueden ser cancelados
automáticamente por la plataforma. Detalles en [Rastreo de Envío](api-rastreio.md).

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Documento fiscal, NF-e obligatoria, contrato por `slug` inválido, cancelación no permitida | Leer `message` y corregir el escenario |
| `401` | Pago no autorizado | Habilitar el tipo de pago en la cuenta |
| `403` | Cotización simple/expirada, o sin permiso para la acción | Rehacer la cotización / solicitar el permiso al soporte |
| `404` | Cotización de otra cuenta, envío/contrato/dirección no encontrado | Verificar el identificador y el `carrier_code`/`slug` |
| `406` | Cotización ya usada o `address_code` incompatible con la cotización | Hacer una nueva cotización / usar la dirección cotizada |
| `422` | Validación de campos, cotización de otra versión, cuenta solo DDP, adjunto inválido | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Guarda siempre el `quotation`: es el identificador más estable del envío.
- Informar `channel_identifier_number` en la cotización es la forma más práctica de consultar y cancelar
  usando el número de tu propio pedido.
- Para informar ciudad, estado y código postal **en el envío** (y no en la cotización), usa el
  [Envío V2](api-remessa-v2.md). Diferencias en [Cotización & Envío: v1 vs v2](api-comparativo-cotacao-remessa.md).
- Para cotización + envío + documentos en **una única llamada síncrona**, consulta [Manifiesto](api-manifesto.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
