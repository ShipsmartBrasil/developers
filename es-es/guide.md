# Guía Técnica de Integración

## Acceso
- Documentación: `https://api.shipsmart.global/docs`
- Base URL: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Resumen
- Primero se hace la cotización y después la remesa.
- V1 es el flujo completo.
- V2 es el flujo simplificado para cotización y remesa, con entrada solo por país.
- La remesa es irreversible después de creada, salvo NF-e y anexos en casos excepcionales.
- El rastreo devuelve `timeline` y `status` juntos.
- El manifiesto se usa cuando se necesita un flujo síncrono.

## Temas
- Cotización
- Remesa
- Documentos de remesa
- Rastreo
- Agendamiento de recogida
- Manifiesto
- Borradores
- Cajas
- Ítems
- Contratos
