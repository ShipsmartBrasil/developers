# Manifesto

- [Volver al README del idioma](README.md)

## Visión general
- Ejecuta el flujo síncrono de cotización + remessa.
- Después de la creación, puede retornar los documentos finales de la operación.
- Asociado al flujo operativo principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- Query opcional `format`: `pdf` o `zpl`.
- El flujo usa `billed` como tipo de pago en el contrato publicado.

## Flujo de uso
1. Envíe los datos completos del manifesto.
2. La API realiza la cotización, crea la remessa y genera documentos.
3. Si `format` es `pdf` o `zpl`, la respuesta favorece ese formato.

## Endpoints
### Crear manifesto
- Método y ruta: `POST /v1/manifest`
- Query params: `format`.

### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `format` | query | string | no | Tipo de retorno | `pdf` | `pdf` o `zpl` |
| `object` | body | string | sí | Objeto | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | sí | Finalidad | `Commercial` | Viene del flujo de negocio |
| `tax` | body | string | sí | Impuesto | `sender` | `sender` o `receiver` |
| `insurance` | body | boolean | sí | Seguro | `true` | - |
| `residential_delivery` | body | boolean | sí | Entrega residencial | `false` | - |
| `non_stackable` | body | boolean | sí | No apilable | `false` | - |
| `measurement` | body | string | sí | Sistema de medida | `metric` | `metric` o `imperial` |
| `ior` | body | boolean | sí | IOR | `false` | - |
| `currency_quote` | body | string | sí | Moneda de la cotización | `USD` | - |
| `currency_payment` | body | string | sí | Moneda de pago | `EUR` | - |
| `address_sender` | body | object | sí | Remitente | - | Manual o `address_code` |
| `address_receiver` | body | object | sí | Destinatario | - | Manual o `address_code` |
| `boxes` | body | array | sí | Cajas | - | Obligatorio |
| `items` | body | array | sí | Ítems | - | Obligatorio excepto `doc` |
| `payment.type` | body | string | sí | Pago | `billed` | Manifesto solo acepta facturado |
| `declared_freight` | body | object | no | Flete declarado | - | Opcional |

### Payload mínimo
```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Florida",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Payload máximo
```json
{
  "format": "pdf",
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto de teste",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Warehouse",
    "street": "Main Street",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678901",
    "state_tax_id": "1234567890",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida",
    "description": "Office",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "federal_tax_id": "7418529630",
    "state_tax_id": "7418529586",
    "foreign": true
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100,
      "selling_unit_value": 110
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Ejemplo de response
```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "000000",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    }
  }
}
```

## Reglas de negocio
- El manifesto solo funciona con pago facturado.
- El documento final puede ser PDF, ZPL o ambos, según el flujo y el transportador.
- Si la cotización es incompatible con la versión, la ejecución falla.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Error del transportador o regla de negocio | Ajustar payload o verificar retorno del transportador |
| `422` | Validación | Corregir campos obligatorios |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- Este endpoint es síncrono y concentra más responsabilidad que los flujos separados de cotización + remessa.
- Recomendado solo para integraciones que realmente necesitan el flujo completo en una llamada.
