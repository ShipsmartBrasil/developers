# Contrato

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Lista los contratos de transporte disponibles para el usuario autenticado, de forma paginada.
- Un "contrato" es la combinación **transportista + servicio + condiciones comerciales** de tu cuenta.
  Es lo que la cotización retorna en `carriers[]`.
- Usa este endpoint para diagnóstico y para saber de antemano qué transportistas pueden aparecer
  en tus cotizaciones.
- Solo lectura: los contratos se configuran comercialmente, no vía API.

> **No necesitas** llamar a este endpoint en el flujo normal de integración. La cotización ya devuelve los
> contratos elegibles para ese envío específico, con precio y plazo.

## URL base
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
lang: pt
```

> Tu perfil necesita tener el permiso de listar contratos. Sin él, la respuesta es `403` con `Has no permission!`.

## Flujo de uso

```
1. GET /v1/contract                 → contratos activos y vigentes (predeterminado)
2. GET /v1/contract?status=all      → incluir también los desactivados
3. GET /v1/contract?status=all&page=2 → páginas siguientes (repite el status en cada página)
```

### Cuándo usarlo

| Situación | Cómo ayuda |
|---|---|
| La cotización retorna `Nenhum transportador encontrado para este usuário!` | Confirma si la cuenta tiene algún contrato activo y vigente |
| La cotización retorna `Nenhum contrato disponível para este usuário!` | Confirma la vinculación de contratos |
| Quieres mostrar al usuario los transportistas habilitados | Lista nombres y vigencias |
| Recibiste `Contrato especificado não foi listado para esta cotação!` | Ayuda a verificar el `code` esperado |

## Conceptos esenciales

| Concepto | Significado |
|---|---|
| Contrato vinculado | Contrato asociado directamente a tu cuenta. Si tu usuario está vinculado a una cuenta principal, valen los contratos de la cuenta principal |
| Contrato predeterminado | Contrato predeterminado de la plataforma, usado cuando la cuenta no tiene contratos vinculados |
| Vigencia | Período entre `initial_date` y `final_date`. **Solo se listan los contratos vigentes hoy**, independientemente del filtro `status` |
| Activo / inactivo | `active` = contrato habilitado; `inactive` = contrato desactivado (aún dentro de la vigencia). **No** se refiere a la vigencia |

## Endpoint: listar contratos

- **Método y ruta:** `GET /v1/contract`
- **Cuándo usarlo:** diagnóstico de cotización sin transportistas o exhibición de los transportistas habilitados.

### Parámetros

| Parámetro | Ubicación | Tipo | Obligatorio | Predeterminado | Valores | Descripción |
|---|---|---|---|---|---|---|
| `status` | query | string | no | `active` | `active`, `inactive`, `all` | Filtra por situación del contrato (habilitado/desactivado). Los valores vacíos o desconocidos caen en el predeterminado `active`, sin error |
| `page` | query | integer | no | `1` | entero ≥ 1 | Página del listado. Los valores inválidos se tratan como `1` |

- **Ítems por página:** 15 (fijo; no hay parámetro para modificarlo).
- **Ordenación:** no garantizada. No dependas del orden de los ítems.

### Payload mínimo

Sin cuerpo y sin query string:

```
GET /v1/contract
```

### Payload máximo

Todos los parámetros aceptados:

```
GET /v1/contract?status=all&page=2
```

### Ejemplo de request

```bash
# Contratos activos (predeterminado)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'

# Todos los contratos, activos e inactivos, página 2
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all&page=2' \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK`

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
        "direct_quotation": false,
        "active": "active"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/contract?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos de la respuesta

Ítems de `data.data[]`:

