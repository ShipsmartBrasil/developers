# Envío (v1)

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md) y [Cotización (v1)](api-cotacao-v1.md)

## Visión general
- Transforma una cotización v1 en un envío real: genera AWB, invoice, declaración y los demás documentos.
- Es el módulo más completo del ciclo: creación, consulta de status, consulta de documentos, actualización,
  eliminación de adjuntos y cancelación.
- Los endpoints de status, documentos y cancelación atienden **también** a envíos creados en el
  [Envío V2](api-remessa-v2.md).

> ⚠️ **El envío es una operación irreversible en la ruta normal.** Una vez creado, entra en procesamiento
> en el transportista. Las únicas alteraciones posteriores admitidas son excepcionales: vincular la NF-e cuando no
> fue informada, adjuntar documentos que no se enviaron en la creación e informar el centro de costo.

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

## Flujo de uso

```
1. POST /v1/quotation                 → quotation + carriers[]
2. POST /v1/shipping                  → envío recibido (asíncrono)
3. GET  /v1/shipping/{shipping}/status→ acompañar hasta "shipped"
4. GET  /v1/shipping/{shipping}/docs  → descargar AWB, invoice, etc.
5. POST /v1/schedule                  → programar recogida (opcional)
```

1. [Crea la cotización](api-cotacao-v1.md) y verifica si es **elegible** para envío.
2. Envía el envío con el `quotation` y el transportista elegido (`carrier_code` **o** `slug`).
3. La creación es **asíncrona**: la respuesta `200` significa "lo recibimos", no "lo despachamos".
4. Consulta el [status](api-rastreio.md) hasta que aparezca en `shipped`.
5. Descarga los [documentos](api-documentos.md).
6. Si es necesario, [programa la recogida](api-agendamento.md).

## El identificador `{shipping}`

En todos los endpoints de consulta/alteración, el parámetro de path `{shipping}` acepta **tres formatos**:

