# Remesa

- [Volver al README del idioma](README.md)

## Visión general
- Crea, actualiza, cancela y consulta remesas.
- También expone documentos y estado/rastreo.
- Es la versión legada más completa del flujo de shipping.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma opcional: `lang`.

## Convenciones
- `Content-Type: application/json`
- La remesa usa `quotation` como vínculo con la cotización.
- La `quotation` debe pertenecer a la misma versión de API.
- `payment.type` define el modelo financiero.

## Flujo de uso
1. Genera la cotización.
2. Verifica si es elegible para remesa.
3. Envía la remesa con el `quotation` devuelto.
4. Consulta `status` hasta el procesamiento.
5. Descarga documentos en `/docs`.
6. Si es necesario, agenda la recogida.

## Reglas de negocio
- `package`/`pallet` con tipo `simple` no son elegibles.
- La cotización no puede estar vencida.
- La remesa no puede existir previamente para la misma cotización.
- El tipo de pago debe estar habilitado para el usuario.
- Una vez creada, la remesa es irreversible en operación normal.
- Los cambios posteriores solo se admiten en casos excepcionales.

## Endpoints
### Crear remesa
- `POST /v1/shipping`

### Parámetros
| Nombre | Ubicación | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sí | ID de la cotización | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Debe ser de v1 |
| `carrier_code` | body | integer | sí | Contrato/transportador | `1` | Debe existir para la cotización |
| `address_sender` | body | object | sí | Remitente | - | Manual o `address_code` |
| `address_receiver` | body | object | sí | Destinatario | - | Manual o `address_code` |
| `payment.type` | body | string | sí | Tipo de pago | `billed` | `billed`, `gateway`, `afterward` |
| `type` | body | string | no | Finalidad | `Commercial` | Opcional |
| `nfkey` | body | string | no | Clave NF-e | `12345678901234567890123456789012345678901234` | 44 dígitos |
| `invoice_number` | body | string | no | Número de factura | `NF-123` | Si no se envía, usa el código interno |
| `shipment_reference` | body | string | no | Referencia | `REF-2024-01` | Si no se envía, usa el ID interno |
| `attachments` | body | array | no | Anexos | - | PDF, XML, JPEG, PNG |
| `biller` | body | object | no | Facturador | - | Usado en la invoice |

### Payload mínimo
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

### Payload máximo
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
  "address_sender": { "address_code": 1 },
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
  "payment": { "type": "gateway" }
}
```

### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: es' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "Doe Jhon", "email": "doejhon@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de respuesta
```json
{
  "status": "success",
  "message": "Recibimos tu remesa con éxito. Sigue el estado usando el mismo código de la cotización."
}
```

### Consultar estado
- `GET /v1/shipping/{shipping}/status`
- `shipping` acepta UUID de cotización, AWB o clave NF-e de 44 dígitos.

### Consultar estado por canal
- `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Consultar documentos
- `GET /v1/shipping/{shipping}/docs`

### Consultar documentos por canal
- `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Actualizar remesa
- `PUT /v1/shipping/{shipping}`
- Usa `base64` y solo acepta `PDF`, `XML`, `JPEG` o `PNG`.
- Tamaño máximo: `5 MB` por archivo.

### Cancelar remesa
- `DELETE /v1/shipping/{shipping}`
- `reason` es opcional en el body y acepta texto libre.

### Ejemplo de cancelación
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

### Ejemplo de respuesta
```json
{
  "status": "success",
  "message": "Solicitud procesada, consulta el estado en breve para seguir el proceso de cancelación"
}
```

## Errores comunes
| Estado | Significado | Cómo corregir |
|---|---|---|
| `400` | Remesa no encontrada o flujo inválido | Verificar el identificador y la cotización |
| `422` | Validación | Corregir campos obligatorios |
| `500` | Error inesperado | Contactar al equipo responsable |

## Observaciones finales
- El endpoint de estado es la fuente canónica para el ciclo de vida operacional.
- Los endpoints de documentos son la forma más rápida de recuperar los archivos generados.
