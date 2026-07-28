# Borrador

- [Volver al README del idioma](README.md)

## Visión general
- Crea, lista, actualiza y elimina borradores de envío.
- El borrador se procesa de forma asíncrona y puede evolucionar a cotización + remessa.
- Asociado al flujo operativo principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- El borrador acepta payloads parciales.
- El `status` del borrador puede ser `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` o `error`.

## Flujo de uso
1. Envíe el borrador con los datos que ya posee.
2. Espere el procesamiento asíncrono.
3. Liste o consulte los borradores.
4. Actualice solo borradores en status `incomplete`.
5. Elimine borradores cuando sea necesario.

## Endpoints
### Crear borrador
- Método y ruta: `POST /v1/draft`

### Actualizar borrador
- Método y ruta: `PUT /v1/draft/{code}`
- Solo acepta edición de borrador `incomplete`.

### Listar borradores
- Método y ruta: `GET /v1/draft`
- Puede filtrar por `drafts.drafts_ids`.

### Eliminar borrador
- Método y ruta: `DELETE /v1/draft`

### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | no | Canal | `1` | - |
| `channel_identifier_number` | body | string | no | Identificador comercial | `334455` | - |
| `carrier_code` | body | integer | no | Transportador preferido | `1` | - |
| `object` | body | string | no | Objeto | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | no | Tipo | `Commercial` | - |
| `tax` | body | string | no | Impuesto | `sender` | - |
| `insurance` | body | boolean | no | Seguro | `true` | - |
| `measurement` | body | string | no | Medidas | `metric` | Default puede venir de preset |
| `currency_quote` | body | string | no | Moneda de la cotización | `USD` | - |
| `currency_payment` | body | string | no | Moneda del pago | `BRL` | - |
| `address_sender` | body | object | no | Remitente | - | Acepta llenado parcial |
| `address_receiver` | body | object | no | Destinatario | - | Acepta llenado parcial |
| `boxes` | body | array | no | Cajas | - | Opcional |
| `items` | body | array | no | Ítems | - | Opcional |
| `payment.type` | body | string | no | Tipo de pago | `billed` | `billed`, `afterward` |

### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP", "city": "Diadema"},
    "address_receiver": {"country_code": "US", "city": "Florida"},
    "boxes": [{"height": 10, "width": 10, "depth": 10}],
    "payment": {"type": "billed"}
  }'
```

### Ejemplo de response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno"
}
```

## Reglas de negocio
- Un borrador sin payload útil retorna warning.
- El backend usa presets de usuario cuando están disponibles.
- Los borradores pueden convertirse en cotización y remessa mediante el procesamiento asíncrono.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | Borrador vacío, no encontrado o no editable | Enviar datos válidos o elegir otro borrador |
| `422` | Validación | Corregir payload |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- El borrador es la mejor opción cuando la integración aún no tiene todos los datos finales.
- Consulte `api-cotacao-v1.md` y `api-remessa-v1.md` para entender lo que el procesamiento asíncrono va a generar.
