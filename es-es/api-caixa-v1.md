# Caja

- [Volver al README del idioma](README.md)

## Visión general
- Gestiona el registro y la consulta de cajas usadas en cotizaciones, borradores y remessas.
- Use para estandarizar dimensiones, peso e identificación de los embalajes.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- La consulta usa `boxes[box_ids][]` en la query.
- El registro/actualización ocurre de forma asíncrona.

## Flujo de uso
1. Liste las cajas disponibles.
2. Registre o actualice cajas base.
3. Elimine cajas cuando sea necesario.
4. Reutilice los códigos en las cotizaciones y borradores.

## Endpoints
### Listar cajas
- Método y ruta: `GET /v1/box`
- Cuándo usar: para buscar cajas registradas o filtrar por IDs.

### Crear o actualizar cajas
- Método y ruta: `POST /v1/box`
- Cuándo usar: al mantener la base de embalajes del integrador.

### Eliminar cajas
- Método y ruta: `DELETE /v1/box`
- Cuándo usar: al excluir una o más cajas registradas.

## Parámetros
### Listado
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `boxes[box_ids][]` | query | array | no | Lista de códigos/identificadores | `1` | Vacío retorna todas |

### Registro
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `boxes[]` | body | array | sí | Lista de cajas | - | Cada ítem se procesa en lote |
| `box_code` | body | string | no | Código de la caja | `1` | Opcional para actualización |
| `box_identifier` | body | string | sí | Identificador de la caja | `CX0236` | Obligatorio en el registro |
| `name` | body | string | sí | Nombre | `Cartucho Individual` | Hasta 155 caracteres |
| `height` | body | number | sí | Altura | `65` | >= 1 |
| `width` | body | number | sí | Ancho | `65` | >= 1 |
| `depth` | body | number | sí | Profundidad | `65` | >= 1 |
| `weight` | body | number | sí | Peso | `2.8` | Regex con hasta 3 decimales |
| `activated` | body | boolean | sí | Activada | `true` | - |

### Exclusión
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `boxes.box_ids` | body | array | sí | IDs de las cajas | `[1, 2]` | Lista obligatoria |

## Payload mínimo y máximo
### Payload mínimo
```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload máximo
```json
{
  "boxes": [
    {
      "box_code": 1,
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

## Ejemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "activated": true
      }
    ]
  }'
```

## Ejemplo de response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Reglas de negocio
- El registro se procesa en job asíncrono.
- El listado puede ser cacheado por usuario y página.
- La exclusión acepta múltiples IDs.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `422` | Validación | Corregir campos obligatorios |
| `400` | Solicitud inválida | Verificar payload | 
| `500` | Error inesperado | Accionar al equipo responsable |
