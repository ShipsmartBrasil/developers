# Plataforma

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Endpoint utilitario que devuelve el nombre, el entorno y la versión de la API Pública.
- Es la llamada más simple y barata de la API — ideal para **validar el token y la conectividad** antes de
  cualquier integración, y para **health check** en monitoreo.
- Solo exige un token válido: **no depende de ningún permiso** del perfil.
- Es de solo lectura y no tiene efectos secundarios.
- Este endpoint no aparece en la documentación interactiva (`/doc`); esta página es su referencia.

## URL base
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
lang: pt
```

> Este endpoint **exige un token válido**, pero no exige ningún permiso específico.
> Por eso es la mejor prueba para saber si tu token está funcionando.
> El header `lang` (`pt`, `en`, `es`) solo altera el texto de `message`.

## Flujo de uso

```
1. GET /v1/platform/version   → 200: token válido y API accesible
2. Verificar data.env         → entorno esperado (producción u homologación)
3. Seguir con la integración  → cotización, envío etc.
```

## Endpoint: versión de la plataforma

- **Método y ruta:** `GET /v1/platform/version`
- **Cuándo usar:** al configurar un token nuevo, al cambiar de entorno y en verificaciones periódicas de salud.

### Parámetros

Ningún parámetro de path, query ni cuerpo. Solo los headers de [Autenticación](#autenticación).

### Payload mínimo

Sin cuerpo:

```
GET /v1/platform/version
```

### Payload máximo

No hay parámetros opcionales. La única variación posible es el idioma del mensaje, vía header:

```
GET /v1/platform/version
lang: en
```

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Accept: application/json' \
  --header 'lang: pt'
```

### Ejemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "production",
    "version": "1.0.0"
  }
}
```

### Campos de la respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | string | Siempre `success` cuando se acepta el token |
| `message` | string | `Registro(s) listado(s) com sucesso.` (traducido según el header `lang`) |
| `data.name` | string \| null | Nombre de la aplicación |
| `data.env` | string \| null | Entorno al que estás conectado (ej.: `production`, `staging`) |
| `data.version` | string \| null | Versión publicada de la API |

> Verifica el `env` para asegurarte de que no estás apuntando al entorno equivocado — es un error común
> en integraciones que comparten configuración entre entornos.
>
> Los valores de `data` son informativos y están definidos por la configuración de cada entorno. No los uses como
> clave de lógica (ej.: comparación exacta de versión) y trata `null` como valor posible.

## Casos de uso

| Caso | Cómo usar |
|---|---|
| Validar un token nuevo | Si responde `200`, el token es válido |
| Diagnosticar `404` de autenticación | Si este endpoint también responde `404`, el problema es el token, no la ruta |
| Diagnosticar `403 Has no permission!` en otra ruta | Si este endpoint responde `200`, el token es válido y solo falta el permiso de esa operación |
| Confirmar el entorno | Leer `data.env` |
| Health check de monitoreo | Llamada ligera, sin efectos secundarios |
| Registrar la versión en tus logs | Útil para correlacionar el comportamiento con releases de la API |

## Reglas de negocio

| Regla | Detalle |
|---|---|
| Autenticación | El token se valida antes de cualquier respuesta. Sin header `Authorization`, la respuesta es `404` |
| Permiso | Ninguno. Cualquier token válido de usuario activo recibe `200` |
| Cuenta bloqueada | El token de un usuario bloqueado recibe `403` |
| Versión en el path | Las llamadas sin el prefijo (`/platform/version`) reciben `308` hacia `/v1/platform/version`, preservando la query string |
| Solo lectura | No altera cotizaciones, envíos ni registros |

## Errores comunes

| Status | Significado/Mensaje | Cómo corregir |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Token ausente, incorrecto o regenerado. Verificar el header `Authorization: Bearer ...` y, si es necesario, generar un nuevo token en el panel |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Cuenta bloqueada. Contactar al soporte |
| `400` | `status: "error"` con la descripción del fallo en la autenticación | Fallo inesperado al validar el token. Repetir con backoff; si persiste, contactar al soporte |
| `308` | Redirección permanente | Llamada sin `/v1`. Usar la ruta completa `/v1/platform/version` |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Este endpoint es **de solo lectura** y no tiene ningún efecto sobre cotizaciones, envíos ni registros.
- Las llamadas sin el prefijo de versión (`/platform/version`) se redirigen con `308` hacia
  `/v1/platform/version`. Prefiere siempre informar la versión.
- Detalles de token, idioma y formato de las respuestas: [Primeros Pasos](api-primeiros-passos.md).
- Lista consolidada de mensajes de error: [Códigos de Error](api-codigos-de-erro.md).
