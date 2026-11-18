# Cotización y Envío: v1 vs v2

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
Existen dos versiones publicadas para el par **cotización + envío**. Usan el **mismo motor de cálculo**
y los **mismos transportistas**. La diferencia está en **cuándo** informas la dirección completa.

| | v1 | v2 |
|---|---|---|
| Dirección completa | Informada **en la cotización** | Informada **en el envío** |
| Payload de la cotización | País + ciudad + código postal/estado | Solo el país (+ estado opcional) |
| Payload del envío | Calle, nombre, contacto | Calle, nombre, contacto **+ ciudad, estado, código postal** |
| Ciclo operativo completo | Sí, en `/v1` | Creación en `/v2`, resto en `/v1` |

## En una frase

> **Usa la v1** si, en el momento de cotizar, ya sabes a qué ciudad/código postal vas a enviar.
> **Usa la v2** si, en el momento de cotizar, solo sabes el país de destino.

## Regla de oro: no mezcles versiones

Una cotización creada en la v1 **solo** genera un envío en `POST /v1/shipping`.
Una cotización creada en la v2 **solo** genera un envío en `POST /v2/shipping`.

Mezclarlas retorna `422`:
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Comparativo de la cotización

| Tema | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Obligatorio | Obligatorio |
| `address_*.city` | **Obligatorio** | No aceptado |
| `address_*.postal_code` | Obligatorio (o `state`) | No aceptado |
| `address_*.state` | Obligatorio (o `state_code`/`postal_code`) | No aceptado |
| `address_*.state_code` | Obligatorio (o `state`/`postal_code`) | **Opcional** (mejora la precisión) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Idéntico |
| `type` | `simple`, `advanced`, `items` | Idéntico |
| `boxes[]` (incl. `kit`, `volume`) | Soportado | Idéntico |
| `items[]` | Soportado | Idéntico, **sin** `selling_unit_value` |
| `ior` (Importer of Record) | Soportado | **No forma parte del contrato** |
| `declared_freight` | Soportado | Idéntico |
| `manufacturer` | Soportado | Idéntico |
| `level` (`simple`/`detailed`) | Soportado | Idéntico |
| Consultar el resultado después | `GET /v1/quotation/{quotation}/{level}` | **No existe** |
| Warnings aduaneros | Soportado | Idéntico |

### Impacto práctico en la cotización
- **Precisión:** en la v1, la ciudad y el código postal permiten aplicar tasas y restricciones regionales **ya en el cálculo**.
  En la v2, el valor puede refinarse cuando la dirección completa entra en el envío.
- **Reconsulta:** la v2 no tiene endpoint de reconsulta. **Guarda la respuesta del `POST`** si necesitas
  volver a mostrar la cotización al usuario.

## Comparativo del envío

| Tema | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | Cotización v1 | Cotización v2 |
| `carrier_code` / `slug` | Uno de los dos | Idéntico |
| `address_*.street`, `name`, `email`, `phone`, `type` | Obligatorios | Idénticos |
| `address_*.city` | No aceptado | **Obligatorio** (mín. 3, máx. 100) |
| `address_*.postal_code` | No aceptado | **Obligatorio** (o `state`) |
| `address_*.state` / `state_code` | No aceptados | **Obligatorios** (uno de ellos, o `postal_code`) |
| `payment.type` | `billed`, `gateway`, `afterward` | Idéntico |
| `nfkey`, `invoice_number`, `shipment_reference` | Soportados | Idénticos |
| `cost_center_code`, `payment_notify_shipping_billing` | Soportados | Idénticos |
| `attachments` | Soportado | Idéntico |
| `biller` | Soportado | Idéntico, con localidad validada |

## Operaciones posteriores a la creación

Todas las operaciones posteriores a la creación del envío usan **los mismos endpoints `/v1`**, independientemente de la
versión en la que se creó el envío.