| Campo | Tipo | Descripción |
|---|---|---|
| `code` | integer | **Código del contrato.** Es el mismo valor que aparece en `carriers[].code` en la cotización y que envías en `carrier_code` en el envío |
| `contract_description` | string \| null | Descripción comercial del contrato |
| `contract_name` | string | Nombre del contrato |
| `country` | string | País de origen del contrato (ISO 3166-1 alpha-2) |
| `carrier` | string \| null | Nombre del servicio del transportista |
| `initial_date` | string | Inicio de la vigencia (`AAAA-MM-DD HH:MM:SS`) |
| `final_date` | string | Fin de la vigencia (`AAAA-MM-DD HH:MM:SS`) |
| `direct_quotation` | boolean | `true` cuando el contrato cotiza directamente en el transportista (en tiempo real), en lugar de usar tabla de flete |
| `active` | string | `active` (habilitado) o `inactive` (desactivado) |

Paginación (en `data`):

| Campo | Tipo | Descripción |
|---|---|---|
| `current_page` | integer | Página actual |
| `from` / `to` | integer \| null | Posición del primer y del último ítem de la página (`null` cuando la página está vacía) |
| `total` | integer | Total de contratos encontrados |
| `per_page` | integer | Ítems por página (`15`) |
| `last_page` | integer | Número de la última página |
| `first_page_url` / `last_page_url` | string | URLs de la primera y de la última página |
| `prev_page_url` / `next_page_url` | string \| null | URLs de la página anterior y de la siguiente (`null` cuando no existen) |
| `path` | string | URL base del listado |
| `links` | array | Enlaces de navegación (`url`, `label`, `active`). El `label` se traduce según el header `lang` |

> ⚠️ Las URLs de paginación **no preservan el filtro `status`**. Al navegar con `status=inactive` o
> `status=all`, construye tú mismo la URL de la página siguiente (`?status=all&page=N`) en lugar de seguir
> `next_page_url` directamente.

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
| La cotización puede retornar **menos** contratos que este endpoint | La cotización considera solo contratos activos del país de origen del envío, con tabla de flete para el tipo de objeto (documento o no documento) y que atiendan a ese peso/dimensión/ruta |
| El envío acepta **solo** contratos listados en la cotización | Enviar un `carrier_code` activo pero ausente en la cotización retorna `404 Contrato especificado não foi listado para esta cotação!` |
| El `slug` no aparece en este listado | Usa el `slug` retornado en `carriers[]` en la cotización |

## Reglas de negocio

| Regla | Detalle |
|---|---|
| Vinculación | El retorno depende de los contratos vinculados a tu cuenta (o a la cuenta principal, para usuarios vinculados) |
| Contratos predeterminados | Si la búsqueda no encuentra ningún contrato vinculado **en la página solicitada**, la plataforma devuelve los contratos predeterminados de la plataforma con el mismo filtro. Por eso, solicitar una página más allá de la última puede traer contratos predeterminados |
| Vigencia | Los contratos fuera de la vigencia (antes de `initial_date` o después de `final_date`) nunca se listan |
| Filtro `status` | `active` (predeterminado), `inactive` o `all`. Cualquier otro valor se trata como `active` |
| Cotización | Los contratos `inactive` (desactivados) **no** aparecen en las cotizaciones |
| Cotización directa | `direct_quotation: true` indica cotización en tiempo real en el transportista. Esos contratos pueden tardar un poco más en responder y dependen de credenciales válidas (`Credenciais inválidas para cotação direta`) |

## Errores comunes

| Status | Significado/Mensaje | Cómo corregirlo |
|---|---|---|
| `403` | `Has no permission!` | Solicitar al soporte el permiso de listar contratos |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contactar al soporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificar el header `Authorization: Bearer ...` |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Repetir con backoff; si persiste, contactar al soporte |

> Los filtros inválidos **no** generan error: un `status` desconocido se convierte en `active` y una `page` inválida en `1`.
> Una lista vacía (`data.data: []`, `total: 0`) significa que no hay contratos vigentes ni contratos predeterminados
> para ese filtro.

## Observaciones finales
- Este endpoint es **solo de lectura**: los contratos se configuran comercialmente, no vía API.
- Úsalo como herramienta de diagnóstico cuando la cotización no retorne transportistas.
- Para entender por qué un contrato específico no apareció en la cotización, consulta
  **Restricciones de carga** en [Cotización (v1)](api-cotacao-v1.md#restricciones-de-carga).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
