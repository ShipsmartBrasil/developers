# Cotización V2

- [Volver al README del idioma](README.md)

## Visión general
- Genera cotizaciones con un contrato simplificado para integraciones nuevas.
- Úsala cuando solo necesites crear la cotización v2.
- El detalle completo no existe en esta versión en la fuente analizada.
- Versión documentada: v2.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma: `lang: pt|en|es`.

## Convenciones
- `Content-Type: application/json`
- `level` existe en la creación y controla el resultado.

## Flujo de uso
1. Crea la cotización en v2.
2. Reutiliza el `quotation` para la remesa v2.
3. No existe endpoint de consulta de cotización en v2 en la fuente analizada.

## Notas funcionales
- v2 documenta `items` como tipo explícito.
- v2 simplifica los campos de dirección, pero la validación sigue exigiendo los datos necesarios.
- El resultado detallado vuelve en la respuesta del `POST`.

## Endpoint
- `POST /v2/quotation`

## Parámetros resumidos
- `object`: `doc`, `package`, `pallet`, `not_doc`
- `type`: `simple`, `advanced`, `items`
- `tax`: `sender` o `receiver`
- `measurement`: `metric` o `imperial`
