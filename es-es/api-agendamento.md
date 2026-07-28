# Programación de Recolección

- [Volver al README del idioma](README.md)

## Visión general
- Crea, consulta y cancela programaciones de recolección vinculadas a la remessa.
- La recolección puede hacerse por cotización/remessa o por AWB.
- Asociado al flujo operativo principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- Fechas en el formato `Y-m-d`.
- Horarios en el formato `H:i:s`.
- La programación debe respetar el país del remitente.

## Flujo de uso
1. Garantice que la remessa esté lista para la programación.
2. Cree la recolección con ventana horaria e instrucción.
3. Consulte el status de la programación.
4. Si es necesario, cancele usando uno de los motivos permitidos.

## Endpoints
### Crear programación
- Método y ruta: `POST /v1/schedule`
- Reglas importantes: informe `shipping` o `awb_number`.

### Consultar programación
- Método y ruta: `GET /v1/schedule/{shipping}`

### Cancelar programación
- Método y ruta: `DELETE /v1/schedule/{shipping}/{reason}`

### Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `shipping` | body/path | string | condicional | UUID de la cotización/remessa | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Obligatorio si `awb_number` no viene |
| `awb_number` | body | string | condicional | AWB | `271249639538` | Obligatorio si `shipping` no viene |
| `date` | body | date | sí | Fecha de la recolección | `2023-11-21` | Formato `Y-m-d` |
| `initial_time` | body | time | sí | Inicio de la ventana | `08:00:00` | Formato `H:i:s` |
| `final_time` | body | time | sí | Fin de la ventana | `18:00:00` | Formato `H:i:s` |
| `instruction` | body | string | sí | Instrucción para recolección | `Deixar na portaria` | 5 a 80 caracteres |
| `address` | body | object | no | Dirección alternativa | - | Debe ser del mismo país del remitente |
| `reason` | path | string | sí | Motivo de la cancelación | `001` | Código fijo |

### Motivos de cancelación
| Código | Significado |
|---|---|
| `001` | Package not ready |
| `002` | Rates too high |
| `003` | Transit time too slow |
| `004` | Take to service center or drop box |
| `005` | Commitment time not met |
| `006` | Reason not given |
| `007` | Other |
| `008` | Pickup modified |

### Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2023-11-21",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Deixar na portaria"
  }'
```

### Ejemplo de response
```json
{
  "status": "success",
  "message": "Agendamento de coleta registrado com sucesso"
}
```

## Reglas de negocio
- Solo es posible programar si la remessa está lista para ello.
- Ya puede existir una programación en progreso, y en este caso la API retorna warning.
- La dirección alternativa debe respetar el mismo país del remitente.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `400` | La programación ya existe o la remessa no está lista | Ajustar el flujo | 
| `422` | Validación | Corregir fecha, hora o campos obligatorios |
| `500` | Error inesperado | Accionar al equipo responsable |

## Observaciones finales
- Consulte el status antes de cancelar para evitar solicitudes redundantes.
