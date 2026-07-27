# Cotización V2

## Visión general
- Genera cotizaciones con un contrato simplificado para integraciones nuevas.
- Úsala cuando solo necesites crear la cotización v2.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma: `lang: pt|en|es`.

## Flujo de uso
1. Crea la cotización en v2.
2. Reutiliza el `quotation` para la remesa v2.
3. No existe endpoint de consulta de cotización en v2 en la fuente analizada.

## Endpoint
### Crear cotización
- `POST /v2/quotation`

### Diferencias principales con v1
- v2 documenta `items` como tipo de cotización.
- v2 simplifica los campos de dirección en el contrato público.
- El resultado detallado viene en la propia respuesta del `POST`.
