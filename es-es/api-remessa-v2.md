# Envío (v2)

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md) y [Cotización (v2)](api-cotacao-v2.md)

## Visión general
- Transforma una cotización **v2** en un envío real.
- La diferencia central respecto al [Envío v1](api-remessa-v1.md): aquí informas la **dirección completa**
  (ciudad, estado y código postal), porque la cotización v2 recibió solo el país.
- La v2 expone **solamente la creación**. Status, documentos, cancelación, actualización y programación de recogida
  usan los endpoints `/v1/...`, que funcionan normalmente para envíos creados en la v2.

> ⚠️ **El envío es una operación irreversible en la ruta normal.** Una vez creado, entra en procesamiento
> en el transportista.

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
1. POST /v2/quotation                 → quotation + carriers[]   (solo país/estado)
2. POST /v2/shipping                  → envío recibido           (dirección completa)
3. GET  /v1/shipping/{shipping}/status→ acompañar hasta "shipped"
4. GET  /v1/shipping/{shipping}/docs  → descargar AWB, invoice, etc.
5. POST /v1/schedule                  → programar recogida (opcional)
```

1. [Crea la cotización v2](api-cotacao-v2.md) y verifica si es elegible para envío.
2. Envía el envío con el `quotation`, el transportista elegido y la dirección completa.
3. La creación es **asíncrona**: `200` significa "lo recibimos", no "lo despachamos".
4. Acompaña por el [seguimiento](api-rastreio.md) y descarga los [documentos](api-documentos.md).

## Mapa de endpoints por versión

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

> No existe `GET /v2/shipping/...` ni `DELETE /v2/shipping/...`. Usa las rutas `/v1`.

## Endpoint: crear envío

- **Método y ruta:** `POST /v2/shipping`
- **Cuándo usarlo:** después de obtener una cotización **v2** elegible, dentro del plazo de validez.

### Parámetros — raíz del payload

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `quotation` | string | **sí** | - | - | - | UUID de la cotización. Debe haber sido creada en la **v2** |
| `carrier_code` | integer | **sí** si no se envía `slug` | - | - | - | Código del contrato (`carriers[].code` de la cotización) |
| `slug` | string | **sí** si no se envía `carrier_code` | - | 20 | - | Identificador textual del contrato (`carriers[].slug` de la cotización) |
| `address_sender` | object | **sí** | - | - | - | Remitente (ver tabla abajo) |
| `address_receiver` | object | **sí** | - | - | - | Destinatario (ver tabla abajo) |
| `payment` | object | **sí** | - | - | - | Forma de pago |
| `type` | string | no | - | 30 | `Commercial` | Finalidad del envío. Valores: `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `nfkey` | string | no | 44 | 44 | `null` | Clave de la NF-e. **Exactamente 44 dígitos** |
| `invoice_number` | string | no | 1 | 50 | código interno | Número de la factura/invoice |
| `shipment_reference` | string | no | 1 | 50 | ID interno | Referencia del envío |
| `cost_center_code` | integer | no | - | - | `null` | Código del centro de costo. Debe existir y estar habilitado para tu usuario |
| `payment_notify_shipping_billing` | boolean | no | - | - | `false` | Notificar al área de facturación sobre el cobro |
| `attachments` | array | no | - | - | `null` | Documentos adjuntados al envío |
| `biller` | object | no | - | - | preset de la cuenta | Facturador mostrado en la invoice en lugar del remitente |

### Parámetros — `address_sender` y `address_receiver`

**Esta es la principal diferencia de la v2:** ciudad, estado y código postal son obligatorios aquí.

**Forma A — dirección ya registrada:**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `address_code` | integer | **sí** en esta forma | Código de la dirección registrada. Cuando se usa, no envíes los demás campos |

**Forma B — dirección manual:**

| Campo | Tipo | Obligatorio | Mín. | Máx. | Predeterminado | Descripción |
|---|---|---|---|---|---|---|
| `city` | string | **sí** | 3 | 100 | - | Ciudad |
| `postal_code` | string | condicional | - | 12 | - | Código postal/ZIP. Obligatorio si no se informa `state`. **Siempre obligatorio para direcciones en Brasil** |
| `state` | string | condicional | - | - | - | Nombre del estado/provincia. Obligatorio si no se informa `postal_code` |
| `state_code` | string | condicional | - | - | - | Sigla del estado/provincia. Obligatorio si no se informan `state` ni `postal_code` |
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
| `foreign` | boolean | condicional | - | - | `false` | Indica si la persona es extranjera. **Obligatorio cuando el país es `BR` y no se usa `address_code`** |

