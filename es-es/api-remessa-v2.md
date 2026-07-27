# Remesa V2

## Visión general
- Crea remesas con un contrato simplificado.
- La fuente analizada solo expone la creación en v2.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Flujo de uso
1. Crea la cotización en v2.
2. Envía la remesa con el `quotation` devuelto.
3. Continúa el flujo con endpoints de v1 cuando corresponda.

## Endpoint
### Crear remesa
- `POST /v2/shipping`

### Diferencias con v1
- v2 deja solo el endpoint de creación en la documentación pública.
- `attachments` acepta solo PDF o XML en la fuente analizada.
