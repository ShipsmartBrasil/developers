# Comparativo Cotización y Envío

- [Volver al README del idioma](README.md)

## Visión general
- Explica la diferencia práctica entre los flujos de cotización y envío en las dos versiones publicadas.
- Use esta guía para elegir la versión correcta antes de integrar.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticación
- Bearer token en el header `Authorization`.
- Header de idioma: `lang`.

## Regla de uso
- El flujo estándar de integración es: cotización y envío usando los endpoints de la primera versión.
- La segunda versión está indicada cuando no quiere informar datos más allá del país en la cotización.

## Cotización
### Diferencia práctica
- En la primera versión, la cotización acepta el flujo completo con origen, destino, cajas e ítems.
- En la segunda versión, la cotización es más reducida para escenarios en los que el integrador quiere informar menos datos al inicio del proceso.

### Comparativo
| Tema | Primera versión | Segunda versión | Impacto |
|---|---|---|---|
| Objetivo | Cotización completa | Cotización simplificada | Elija el nivel de detalle |
| Dirección | Más campos en el contrato | Contrato más reducido | Menos datos en la entrada |
| Ítems | Flujo completo y avanzado | Flujo explícito para ítems | Mejor para payloads reducidos |
| Consulta | Consulta y status disponibles en el endpoint de la primera versión | Información no identificada en la fuente analizada | Ambas consultas operativas quedan en la primera versión |

## Envío
### Diferencia práctica
- En la primera versión, el envío cubre creación, consulta de status, consulta de documentos, cancelación, actualización y anexos.
- En la segunda versión, la fuente analizada muestra solo la creación del envío.

### Comparativo
| Tema | Primera versión | Segunda versión | Impacto |
|---|---|---|---|
| Creación | Disponible | Disponible | Flujo base común |
| Status | Disponible | Información no identificada en la fuente analizada | El seguimiento operativo depende de la primera versión |
| Documentos | Disponible | Información no identificada en la fuente analizada | La descarga de documentos queda en la primera versión |
| Cancelación | Disponible | Información no identificada en la fuente analizada | El ciclo completo queda en la primera versión |
| Actualización | Disponible solo en casos excepcionales | Información no identificada en la fuente analizada | Como regla, el envío es irreversible después de creado |

## Cuándo usar cada una
1. Use la primera versión cuando quiera el flujo operativo completo.
2. Use la segunda versión cuando el payload inicial de cotización deba ser más simple.
3. Si su flujo depende de status, documentos o cancelación, mantenga la primera versión.
4. Si crea por la segunda versión, el seguimiento de las acciones se realiza mediante las APIs de la primera versión.

## Observaciones finales
- La fuente analizada no trae una documentación equivalente completa de la segunda versión para status, docs y cancelación.
- Para operación completa, la primera versión sigue siendo la referencia funcional.
