# Ítems

- [Volver al README del idioma](README.md)

## Visión general
- Gestiona ítems registrados para uso en cotizaciones, borradores y remessas.
- Los ítems pueden consultarse, crearse, actualizarse y eliminarse en lote.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- La consulta usa `items[skus][]` en la query.
- Registro/actualización/removal son asíncronos.

## Flujo de uso
1. Liste los ítems ya registrados.
2. Registre los ítems base.
3. Actualice los ítems cuando haya cambio de dimensión, peso o clasificación.
4. Elimine los ítems obsoletos.

## Endpoints
### Listar ítems
- Método y ruta: `GET /v1/item`

### Crear ítems
- Método y ruta: `POST /v1/item`

### Actualizar ítems
- Método y ruta: `PUT /v1/item`

### Eliminar ítems
- Método y ruta: `DELETE /v1/item`

## Parámetros
### Listado
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `items[skus][]` | query | array | no | Lista de SKUs | `TSH-000-S` | Vacío retorna todos |

### Registro/actualización
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `items[]` | body | array | sí | Lista de ítems | - | Procesamiento en lote |
| `name` | body | string | sí | Nombre del ítem | `Tenis` | Obligatorio |
| `description` | body | string | sí | Descripción | `Nike 44` | Hasta 155 caracteres |
| `category` | body | string | sí | Categoría | `950450` | Valor libre en la fuente |
| `country_code` | body | string | sí | País de origen | `BR` | ISO2 |
| `sku` | body | string | sí | SKU | `oLdTsW222` | Obligatorio |
| `ean` | body | string | no | EAN | `7894900010015` | Opcional |
| `height` | body | integer | sí | Altura | `5` | - |
| `width` | body | integer | sí | Ancho | `5` | - |
| `depth` | body | integer | sí | Profundidad | `5` | - |
| `weight` | body | number | sí | Peso | `0.3` | - |
| `unit_value` | body | number | no | Valor unitario | `220.25` | Opcional |
| `selling_unit_value` | body | number | no | Valor de venta | `330.00` | Opcional |
| `hscode` | body | string | sí | HS code | `950450` | Obligatorio |
| `is_groupable` | body | boolean | sí | Puede agrupar | `true` | Obligatorio |
| `box_code` | body | integer | no | Caja estándar | `3` | Opcional |
| `box_identifier` | body | string | no | Caja estándar | `CX0236` | Opcional |

### Exclusión
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---|---|---|---|---|
| `items.skus` | body | array | sí | SKUs a eliminar | `["TSH-000-S"]` | Obligatorio |

## Payload mínimo y máximo
### Payload mínimo
```json
{
  "items": [
    {
      "name": "Teste prod 1",
      "description": "Teste prod 1",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "hscode": "673057",
      "is_groupable": false
    }
  ]
}
```

### Payload máximo
```json
{
  "items": [
    {
      "name": "Teste prod 1 update",
      "description": "Teste prod 1 update",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "ean": "7894900010015",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "unit_value": 220.25,
      "selling_unit_value": 330,
      "hscode": "673057",
      "is_groupable": false,
      "box_code": 129,
      "box_identifier": "CX0236"
    }
  ]
}
```

## Ejemplo de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Ejemplo de response
```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Tenis",
        "description": "Nike 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "oLdTsW222",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "hscode": "950450",
        "is_groupable": true
      }
    ]
  }
}
```

## Reglas de negocio
- Los ítems se procesan en job asíncrono.
- El listado usa colección por usuario.
- `box_code` y `box_identifier` son opcionales y sirven como vínculo estándar con caja.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `422` | Validación | Corregir campos obligatorios |
| `500` | Error inesperado | Accionar al equipo responsable |
