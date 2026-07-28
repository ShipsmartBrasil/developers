# Cotización V2

- [Volver al README del idioma](README.md)

## Visión general
- Genera cotizaciones con contrato simplificado para integraciones nuevas.
- Use cuando quiera solamente crear la cotización en la v2.
- La consulta detallada del resultado no existe en esta versión en la fuente analizada.
- Versión documentada: v2.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang: pt|en|es`.

## Convenções
- `Content-Type: application/json`
- `level` existe en el request body/query de la creación y controla el retorno del cálculo.
- HTTP más comunes: `200`, `400`, `422`, `500`.

## Flujo de uso
1. Envíe la cotización en la v2.
2. Reciba el `quotation` y use ese código para crear el envío en la v2.
3. No hay endpoint de consulta de cotización en la v2 en la fuente analizada.

## Endpoints
### Crear cotización
- Método y ruta: `POST /v2/quotation`
- Cuándo usar: nuevo flujo de pricing.
- Reglas importantes: `object`, `type`, `tax`, `insurance`, `currency_quote`, `currency_payment`, `address_sender`, `address_receiver`, `boxes` y/o `items` según el tipo.

#### Diferencias funcionales en relación con la v1
- La v2 documenta `items` como tipo de cotización explícitamente.
- La v2 simplifica los campos de dirección en el contrato OpenAPI, pero la validación sigue exigiendo los datos necesarios según origen/destino.
- El resultado detallado se entrega en el propio `POST`.

#### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `object` | body | string | sí | Objeto enviado | `package` | `doc`, `package`, `pallet`, `not_doc` en la fuente |
| `type` | body | string | sí | Tipo de la cotización | `items` | `simple`, `advanced`, `items` |
| `tax` | body | string | sí | Impuesto | `sender` | `sender` o `receiver` |
| `insurance` | body | boolean | sí | Seguro | `true` | - |
| `currency_quote` | body | string | sí | Moneda de cotización | `USD` | ISO 4217 |
| `currency_payment` | body | string | sí | Moneda de pago | `EUR` | ISO 4217 |
| `address_sender` | body | object | sí | Remitente | - | `address_code` o manual |
| `address_receiver` | body | object | sí | Destinatario | - | `address_code` o manual |
| `boxes` | body | array | sí | Cajas/palets | - | Obligatorio salvo cuando `type=items` |
| `items` | body | array | sí/condicional | Ítems | - | Las reglas dependen del tipo |
| `manufacturer` | body | object | no | Fabricante | - | Opcional |
| `declared_freight` | body | object | no | Flete declarado | - | Opcional |

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
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
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

#### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/quotation \
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
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
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

#### Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Transportador no disponible o regla denegada | Ajustar país, peso, caja o contrato |
| `422` | Validación | Corregir campos obligatorios |
| `406` | Estructura incompatible | Revisar payload y versión |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- La documentación OpenAPI de la v2 y la validación de request no son idénticas en todos los detalles. Validar con el equipo responsable de la API.
- Para consulta detallada de la cotización, la fuente analizada no trae endpoint v2 equivalente a `GET /v1/quotation/{quotation}/{level}`.
- Todas las acciones de cotización y envío en la segunda versión pueden acompañarse mediante las otras APIs de la primera versión.
