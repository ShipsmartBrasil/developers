# Remesa V2

- [Volver al README del idioma](README.md)

## Visión general
- Crea remesas con un contrato simplificado.
- La fuente analizada solo expone la creación en v2.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma: `lang`.

## Convenciones
- `Content-Type: application/json`
- `quotation` debe haberse creado en v2.
- `payment.type` continúa definiendo el flujo financiero.

## Flujo de uso
1. Crea la cotización en v2.
2. Envía la remesa con el `quotation` devuelto.
3. Continúa el flujo con endpoints de v1 cuando corresponda.

## Endpoint
- `POST /v2/shipping`

## Diferencias con v1
- v2 deja solo el endpoint de creación en la documentación pública.
- `attachments` acepta solo PDF o XML en la fuente analizada.

## Parámetros resumidos
- `quotation`: ID de la cotización v2.
- `carrier_code`: contrato.
- `address_sender` y `address_receiver`: remitente y destinatario.
- `payment.type`: `billed`, `gateway`, `afterward`.
