# Envío V2

- [Volver al README del idioma](README.md)

## Visión general
- Crea envíos con contrato simplificado.
- La v2 no expone, en la fuente analizada, endpoints de status, documentos, cancelación o actualización.
- Versión documentada: v2.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- `quotation` debe haber sido creada en la v2.
- `payment.type` sigue definiendo el flujo financiero.

## Flujo de uso
1. Cree la cotización en la v2.
2. Envíe el envío con el `quotation` retornado.
3. Siga el resto del flujo usando los endpoints de la v1, si corresponde al contrato legado.

## Endpoints
### Crear envío
- Método y ruta: `POST /v2/shipping`
- Cuándo usar: nuevo flujo de shipping.
- Reglas importantes: la cotización debe ser v2; `payment.type` debe estar permitido para el usuario.

#### Diferencias en relación con la v1
- La v2 mantiene el endpoint de creación, pero la documentación pública no muestra los demás endpoints del ciclo.
- `attachments` en la v2 acepta solo `PDF` o `XML` según la fuente analizada.
- El contrato es más estricto para el uso de campos de dirección.

#### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sí | ID de la cotización | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Debe ser v2 |
| `carrier_code` | body | integer | sí | Contrato | `1` | Debe existir |
| `address_sender` | body | object | sí | Remitente | - | Acepta campos manuales o `address_code` |
| `address_receiver` | body | object | sí | Destinatario | - | Acepta campos manuales o `address_code` |
| `payment.type` | body | string | sí | Tipo de pago | `billed` | `billed`, `gateway`, `afterward` |
| `attachments` | body | array | no | Anexos | - | PDF/XML en la fuente |
| `biller` | body | object | no | Facturante | - | Opcional |
| `nfkey` | body | string | no | Clave NF-e | - | 44 dígitos |

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

#### Payload máximo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "nfkey": "12345678901234567890123456789012345678901234",
  "type": "Commercial",
  "invoice_number": "NF-123",
  "shipment_reference": "REF-2024-01",
  "carrier_code": 1,
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
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
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ]
}
```

#### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Payload mínimo y máximo
- El payload mínimo exige solo los campos indispensables para identificación de la cotización, transportador, direcciones y pago.
- El payload máximo incluye NF-e, anexos, facturante y referencia comercial.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `422` | Validación | Ajustar contrato y campos obligatorios |
| `400` | Regla de negocio | Verificar elegibilidad de la cotización y del pago |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- La v2 documentada en la fuente es reducida; para soporte operativo de documentos y seguimiento, la fuente analizada apunta a la v1.
- Todas las acciones de cotización y envío en la segunda versión pueden acompañarse mediante las otras APIs de la primera versión.
