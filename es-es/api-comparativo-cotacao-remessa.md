# Cotización y Envío: v1 vs v2

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
Existen dos versiones publicadas para el par **cotización + envío**. Usan el **mismo cálculo**
y los **mismos transportistas**. Las diferencias están en **cuándo** informas la dirección completa y en
**cómo** describes la carga: solo la v2 acepta la cotización por artículos, sin informar las cajas.

| | v1 | v2 |
|---|---|---|
| Dirección completa | Informada **en la cotización** | Informada **en el envío** |
| Payload de la cotización | País + ciudad + código postal/estado | Solo el país (+ estado opcional) |
| Cotización por artículos (`type: items`) | No disponible: cajas siempre informadas | **Disponible**: solo el `sku` de los artículos registrados |
| Payload del envío | Calle, nombre, contacto | Calle, nombre, contacto **+ ciudad, estado, código postal** |
| Validación de la dirección | En la cotización | En el envío |
| Ciclo operativo completo | Sí, en `/v1` | Creación en `/v2`, resto en `/v1` |

## En una frase

> **Usa la v1** si, en el momento de cotizar, ya sabes a qué ciudad/código postal vas a enviar.
> **Usa la v2** si, en el momento de cotizar, solo sabes el país de destino **o** quieres cotizar solo por los SKU
> registrados, sin informar las cajas.

## Regla de oro: no mezcles versiones

Una cotización creada en la v1 **solo** genera un envío en `POST /v1/shipping`.
Una cotización creada en la v2 **solo** genera un envío en `POST /v2/shipping`.

Mezclarlas retorna `422`, con el siguiente mensaje en `data.quotation`:
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Comparativo de la cotización

| Tema | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Obligatorio | Obligatorio |
| `address_*.city` | **Obligatorio** | Ignorado |
| `address_*.postal_code` | Obligatorio (o `state`); **siempre obligatorio en Brasil** | Ignorado |
| `address_*.state` | Obligatorio (o `state_code`/`postal_code`) | Ignorado |
| `address_*.state_code` | Obligatorio (o `state`/`postal_code`) | **Opcional** (mejora la precisión) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Idéntico |
| `type` | `simple`, `advanced` | `simple`, `advanced` **y `items`** (exclusivo de la v2) |
| `boxes[]` (incl. `kit`, `volume`) | **Siempre obligatorio** | Obligatorio, excepto en `type: items` (según la agrupación de la cuenta) |
| `items[]` (incl. `ean`, `nsmp`) | Soportado | Idéntico, **sin** `selling_unit_value` |
| `ior` (Importer of Record) | Soportado | **No forma parte del contrato** |
| `declared_freight` | Soportado | Idéntico |
| `manufacturer` | Soportado | Idéntico |
| `level` (`simple`/`detailed`) | Soportado | Idéntico |
| Validación de la dirección (código postal, ciudad, estado) | En la cotización | No hay (se hace en el envío) |
| Consultar el resultado después | `GET /v1/quotation/{quotation}/{level}` | **No existe** |
| Warnings aduaneros | Soportado | Idéntico |
| Predefiniciones de la cuenta (impuesto, seguro automático, etc.) | Aplicadas | Idéntico |
| Escenarios (documento, paquete, pallet, artículos, kit) | [Escenarios de la v1](api-cotacao-v1.md#escenarios-de-cotización) | [Escenarios de la v2](api-cotacao-v2.md#escenarios-de-cotización) |

### Impacto práctico en la cotización
- **Precisión:** en la v1, la ciudad y el código postal permiten aplicar tasas y restricciones regionales **ya en el cálculo**.
  En la v2, el cálculo considera país y estado; la dirección completa entra solo en el envío.
- **Errores de dirección:** en la v1 aparecen en la cotización; en la v2 aparecen en el envío.
- **Reconsulta:** la v2 no tiene endpoint de reconsulta. **Guarda la respuesta del `POST`** si necesitas
  volver a mostrar la cotización al usuario.

## Comparativo del envío

| Tema | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | Cotización v1 | Cotización v2 |
| `carrier_code` / `slug` | Uno de los dos | Idéntico |
| `address_*.street`, `name`, `email`, `phone`, `type` | Obligatorios | Idénticos |
| `address_*.city` | Ignorado (viene de la cotización) | **Obligatorio** (mín. 3, máx. 100) |
| `address_*.postal_code` | Ignorado | **Obligatorio** si no se envía `state` |
| `address_*.state` | Ignorado | **Obligatorio** si no se envía `postal_code` |
| `address_*.state_code` | Ignorado | Opcional; prevalece el de la cotización |
| `address_*.address_code` | Debe coincidir con la dirección de la cotización | Solo funciona si la cotización v2 usó el mismo `address_code` |
| `payment.type` | `billed`, `gateway`, `afterward` | Idéntico |
| `nfkey`, `invoice_number`, `shipment_reference` | Soportados | Idénticos |
| `cost_center_code`, `payment_notify_shipping_billing` | Soportados | Idénticos |
| `attachments` | Soportado | Idéntico |
| `biller` | Soportado | Idéntico, con localidad validada |
| Elegibilidad de la cotización | [No elegibles v1](api-remessa-v1.md#cotizaciones-no-elegibles-para-envío) | [No elegibles v2](api-remessa-v2.md#cotizaciones-no-elegibles-para-envío) |

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
    "phone": "13055550100",
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
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
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
5. **¿Quieres cotizar solo con los SKU registrados, sin informar las cajas?**
   - Sí → **v2** ([Cotización de Artículos](api-cotacao-v2.md#cotización-de-artículos))
   - No → cualquiera

### Escenarios típicos

| Escenario | Versión recomendada | Motivo |
|---|---|---|
| Checkout de e-commerce (dirección ya completada) | **v1** | Cotización precisa, mostrada al cliente final |
| Calculadora de flete en la página del producto (solo país) | **v2** | El cliente aún no informó la dirección |
| Marketplace que recibe el pedido completo vía webhook | **v1** | Todos los datos ya existen |
| Integración B2B con despacho en lote | **v1** | Flujo operativo completo en la misma versión |
| Simulación en landing page internacional | **v2** | Payload mínimo, menos fricción |
| ERP con catálogo de productos ya registrado | **v2** | Cotización por artículos: solo SKU y cantidad, la plataforma arma las cajas |
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
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
