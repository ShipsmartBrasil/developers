# Programación de recogida

- [Volver al README del idioma](README.md)

## Visión general
- Crea, consulta y cancela programaciones de recogida vinculadas a la remesa.

## Convenciones
- Fechas en formato `Y-m-d`.
- Horas en formato `H:i:s`.
- La programación debe respetar el país del remitente.

## Endpoints
- `POST /v1/schedule`
- `GET /v1/schedule/{shipping}`
- `DELETE /v1/schedule/{shipping}/{reason}`
