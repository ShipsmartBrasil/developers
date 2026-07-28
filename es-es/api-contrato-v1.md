# Contrato

- [Volver al README del idioma](README.md)

## Visión general
- Lista los contratos disponibles para el usuario autenticado.
- Los contratos se usan en la cotización y en la creación de remessas.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- El listado puede filtrar por `status`.

## Flujo de uso
1. Autentique al usuario.
2. Liste los contratos disponibles.
3. Use el `code` del contrato en la cotización o remessa.

## Endpoints
### Listar contratos
- Método y ruta: `GET /v1/contract`
- Cuándo usar: para identificar contratos activos/inactivos antes de la cotización.

## Parámetros
| Nombre | Local | Tipo | Obligatorio | Descripción | Ejemplo | Observaciones |
|---|---|---:|---:|---|---|---|
| `status` | query | string | no | Filtra contratos | `active` | `active`, `inactive`, `all` |

## Payload mínimo y máximo
- No se aplica al endpoint de listado.

## Ejemplo de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=active' \
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
        "code": 3,
        "contract_description": "Contrato padrão PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "Carrier Name",
        "initial_date": "2023-03-06 00:00:00",
        "final_date": "2023-06-30 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ]
  }
}
```

## Reglas de negocio
- Si no hay contratos en el primer filtro, la API intenta una segunda búsqueda con comportamiento alternativo del backend.
- El retorno depende de la vinculación del usuario con los contratos.

## Errores comunes
| Status | Significado | Cómo corregir |
|---|---|---|
| `500` | Error inesperado | Accionar al equipo responsable |