| Formato | Ejemplo | Observación |
|---|---|---|
| UUID de la cotización | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Forma recomendada |
| Número del AWB | `271249639538` | Funciona para envíos creados en los **últimos 90 días** |
| Clave de la NF-e | `12345678901234567890123456789012345678901234` | Exactamente 44 dígitos numéricos |

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
| `slug` | string | **sí** si no se envía `carrier_code` | - | 20 | - | Identificador textual del contrato (`carriers[].slug` de la cotización) |
| `address_sender` | object | **sí** | - | - | - | Remitente (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | Destinatario (ver tabla abajo) |
| `payment` | object | **sí** | - | - | - | Forma de pago (ver tabla abajo) |
| `type` | string | no | - | 30 | `Commercial` | Finalidad del envío. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | no | 44 | 44 | `null` | Clave de la NF-e. Cuando se informa, debe tener **exactamente 44 dígitos** |
| `invoice_number` | string | no | 1 | 50 | código interno | Número de la factura/invoice. Si se omite, usa el código interno del envío |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Referencia del envío. Si se omite, usa el ID interno del envío |
| `cost_center_code` | integer | no | - | - | `null` | Código del centro de costo. Debe existir y estar habilitado para tu usuario |
| `payment_notify_shipping_billing` | boolean | no | - | - | `false` | Notificar al área de facturación sobre el cobro del envío |
| `attachments` | array | no | - | - | `null` | Documentos adjuntados al envío (ver tabla abajo) |
| `biller` | object | no | - | - | preset de la cuenta | Facturador mostrado en la invoice en lugar del remitente (ver tabla abajo) |

> **`carrier_code` o `slug`:** envía **uno** de los dos. El contrato informado debe estar en la lista
> `carriers[]` retornada por la cotización; de lo contrario la API responde `404` con
> `Contrato especificado não foi listado para esta cotação!`.

### Parámetros — `address_sender` y `address_receiver`

Dos formas, no las mezcles:

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de la dirección registrada. **Debe ser la misma dirección usada en la cotización** |

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `street` | string | **sí** | - | 70 | - | Calle |
| `name` | string | **sí** | 5 | 35 | - | Nombre del contacto/razón social |
| `email` | string | **sí** | - | 60 | - | E-mail válido (recibe notificaciones del transportista) |
| `phone` | string | **sí** | - | 25 | - | Teléfono |
| `type` | string | **sí** | - | - | `pj` | `pf` (persona física) o `pj` (persona jurídica) |
| `number` | string | no | - | 6 | `null` | Número del inmueble |
| `district` | string | no | - | 30 | `null` | Barrio |
| `complement` | string | no | - | 20 | `null` | Complemento |
| `description` | string | no | - | 155 | `null` | Etiqueta interna de la dirección |
| `phone_extension` | string | no | - | 5 | `null` | Extensión |
| `federal_tax_id` | string | condicional | - | 20 | `null` | CPF/CNPJ o documento federal. **Obligatorio cuando el país es `BR` o `AR`** |
| `state_tax_id` | string | condicional | - | 20 | `null` | Inscripción estatal. **Obligatorio cuando el país es `BR`** |
| `foreign` | boolean | condicional | - | - | `false` | Indica si la persona es extranjera. **Obligatorio cuando el país es `BR` y no se usa `address_code`** — es lo que permite validar correctamente el `federal_tax_id` |

> El país, el estado, la ciudad y el código postal **no** se informan aquí: vienen de la cotización.
> Si necesitas informarlos en el envío, usa el [Envío V2](api-remessa-v2.md).

### Parámetros — `payment`

| Campo | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|
| `payment.type` | string | **sí** | `billed`, `gateway`, `afterward` | Modelo financiero del envío |

| Valor | Significado |
|---|---|
| `billed` | Facturado: el valor entra en tu factura periódica |
| `gateway` | Pago inmediato vía gateway, en la moneda definida en `currency_payment` |
| `afterward` | Pago posterior, acordado fuera del flujo automático |

> El tipo debe estar **habilitado para tu cuenta**. De lo contrario:
> `400 Tipo de pagamento inexistente!` o `401 Opção de pagamento não autorizado!`.

### Parámetros — `attachments[]`

| Campo | Tipo | Obligatorio | Máx. | Descripción |
|---|---|---|---|---|
| `description` | string | **sí** | 100 | Nombre/descripción del documento (usado después para localizar o eliminar el adjunto) |
| `base64` | string | **sí** | - | Contenido del archivo codificado en Base64 |

| Límite | Valor |
|---|---|
| Formatos aceptados | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamaño por archivo | **5 MB** |
| Tamaño total de la solicitud | **90 MB** |

> Archivo fuera del formato: `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.`
> Archivo por encima del límite: `O arquivo enviado excede o limite máximo de 5MB.`
> XML corrupto: `O arquivo XML enviado é inválido ou está corrompido.`

### Parámetros — `biller` (facturador)

Cuando se informa, esta dirección aparece en la invoice en lugar de la dirección del remitente.
Envía `address_code` **o** los campos manuales. **Si no vas a usar facturador, no envíes el campo.**

| Campo | Tipo | Mín. | Máx. | Descripción |
|---|---|---|---|---|
| `address_code` | integer | - | - | Código de dirección registrada |
| `country_code` | string | 2 | 2 | País en ISO 3166-1 alpha-2 |
| `state` | string | - | 100 | Nombre del estado |
| `state_code` | string | - | 10 | Sigla del estado |
| `postal_code` | string | - | 12 | Código postal/ZIP |
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
    "street": "Calle Principal",
    "name": "Tienda Ejemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

> `federal_tax_id`, `state_tax_id` y `foreign` aparecen en el payload mínimo porque el remitente es brasileño.
> Para un remitente de otro país (excepto `AR`), esos tres campos pueden omitirse.

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
    "street": "Calle Principal",
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
    "phone": "11992494597",
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
  "payment": {
    "type": "gateway"
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
    "address_sender": {"street": "Calle Principal", "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

> La respuesta **no** trae AWB ni documentos: el envío solo fue recibido. Usa el
> [seguimiento](api-rastreio.md) para acompañar y los [documentos](api-documentos.md) para descargar los archivos.

### Errores específicos de la creación

| Status | Mensaje | Causa / corrección |
|---|---|---|
| `400` | `Tipo de pagamento inexistente!` | `payment.type` fuera de `billed`/`gateway`/`afterward` o no habilitado |
| `400` | `Esta cotação está expirada. Porfavor, faça outra cotação...` | `valid_until` superado — vuelve a hacer la cotización |
| `400` | `Cotação já foi utilizada em outra remessa!` | Una cotización genera **un** envío |
| `400` | `Cotação Simples é inelegível para remessa!` | Usa `type: advanced` o `items` en la cotización |
| `400` | `Chave da Nota não foi informada!!` | El contrato exige `nfkey` para este escenario |
| `400` | `O endereço especificado, não foi encontrado!` | `address_code` inválido o de otra cuenta |
| `400` | `O país :country é diferente do enviado na cotação!` | La dirección del envío debe coincidir con la de la cotización |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | País `BR`/`AR` sin `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | País `BR` sin `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ con dígito verificador incorrecto |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF con dígito verificador incorrecto |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | `type: pf` con `state_tax_id` exento |
| `401` | `Opção de pagamento não autorizado!` | Habilita el tipo de pago en tu cuenta |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fuera de la lista de la cotización |
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | Cotización v2 siendo usada en `/v1/shipping` |

## Endpoint: consultar el status del envío

- **Método y ruta:** `GET /v1/shipping/{shipping}/status`
- **Por canal:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

| Parámetro | Lugar | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `shipping` | path | string | **sí** | UUID de la cotización, AWB (últimos 90 días) o clave NF-e (44 dígitos) |
| `channel_identifier_number` | path | string | **sí** | Identificador comercial informado en la cotización |
| `channel_code` | path | integer | no | Código del canal de venta |

Detalles, formatos de respuesta y códigos de status en [Seguimiento del Envío](api-rastreio.md).

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: consultar documentos

- **Método y ruta:** `GET /v1/shipping/{shipping}/docs`
- **Por canal:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`

| Parámetro | Lugar | Tipo | Obligatorio | Predeterminado | Descripción |
|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | UUID, AWB o clave NF-e |
| `attachments` | query | boolean | no | `true` | Incluir los adjuntos enviados por ti |
| `embedded` | query | boolean | no | `true` | Incluir documentos incorporados (requiere permiso específico) |
| `base64` | query | boolean | no | `true` | Incluir el contenido en Base64 además de la `url` |
| `documents` | query | string | no | todos | Lista separada por comas con los tipos deseados |

Tipos aceptados en `documents`: `awb_pdf`, `awb_zpl`, `awb_prc_pdf`, `awb_prc_zpl`, `awb_prc_fedex_pdf`,
`awb_prc_fedex_zpl`, `declaration_pdf`, `invoice_pdf`, `non_dangerous_pdf`, `packing_list_pdf`.

Detalles completos en [Documentos del Envío](api-documentos.md).

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

## Endpoint: actualizar envío

- **Método y ruta:** `PUT /v1/shipping/{shipping}`
- **Cuándo usarlo:** solo en casos excepcionales — vincular la NF-e, adjuntar documentos faltantes
  o informar el centro de costo.
- **Regla:** envía `nfkey` y/o `attachments` y/o `cost_center_code`. **Al menos uno** debe estar presente.

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `nfkey` | string | condicional | 44 | 44 | Clave de la NF-e (44 dígitos numéricos) |
| `cost_center_code` | integer | condicional | - | - | Código del centro de costo habilitado para tu usuario |
| `attachments` | array | condicional | - | - | Archivos a adjuntar |
| `attachments[].description` | string | **sí** si se envía `attachments` | - | 100 | Descripción del documento |
| `attachments[].base64` | string | **sí** si se envía `attachments` | - | - | Contenido en Base64. Formatos: `PDF`, `XML`, `JPEG`, `PNG`. Máx. **5 MB** por archivo |

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "cost_center_code": 1,
    "attachments": [
      {"description": "Nota Fiscal", "base64": "JVBERi0xLjcK..."}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | No se envió ninguno de los tres campos |
| `404` | `Remessa não encontrada!` | `{shipping}` inválido |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | El envío ya avanzó demasiado en el flujo |

## Endpoint: eliminar adjuntos

- **Método y ruta:** `DELETE /v1/shipping/{shipping}/docs`
- **Cuándo usarlo:** para borrar un adjunto enviado por error.

| Campo | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|
| `attachments` | array | **sí** | 1 ítem | - | Lista de los adjuntos a eliminar |
| `attachments[].id` | integer | **sí** si no se envía `name` | 1 | - | ID del adjunto (retornado en `GET .../docs`) |
| `attachments[].name` | string | **sí** si no se envía `id` | - | 100 | Descripción exacta del adjunto |

> Envía **`id` O `name`** por ítem, nunca los dos.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

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

> La operación es **parcial**: lo que se eliminó aparece en `deleted` y lo que falló en `failed`,
> con el motivo. Verifica siempre el array `failed`.

## Endpoint: cancelar envío

- **Método y ruta:** `DELETE /v1/shipping/{shipping}`
- **Por canal:** `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}`
- **Cuándo usarlo:** para solicitar la cancelación. El procesamiento es **asíncrono**.

| Campo | Lugar | Tipo | Obligatorio | Mín. | Máx. | Descripción |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sí** | - | - | UUID, AWB o clave NF-e |
| `channel_identifier_number` | path | string | **sí** (variante por canal) | - | - | Identificador comercial |
| `channel_code` | path | integer | no | - | - | Código del canal. Solo los valores **numéricos** se interpretan como canal |
| `reason` | body | string | no | 10 | 255 | Motivo de la cancelación, en texto libre |

> **Atención a la ambigüedad de ruta:** `DELETE /v1/shipping/{valor}/{numero}` se interpreta como
> **cancelación por canal**. Para cancelar por identificador del envío usa siempre
> `DELETE /v1/shipping/{shipping}` (un único segmento).

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"El cliente solicito la cancelacion del pedido"}'
```

```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

| Status | Mensaje | Causa |
|---|---|---|
| `400` | `Remessa especificada não foi encontrada ou não é possivel mais cancelar a mesma` | Identificador inválido o envío en un status que no permite cancelación |
| `422` | Validación | `reason` con menos de 10 o más de 255 caracteres |

> Después de solicitar la cancelación, consulta el [status](api-rastreio.md): pasa por `cancelling`
> y, cuando lo confirma el transportista, va a `cancelled`.

## Reglas de negocio

### Elegibilidad de la cotización
| Condición | Mensaje cuando falla |
|---|---|
| La cotización existe y pertenece a la cuenta | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Cotización no expirada | `Esta cotação está expirada...` |
| Cotización no usada antes | `Cotação já foi utilizada em outra remessa!` |
| Misma versión de API | `Não é possivel finalizar, pois seu quotation pode estar inválido...` |
| No es `simple` de `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| El contrato está en la lista de la cotización | `Contrato especificado não foi listado para esta cotação!` |

### Coherencia entre cotización y envío
La dirección informada en el envío debe ser compatible con la de la cotización. Las divergencias retornan:
- `O país :country é diferente do enviado na cotação!`
- `O Estado :state é diferente do enviado na cotação!`
- `A cidade :city é diferente do enviado na cotação!`

### Documentos fiscales obligatorios
| País de la dirección | `federal_tax_id` | `state_tax_id` | `foreign` |
|---|---|---|---|
| `BR` | Obligatorio (CPF o CNPJ válido) | Obligatorio | Obligatorio sin `address_code` |
| `AR` | Obligatorio | - | - |
| Otros | Opcional | Opcional | Opcional |

- Para `type: pf`, la inscripción estatal no puede ser `ISENTO`/`ISENTA`.
- El `foreign` existe para que la validación del `federal_tax_id` se haga en el formato correcto
  (documento brasileño vs. documento extranjero).

### Restricciones de transportista
- `Cotações e envios via FedEx entre Europa e Oriente Médio estão temporariamente suspensos pela transportadora.`
- Para algunos contratos (ej.: DHL con remitente persona jurídica en Brasil), la `nfkey` es obligatoria.
  Su ausencia retorna `Chave da Nota não foi informada!!`.

### Cancelación automática
Los envíos que permanecen **29 días o más** sin evento de seguimiento en un status elegible pueden ser cancelados
automáticamente por la plataforma.

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Regla de negocio (cotización, dirección, pago, documento fiscal) | Leer `message` y corregir el escenario |
| `401` | Pago no autorizado | Habilitar el tipo de pago en la cuenta |
| `403` | Sin permiso para la acción | Solicitar el permiso al soporte |
| `404` | Envío/contrato no encontrado | Verificar el identificador y el `carrier_code`/`slug` |
| `422` | Validación de campos o envío en un status no alterable | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Guarda siempre el `quotation`: es el identificador más estable del envío.
- Informar `channel_identifier_number` en la cotización es la forma más práctica de consultar y cancelar
  usando el número de tu propio pedido.
- Para informar ciudad, estado y código postal **en el envío** (y no en la cotización), usa el
  [Envío V2](api-remessa-v2.md).
- Para cotización + envío + documentos en **una única llamada sincrónica**, consulta [Manifiesto](api-manifesto.md).
