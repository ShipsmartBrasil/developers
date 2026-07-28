# Envío

- [Volver al README del idioma](README.md)

## Visión general
- Crea, actualiza, cancela y consulta envíos.
- También expone consulta de documentos y status/seguimiento.
- Es la versión legada más completa del flujo de shipping.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- El envío usa `quotation` como vínculo con la cotización.
- El `quotation` debe ser de la misma versión de la API de envío.
- `payment.type` define el modelo financiero del envío.

## Flujo de uso
1. Genere la cotización.
2. Verifique si es elegible para envío.
3. Envíe el envío con el `quotation` retornado.
4. Consulte `status` hasta que el envío sea procesado.
5. Descargue los documentos en `/docs`.
6. Si es necesario, programe recolección.

## Reglas de negocio
- Cotización `package`/`pallet` con tipo `simple` no es elegible para envío.
- La cotización no puede estar expirada.
- El envío no puede haber sido creado anteriormente para la misma cotización.
- El tipo de pago debe estar habilitado para el usuario.
- Para DHL con remitente PJ en Brasil, `nfkey` puede ser obligatoria.
- Después de creado, el envío es irreversible en la operación normal.
- Cambios posteriores solo se admiten en casos excepcionales: envío del número de la nota fiscal cuando no fue informado y envío de documentos para anexar al envío cuando no fueron enviados durante la creación.

## Endpoints
### Crear envío
- Método y ruta: `POST /v1/shipping`
- Cuándo usar: después de obtener una cotización elegible.

#### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sí | ID de la cotización | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Debe ser de la primera versión |
| `carrier_code` | body | integer | sí | Contrato/transportador | `1` | Debe existir para la cotización |
| `address_sender` | body | object | sí | Remitente | - | Puede usar `address_code` o campos manuales |
| `address_receiver` | body | object | sí | Destinatario | - | Puede usar `address_code` o campos manuales |
| `payment.type` | body | string | sí | Tipo de pago | `billed` | `billed`, `gateway`, `afterward` |
| `type` | body | string | no | Finalidad del envío | `Commercial` | Opcional |
| `nfkey` | body | string | no | Clave NF-e | `12345678901234567890123456789012345678901234` | 44 dígitos |
| `invoice_number` | body | string | no | Número de la factura | `NF-123` | Si no se envía, usa el código interno |
| `shipment_reference` | body | string | no | Referencia del envío | `REF-2024-01` | Si no se envía, usa el ID interno |
| `attachments` | body | array | no | Anexos | - | PDF, XML, JPEG, PNG |
| `biller` | body | object | no | Facturante | - | Usado en la invoice |

#### Payload mínimo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "Doe Jhon",
    "email": "doejhon@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

#### Payload máximo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "nfkey": "12345678901234567890123456789012345678901234",
  "type": "Commercial",
  "invoice_number": "NF-123",
  "shipment_reference": "REF-2024-01",
  "carrier_code": 1,
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ],
  "address_sender": {
    "address_code": 1
  },
  "address_receiver": {
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Factory LTDA",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  }
}
```

#### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "Doe Jhon", "email": "doejhon@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

#### Ejemplo de response
```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Consultar status del envío
- Método y ruta: `GET /v1/shipping/{shipping}/status`
- `shipping` acepta: UUID de la cotización, AWB o clave NF-e de 44 dígitos.

### Consultar status por canal
- Método y ruta: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Consultar documentos
- Método y ruta: `GET /v1/shipping/{shipping}/docs`
- Retorna documentos, anexos y, si está permitido, docs incorporados.

### Consultar documentos por canal
- Método y ruta: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Actualizar envío
- Método y ruta: `PUT /v1/shipping/{shipping}`
- Use para anexar documento y/o vincular NF-e en caso excepcional.
- Los archivos anexados deben estar en `base64` y aceptar solo `PDF`, `XML`, `JPEG` o `PNG`.
- El tamaño máximo es de `5 MB` por archivo.

#### Ejemplo de request
```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "attachments": [
      {
        "description": "Nota Fiscal",
        "base64": "JVBERi0xLjcK..."
      }
    ]
  }'
```

#### Ejemplo de response
```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Cancelar envío
- Método y ruta: `DELETE /v1/shipping/{shipping}`
- `reason` es opcional en el body y acepta texto libre.

#### Ejemplo de request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

#### Ejemplo de response
```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

### Eliminar anexos
- Método y ruta: `DELETE /v1/shipping/{shipping}/docs`
- Envíe `attachments` con `id` o `name`.

#### Ejemplo de request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

#### Ejemplo de response
```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!"
}
```

## Consultas y documentos
### Parámetros de documentos
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `attachments` | query | boolean | no | Incluye anexos | `true` | Default `true` |
| `embedded` | query | boolean | no | Incluye docs incorporados | `true` | Default `true` |
| `base64` | query | boolean | no | Incluye base64 | `true` | Default `true` |
| `documents` | query | string | no | Filtra tipos | `invoice_pdf,awb_pdf` | Lista separada por coma |

### Tipos de documentos identificados
- `awb_pdf`
- `awb_zpl`
- `awb_prc_pdf`
- `awb_prc_zpl`
- `awb_prc_fedex_pdf`
- `awb_prc_fedex_zpl`
- `declaration_pdf`
- `invoice_pdf`
- `non_dangerous_pdf`
- `packing_list_pdf`

### Ejemplo de consulta de documentos
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Envío no encontrado o regla de negocio | Verificar `quotation`, `carrier_code` y elegibilidad |
| `401` | Pago no autorizado | Ajustar la configuración de la cuenta |
| `404` | Recurso no encontrado | Confirmar el identificador informado |
| `422` | Validación | Corregir payload |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- El envío v1 es el punto de integración más completo del ciclo operativo.
- Consulte `api-documentos.md` y `api-rastreio.md` para ejemplos dedicados de consulta y seguimiento.