| Operación | Ruta | ¿Sirve para envío v1? | ¿Sirve para envío v2? |
|---|---|---|---|
| Consultar status | `GET /v1/shipping/{shipping}/status` | Sí | **Sí** |
| Status por canal | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` | Sí | **Sí** |
| Consultar documentos | `GET /v1/shipping/{shipping}/docs` | Sí | **Sí** |
| Documentos por canal | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` | Sí | **Sí** |
| Actualizar (NF-e, adjuntos, centro de costo) | `PUT /v1/shipping/{shipping}` | Sí | **Sí** |
| Eliminar adjuntos | `DELETE /v1/shipping/{shipping}/docs` | Sí | **Sí** |
| Cancelar | `DELETE /v1/shipping/{shipping}` | Sí | **Sí** |
| Cancelar por canal | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` | Sí | **Sí** |
| Programar recogida | `POST /v1/schedule` | Sí | **Sí** |
| Consultar programación | `GET /v1/schedule/{shipping}` | Sí | **Sí** |
| Cancelar programación | `DELETE /v1/schedule/{shipping}/{reason}` | Sí | **Sí** |

> No existen rutas `GET`, `PUT` o `DELETE` bajo `/v2/shipping`.

## Comparación de los payloads lado a lado

### Cotización

**v1 — dirección completa en la cotización:**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "Camiseta algodon", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

**v2 — solo país/estado en la cotización:**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "Camiseta algodon", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

### Envío

**v1 — sin localidad (vino de la cotización):**
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
  "payment": { "type": "billed" }
}
```

**v2 — con localidad (no vino de la cotización):**
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
  "payment": { "type": "billed" }
}
```

## Cómo elegir

Responde a las preguntas en orden:

1. **¿En el momento de cotizar, ya sabes la ciudad y el código postal de destino?**
   - Sí → **v1**
   - No → **v2**
2. **¿Necesitas reconsultar la cotización después, en otra solicitud?**
   - Sí → **v1** (`GET /v1/quotation/{quotation}/{level}`)
   - No → cualquiera
3. **¿Usas `ior` (Importer of Record) o `selling_unit_value`?**
   - Sí → **v1**
   - No → cualquiera
4. **¿Quieres el mayor nivel de precisión de impuestos ya en la cotización?**
   - Sí → **v1**
   - No → cualquiera

### Escenarios típicos

| Escenario | Versión recomendada | Motivo |
|---|---|---|
| Checkout de e-commerce (dirección ya completada) | **v1** | Cotización precisa, mostrada al cliente final |
| Calculadora de flete en la página del producto (solo país) | **v2** | El cliente aún no informó la dirección |
| Marketplace que recibe el pedido completo vía webhook | **v1** | Todos los datos ya existen |
| Integración B2B con despacho en lote | **v1** | Flujo operativo completo en la misma versión |
| Simulación en landing page internacional | **v2** | Payload mínimo, menos fricción |
| Cotización + envío + documentos en una sola llamada | **[Manifiesto](api-manifesto.md)** | Flujo sincrónico, independiente de v1/v2 |

## Otras formas de hacer un envío

Además del par cotización + envío, existen dos flujos alternativos:

| Flujo | Ruta | Característica |
|---|---|---|
| [Manifiesto](api-manifesto.md) | `POST /v1/manifest` | **Sincrónico**: cotiza, crea el envío en el transportista y devuelve los documentos en la misma respuesta. Solo acepta pago `billed` |
| [Borrador](api-rascunho.md) | `POST /v1/draft` | **Asíncrono**: acepta payload parcial, valida después y evoluciona a cotización + envío. Acepta pago `billed` o `afterward` |

## Observaciones finales
- Las dos versiones conviven: puedes usar la v2 en la vitrina y la v1 en el checkout, siempre que **cada cotización
  genere el envío en su propia versión**.
- Los endpoints de apoyo ([Caja](api-caixa-v1.md), [Artículos](api-itens-v1.md),
  [Contrato](api-contrato-v1.md)) son compartidos y existen solo en `/v1`.
