# Cotización

## Visión general
- Genera cotizaciones internacionales con desglose de flete, seguro e impuestos.
- Úsala antes de crear la remesa.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma aceptado: `lang: pt|en|es`.

## Convenciones
- `Content-Type: application/json`
- Las fechas usan `Y-m-d H:i:s` en las respuestas.
- `level` controla el detalle: `simple` o `detailed`.

## Flujo de uso
1. Envía la cotización con origen, destino, tipo de objeto y cajas/ítems.
2. Recibe el código `quotation`.
3. Consulta el resultado con `GET /v1/quotation/{quotation}/{level}`.
4. Si es elegible, reutiliza el mismo `quotation` para crear la remesa.

## Notas funcionales
- `simple` calcula por caja.
- `advanced` calcula con cajas e ítems distribuidos por `box_id`.
- `items` es aceptado por validación, aunque el contrato OpenAPI no lo expone con claridad.

## Elegibilidad para remesa
- La cotización debe existir, no estar vencida, pertenecer a la misma versión de API y no haber sido usada.
- `package`/`pallet` con tipo `simple` no son elegibles.