> **Regla práctica:** informa `city` + (`postal_code` **o** `state_code`) + `street` + `name` + `email` +
> `phone` + `type`. El país viene de la cotización.

### Parámetros — `payment`

| Campo | Tipo | Obligatorio | Valores | Descripción |
|---|---|---|---|---|
| `payment.type` | string | **sí** | `billed`, `gateway`, `afterward` | Modelo financiero del envío |

| Valor | Significado |
|---|---|
| `billed` | Facturado: el valor entra en tu factura periódica |
| `gateway` | Pago inmediato vía gateway, en la moneda definida en `currency_payment` |
| `afterward` | Pago posterior, acordado fuera del flujo automático |

### Parámetros — `attachments[]`

| Campo | Tipo | Obligatorio | Máx. | Descripción |
|---|---|---|---|---|
| `description` | string | **sí** | 100 | Nombre/descripción del documento (usado después para localizar o eliminar el adjunto) |
| `base64` | string | **sí** | - | Contenido del archivo codificado en Base64 |

| Límite | Valor |
|---|---|
| Formatos aceptados | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamaño por archivo | **5 MB** |

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
    "postal_code": "33101",
    "city": "Miami",
    "state_code": "FL",
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
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"postal_code": "09941070", "city": "Diadema", "street": "Calle Principal", "name": "Tienda Ejemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012", "foreign": false},
    "address_receiver": {"postal_code": "33101", "city": "Miami", "state_code": "FL", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Diferencias respecto a la v1

| Tema | v1 | v2 |
|---|---|---|
| Dirección en el envío | Calle, nombre, e‑mail, teléfono, tipo (+ docs fiscales) | Los mismos campos **+ `city`, `state`/`state_code`, `postal_code`** |
| `city` | No aceptado (viene de la cotización) | **Obligatorio**, mín. 3 y máx. 100 caracteres |
| `biller` | Campos de localidad aceptados | Campos de localidad **validados** (`city` mín. 3) |
| Endpoints del ciclo | Creación, status, documentos, actualización, adjuntos, cancelación | **Solamente creación.** El resto usa `/v1` |
| Origen de la cotización | `POST /v1/quotation` | `POST /v2/quotation` |

## Reglas de negocio

### Elegibilidad de la cotización
| Condición | Mensaje cuando falla |
|---|---|
| La cotización existe y pertenece a la cuenta | `Nenhuma cotação foi encontrada com os dados especificados!` |
| Cotización no expirada | `Esta cotação está expirada...` |
| Cotización no usada antes | `Cotação já foi utilizada em outra remessa!` |
| Creada en la **v2** | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` |
| No es `simple` de `package`/`pallet` | `Cotação Simples é inelegível para remessa!` |
| El contrato está en la lista de la cotización | `Contrato especificado não foi listado para esta cotação!` |

### Coherencia entre cotización y envío
El país informado en el envío debe ser el mismo de la cotización:
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

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `400` | Regla de negocio (cotización, dirección, pago, documento fiscal) | Leer `message` y corregir el escenario |
| `401` | Pago no autorizado | Habilitar el tipo de pago en la cuenta |
| `403` | Sin permiso para la acción | Solicitar el permiso al soporte |
| `404` | `Contrato especificado não foi listado para esta cotação!` | Verificar `carrier_code`/`slug` |
| `422` | Validación de campos o cotización de versión incompatible | Corregir los campos listados en `data` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Usa la v2 cuando la dirección completa solo exista en el momento del envío.
- Todo el ciclo operativo posterior a la creación se hace por los endpoints `/v1` — ellos no distinguen la versión
  en la que se creó el envío.
- Para el detalle de cada operación posterior a la creación, consulta [Envío (v1)](api-remessa-v1.md),
  [Seguimiento](api-rastreio.md), [Documentos](api-documentos.md) y [Programación](api-agendamento.md).
