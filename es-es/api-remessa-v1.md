# Remesa

- [Volver al README del idioma](README.md)

## Visión general
- Crea, actualiza, cancela y consulta remesas.
- También expone documentos y estado/rastreo.

## Base URL
- Producción: `https://api.shipsmart.global`
- Documentación: `https://api.shipsmart.global/docs`
- Homologación: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en `Authorization`.
- Idioma opcional: `lang`.

## Flujo de uso
1. Genera la cotización.
2. Verifica si es elegible.
3. Envía la remesa con el `quotation`.
4. Consulta `status` hasta el procesamiento.
5. Descarga documentos en `/docs`.

## Reglas de negocio
- La cotización `package`/`pallet` con tipo `simple` no es elegible.
- La remesa no puede existir previamente para la misma cotización.
- Una vez creada, la remesa es irreversible en operación normal.

## Endpoint
### Crear remesa
- `POST /v1/shipping`

### Notas
- `quotation` debe ser de v1.
- `attachments` acepta PDF, XML, JPEG y PNG.
