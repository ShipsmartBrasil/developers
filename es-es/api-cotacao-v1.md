# Cotización

- [Volver al README del idioma](README.md)

## Visión general
- Genera cotizaciones internacionales con detalle de flete, seguro e impuestos.
- Use cuando necesite simular el envío antes de crear el envío.
- La cotización es la base para envío, seguimiento y documentos.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Tipo: Bearer token.
- Header: `Authorization: Bearer SEU_TOKEN`
- Header adicional aceptado: `lang: pt|en|es`.
- El idioma estándar es `pt`.

## Convenções
- `Content-Type: application/json`
- Fechas en el formato `Y-m-d H:i:s` en las respuestas.
- Monedas en código ISO 4217 con 3 letras.
- `level` controla el detalle del resultado: `simple` o `detailed`.
- Status HTTP comunes: `200`, `400`, `404`, `406`, `422`, `500`.

## Flujo de uso
1. Envíe una cotización con origen, destino, objeto, tipo e ítems/cajas.
2. Reciba el `quotation` retornado por la API.
3. Consulte el resultado con `GET /v1/quotation/{quotation}/{level}`.
4. Si la cotización es elegible, use el mismo `quotation` para crear el envío.

## Diferencias de contrato
| Endpoint | Diferencia principal | Impacto |
|---|---|---|
| `POST /v1/quotation` | Crea la cotización | Flujo inicial |
| `GET /v1/quotation/{quotation}/{level}` | Consulta resultado | Permite revisar carriers |

## Temas funcionales
### Cotización simple, avanzada y `items`
- `simple`: calcula por caja con peso/valor directo. Para `doc`, es el flujo natural.
- `advanced`: calcula con cajas e ítems distribuidos por `box_id`.
- `items`: la validación del backend acepta este tipo, pero el contrato OpenAPI no lo expone claramente. Validar con el equipo responsable de la API.

### `package`, `pallet` e `doc`
- `doc`: documento. La cotización simple para documentos usa caja con peso y precio.
- `package`: paquete/caja.
- `pallet`: palet. Puede usar `volume` en vez de `height/width/depth`.

### Cotización elegible para envío
- La cotización debe existir, no estar expirada, pertenecer al mismo `api_version` del envío y aún no haber sido usada.
- `package`/`pallet` con tipo `simple` no son elegibles para envío.
- El envío también exige contrato compatible y tipo de pago permitido.

## Endpoints
### Crear cotización
- Objetivo: calcular flete, seguro e impuestos.
- Método y ruta: `POST /v1/quotation`
- Cuándo usar: antes de crear envío.
- Reglas importantes: origen y destino obligatorios; `currency_quote` y `currency_payment` obligatorios; `measurement` acepta `metric` o `imperial`.

#### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Canal de venta | `1` | Debe existir en el registro |
| `channel_identifier_number` | body | string | no | Identificador del canal | `334455` | Libre |
| `object` | body | string | sí | Objeto enviado | `package` | `doc`, `package`, `pallet`, `not_doc` en la fuente |
| `type` | body | string | sí | Tipo de la cotización | `advanced` | `simple`, `advanced`, `items` en la validación |
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
| `declared_freight` | body | object | no | Flete declarado | `{"declared":true,"value":0,"currency":"USD"}` | Solo cuando sea aplicable |
| `level` | query | string | no | Nivel de detalle | `simple` | `simple` o `detailed` |

#### Payload mínimo
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

#### Payload máximo
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
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
      "name": "Caixa 1",
      "height": 20,
      "width": 20,
      "depth": 20
    }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "BR",
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

#### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/quotation \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
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
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

#### Ejemplo de response
```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "description": null,
    "channel": "Domenic Dooley",
    "channel_identifier_number": "334455",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "valid_until": "2021-09-28 20:28:10",
        "transit_days": 5,
        "freight": "181.95",
        "insurance": "11.45",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

#### Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Falla de regla de negocio | Ajustar peso, CEP, elegibilidad o tipo de cotización |
| `406` | Payload inconsistente | Verificar campos obligatorios y tipos |
| `422` | Validación | Corregir campos obligatorios/formatos |
| `500` | Error inesperado | Repetir y accionar al equipo responsable |

## Consultar cotización
### Método y ruta
- `GET /v1/quotation/{quotation}/{level}`
- `level` obligatorio en el path: `simple` o `detailed`.

### Ejemplo
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Response resumida
```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

## Observaciones finales
- La respuesta puede traer `warnings` cuando haya alertas de registro o de transporte.
- La fuente analizada muestra divergencia entre documentación OpenAPI y validación en algunos campos. Validar con el equipo responsable de la API antes de integrar en producción.
