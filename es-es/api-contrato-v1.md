# Contrato

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Lista los contratos de transporte disponibles para el usuario autenticado.
- Un "contrato" es la combinación **transportista + servicio + condiciones comerciales** de tu cuenta.
  Es lo que la cotización retorna en `carriers[]`.
- Usa este endpoint para diagnóstico y para saber de antemano qué transportistas pueden aparecer
  en tus cotizaciones.

> **No necesitas** llamar a este endpoint en el flujo normal de integración. La cotización ya devuelve los
> contratos elegibles para ese envío específico, con precio y plazo.

## Base URL
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
lang: pt
```

## Cuándo usarlo

| Situación | Cómo ayuda |
|---|---|
| La cotización retorna `Nenhum transportador encontrado para este usuário!` | Confirma si la cuenta tiene algún contrato activo |
| La cotización retorna `Nenhum contrato disponível para este usuário!` | Confirma la vinculación de contratos |
| Quieres mostrar al usuario los transportistas habilitados | Lista nombres y vigencias |
| Recibiste `Contrato especificado não foi listado para esta cotação!` | Ayuda a verificar el `code`/`slug` esperado |

## Endpoint: listar contratos

- **Método y ruta:** `GET /v1/contract`

| Parámetro | Lugar | Tipo | Obligatorio | Predeterminado | Valores | Descripción |
|---|---|---|---|---|---|---|
| `status` | query | string | no | `active` | `active`, `inactive`, `all` | Filtra por vigencia del contrato. Los valores desconocidos caen en el predeterminado `active` |
| `page` | query | integer | no | `1` | - | Página del listado |

- **Ítems por página:** 15

### Ejemplo de request

```bash
# Contratos activos (predeterminado)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'

# Todos los contratos, activos e inactivos
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contrato estandar PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "DHL",
        "initial_date": "2026-01-06 00:00:00",
        "final_date": "2026-12-31 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `code` | integer | **Código del contrato.** Es el mismo valor que aparece en `carriers[].code` en la cotización y que envías en `carrier_code` en el envío |
| `contract_description` | string | Descripción comercial del contrato |
| `contract_name` | string | Nombre del contrato |
| `country` | string | País de origen del contrato (ISO 3166-1 alpha-2) |
| `carrier` | string | Nombre del transportista |
| `initial_date` | string | Inicio de la vigencia (`Y-m-d H:i:s`) |
| `final_date` | string | Fin de la vigencia (`Y-m-d H:i:s`) |
| `active` | string | `active` o `inactive` |
| `direct_quotation` | boolean | `true` cuando el contrato cotiza directamente en el transportista (en tiempo real), en lugar de usar tabla de flete |

## Relación con la cotización y el envío

```
GET /v1/contract            → contratos habilitados en la cuenta (visión general)
        │
POST /v1/quotation          → carriers[] : contratos elegibles PARA ESE ENVÍO
        │                     (con code, slug, precio, plazo y valid_until)
        ▼
POST /v1/shipping           → carrier_code (o slug) elegido
```

| Punto de atención | Explicación |
|---|---|
| La cotización puede retornar **menos** contratos que este endpoint | El contrato existe, pero no atiende a ese peso/dimensión/ruta |
| El envío acepta **solo** contratos listados en la cotización | Enviar un `carrier_code` activo pero ausente en la cotización retorna `404 Contrato especificado não foi listado para esta cotação!` |
| El `slug` no aparece en este listado | Usa el `slug` retornado en `carriers[]` en la cotización |

## Reglas de negocio
- El retorno depende de la **vinculación de tu usuario** a los contratos.
- Si no se encuentra ningún contrato con el filtro informado, la plataforma hace una segunda búsqueda
  considerando los contratos predeterminados de la cuenta.
- Los contratos `inactive` (fuera de vigencia) **no** aparecen en las cotizaciones.
- `direct_quotation: true` indica cotización en tiempo real en el transportista. Esos contratos pueden tardar
  un poco más en responder y dependen de credenciales válidas
  (`Credenciais inválidas para cotação direta`).

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `403` | Sin permiso para listar contratos | Solicitar el permiso al soporte |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Este endpoint es **solo de lectura**: los contratos se configuran comercialmente, no vía API.
- Úsalo como herramienta de diagnóstico cuando la cotización no retorne transportistas.
- Para entender por qué un contrato específico no apareció en la cotización, consulta
  **Restricciones de carga** en [Cotización (v1)](api-cotacao-v1.md#restricciones-de-carga).
