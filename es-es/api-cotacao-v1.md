# Cotización

- [Volver al README del idioma](README.md)

## Visión general
- Genera cotizaciones internacionales con desglose de flete, seguro e impuestos.
- Úsala para simular el envío antes de crear la remesa.
- La cotización es la base para remesa, rastreo y documentos.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma aceptado: `lang: pt|en|es`.
- El idioma por defecto es `pt`.

## Convenciones
- `Content-Type: application/json`
- Fechas en formato `Y-m-d H:i:s` en las respuestas.
- Monedas en código ISO 4217 de 3 letras.
- `level` controla el detalle del resultado: `simple` o `detailed`.
- Status HTTP comunes: `200`, `400`, `404`, `406`, `422`, `500`.

## Flujo de uso
1. Envía una cotización con origen, destino, objeto, tipo e ítems/cajas.
2. Recibe el `quotation` devuelto por la API.
3. Consulta el resultado con `GET /v1/quotation/{quotation}/{level}`.
4. Si la cotización es elegible, usa el mismo `quotation` para crear la remesa.

## Notas funcionales
### Cotización simple, avanzada e `items`
- `simple`: calcula por caja con peso/valor directo. Para `doc`, es el flujo natural.
- `advanced`: calcula con cajas e ítems distribuidos por `box_id`.
- `items`: la validación del backend acepta este tipo, pero el contrato OpenAPI no lo expone con claridad.

### `package`, `pallet` y `doc`
- `doc`: documento. La cotización simple para documentos usa caja con peso y precio.
- `package`: paquete/caja.
- `pallet`: palé. Puede usar `volume` en vez de `height/width/depth`.

### Cotización elegible para remesa
- La cotización debe existir, no estar vencida, pertenecer a la misma versión de API de remesa y no haber sido usada.
- `package`/`pallet` con tipo `simple` no son elegibles para remesa.
- La remesa también exige contrato compatible y tipo de pago permitido.

## Endpoints
### Crear cotización
- Objetivo: calcular flete, seguro e impuestos.
- Método y ruta: `POST /v1/quotation`
- Cuándo usar: antes de crear la remesa.
- Reglas importantes: origen y destino obligatorios; `currency_quote` y `currency_payment` obligatorios; `measurement` acepta `metric` o `imperial`.

### Parámetros
| Nombre | Ubicación | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Canal de venta | `1` | Debe existir en el cadastro |
| `channel_identifier_number` | body | string | no | Identificador del canal | `334455` | Libre |
| `object` | body | string | sí | Objeto enviado | `package` | `doc`, `package`, `pallet`, `not_doc` en la fuente |
| `type` | body | string | sí | Tipo de cotización | `advanced` | `simple`, `advanced`, `items` en validación |
| `tax` | body | string | sí | Quién paga impuestos | `sender` | `sender` o `receiver` |
| `insurance` | body | boolean | sí | Seguro | `true` | - |
| `measurement` | body | string | no | Sistema de medida | `metric` | `metric` o `imperial` |
| `residential_delivery` | body | boolean | no | Entrega residencial | `false` | - |
| `non_stackable` | body | boolean | no | Indica no apilable | `false` | - |
| `ior` | body | boolean | no | Importer of Record | `false` | - |
| `currency_quote` | body | string | sí | Moneda de la cotización | `BRL` | ISO 4217 |
| `currency_payment` | body | string | sí | Moneda de pago | `BRL` | ISO 4217 |
| `address_sender` | body | object | sí | Origen | - | Puede usar `address_code` o campos manuales |
| `address_receiver` | body | object | sí | Destino | - | Puede usar `address_code` o campos manuales |
| `boxes` | body | array | sí | Cajas/paquetes | - | Obligatorio salvo cuando `type=items` |
| `items` | body | array | no | Ítems de la carga | - | Obligatorio en `advanced`/`items` |
| `manufacturer` | body | object | no | Fabricante en la invoice | - | Opcional |
| `declared_freight` | body | object | no | Flete declarado | `{"declared":true,"value":0,"currency":"USD"}` | Solo cuando aplica |
| `level` | query | string | no | Nivel de detalle | `simple` | `simple` o `detailed` |

### Payload mínimo
```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ]
}
```

### Payload máximo
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 8, "width": 8, "depth": 8 },
    { "box_code": 3, "height": 10, "width": 10, "depth": 10 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100
    }
  ],
  "manufacturer": {
    "name": "Factory LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj"
  }
}
```

### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/quotation \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: es' \
  --data '{
    "object": "doc",
    "type": "simple",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "BRL",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

### Ejemplo de respuesta
```json
{
  "status": "success",
  "message": "Cotización realizada con éxito!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "object": "package",
    "type": "advanced",
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "freight": "181.95",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

### Errores comunes
| Estado | Significado | Cómo corregir |
|---|---|---|
| `400` | Transportador no disponible o regla negada | Ajustar país, peso, caja o contrato |
| `422` | Validación | Corregir campos obligatorios |
| `406` | Estructura incompatible | Revisar payload y versión |
| `500` | Error inesperado | Contactar al equipo responsable |

## Observaciones finales
- La documentación OpenAPI de v2 y la validación de request no son idénticas en todos los detalles.
- Para consulta detallada de la cotización, la fuente analizada no trae un endpoint v2 equivalente a `GET /v1/quotation/{quotation}/{level}`.
- Todas las acciones de cotización y remesa en la segunda versión pueden seguirse mediante las otras APIs de la primera versión.
