# Plataforma

- [Volver al README del idioma](README.md)
- Prerrequisito: [Primeros Pasos](api-primeiros-passos.md)

## Visión general
- Endpoint utilitario que devuelve el nombre, el entorno y la versión de la API Pública.
- Es la llamada más simple y barata de la API — ideal para **validar token y conectividad** antes de
  cualquier integración, y para **health check** en monitoreo.

## Base URL
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
```

> Este endpoint **exige un token válido**, pero no exige ningún permiso específico.
> Por eso es la mejor prueba para saber si tu token está funcionando.

## Endpoint: versión de la plataforma

- **Método y ruta:** `GET /v1/platform/version`
- **Parámetros:** ninguno.

### Ejemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer SU_TOKEN' \
  --header 'Accept: application/json'
```

### Ejemplo de response

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
| `data.name` | string | Nombre de la aplicación |
| `data.env` | string | Entorno al que estás conectado (ej.: `production`, `staging`) |
| `data.version` | string | Versión publicada de la API |

> Verifica el `env` para garantizar que no estás apuntando al entorno equivocado — es un error común
> en integraciones que comparten configuración entre entornos.

## Casos de uso

| Caso | Cómo usarlo |
|---|---|
| Validar un token nuevo | Si responde `200`, el token es válido |
| Diagnosticar `404` de autenticación | Si este endpoint también falla, el problema es el token, no la ruta |
| Confirmar el entorno | Leer `data.env` |
| Health check de monitoreo | Llamada liviana, sin efecto colateral |
| Registrar la versión en tus logs | Útil para correlacionar comportamiento con releases de la API |

## Errores comunes

| Status | Significado | Cómo corregirlo |
|---|---|---|
| `404` | Token no encontrado o inválido | Verificar el header `Authorization: Bearer ...` y generar un nuevo token en el panel |
| `403` | Cuenta bloqueada | Contactar al soporte |
| `500` | Error inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Observaciones finales
- Este endpoint es **solo de lectura** y no tiene ningún efecto sobre cotizaciones, envíos o registros.
- Las llamadas sin el prefijo de versión (`/platform/version`) se redirigen con `308` a
  `/v1/platform/version`. Prefiere siempre informar la versión.
