# Guía de Integración Técnica

> Guía de integración con la **API Pública de Shipsmart Global**: cotización internacional, creación de
> envíos, documentos, seguimiento y programación de recogida.

- [Volver al README principal](../README.md)

> ⚠️ Todos los ejemplos de esta guía (payloads, respuestas, códigos, documentos y tokens) son **meramente ilustrativos**
> y **no son datos válidos**.

## Empieza por aquí

| Paso | Página | Lo que aprendes |
|---|---|---|
| 1 | [Primeros Pasos](api-primeiros-passos.md) | Entornos, autenticación, idioma, formato de respuesta, errores y predefiniciones |
| 2 | [Cotización y Envío: v1 vs v2](api-comparativo-cotacao-remessa.md) | Qué versión usar en tu integración |
| 3 | [Cotización](api-cotacao-v1.md) | Calcular flete, seguro e impuestos |
| 4 | [Envío](api-remessa-v1.md) | Transformar la cotización en un envío real |
| 5 | [Seguimiento del Envío](api-rastreio.md) | Acompañar el envío hasta la entrega |
| 6 | [Documentos del Envío](api-documentos.md) | Descargar AWB, invoice y demás documentos |
| 7 | [Programación de Recogida](api-agendamento.md) | Solicitar la retirada del bulto en tu dirección |
| 8 | [Códigos de Error](api-codigos-de-erro.md) | Tratar cada mensaje de error de la API |

## El flujo principal

```
Cotización  ──►  Envío  ──►  Seguimiento  ──►  Documentos  ──►  Recogida
```

1. **Cotizar:** [Cotización](api-cotacao-v1.md) o [Cotización V2](api-cotacao-v2.md)
2. **Enviar:** [Envío](api-remessa-v1.md) o [Envío V2](api-remessa-v2.md)
3. **Acompañar:** [Seguimiento del Envío](api-rastreio.md)
4. **Documentar:** [Documentos del Envío](api-documentos.md)
5. **Recoger:** [Programación de Recogida](api-agendamento.md)

## Formas de hacer un envío

| Flujo | Ruta principal | Característica |
|---|---|---|
| [Cotización](api-cotacao-v1.md) + [Envío](api-remessa-v1.md) | `POST /v1/quotation` → `POST /v1/shipping` | Flujo estándar. Eliges el transportista entre las opciones cotizadas |
| [Cotización V2](api-cotacao-v2.md) + [Envío V2](api-remessa-v2.md) | `POST /v2/quotation` → `POST /v2/shipping` | Para cuando solo se conoce el país en el momento de cotizar |
| [Manifiesto](api-manifesto.md) | `POST /v1/manifest` | **Sincrónico:** cotiza, despacha y devuelve los documentos en una sola llamada |
| [Borrador](api-rascunho.md) | `POST /v1/draft` | **Asíncrono:** acepta payload parcial y evoluciona a cotización + envío |

## APIs de apoyo

| Página | Para qué sirve |
|---|---|
| [Caja](api-caixa-v1.md) | Registrar embalajes y referenciarlos en la cotización |
| [Artículos](api-itens-v1.md) | Registrar productos (peso, dimensiones, HS Code) y usarlos por SKU |
| [Contrato](api-contrato-v1.md) | Listar los contratos de transporte habilitados en tu cuenta |
| [Plataforma](api-plataforma.md) | Probar token y conectividad; health check |

## Índice completo

### Introducción
- [Primeros Pasos](api-primeiros-passos.md)
- [Cotización y Envío: v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Códigos de Error](api-codigos-de-erro.md)

### Cotización
- [Cotización (v1)](api-cotacao-v1.md)
  - [Cotización de Documento](api-cotacao-v1.md#cotización-de-documento)
  - [Cotización de Paquete (Caja) — simple](api-cotacao-v1.md#cotización-de-paquete-caja--simple)
  - [Cotización de Paquete (Caja) — avanzada](api-cotacao-v1.md#cotización-de-paquete-caja--avanzada)
  - [Cotización de Pallet — por LAA](api-cotacao-v1.md#cotización-de-pallet--por-laa)
  - [Cotización de Pallet — por m³](api-cotacao-v1.md#cotización-de-pallet--por-m³)
  - [Cotización con Kit](api-cotacao-v1.md#cotización-con-kit)
  - [Cotizaciones no elegibles para envío](api-cotacao-v1.md#cotizaciones-no-elegibles-para-envío)
- [Cotización (v2)](api-cotacao-v2.md)
  - [Escenarios de cotización v2](api-cotacao-v2.md#escenarios-de-cotización)
  - [Cotización de Artículos (exclusiva de la v2)](api-cotacao-v2.md#cotización-de-artículos)
  - [Cotizaciones no elegibles para envío](api-cotacao-v2.md#cotizaciones-no-elegibles-para-envío)
- [Diferencia entre Cotización v1 y v2](api-comparativo-cotacao-remessa.md#comparativo-de-la-cotización)

### Envío
- [Envío (cotización v1)](api-remessa-v1.md)
  - [Cotizaciones no elegibles para envío](api-remessa-v1.md#cotizaciones-no-elegibles-para-envío)
  - [Actualizar envío](api-remessa-v1.md#endpoint-actualizar-envío)
  - [Cancelar envío](api-remessa-v1.md#endpoint-cancelar-envío)
- [Envío (cotización v2)](api-remessa-v2.md)
- [Diferencia entre Envío v1 y v2](api-comparativo-cotacao-remessa.md#comparativo-del-envío)
- [Manifiesto](api-manifesto.md)
- [Borrador](api-rascunho.md)

### Post-envío
- [Seguimiento del Envío](api-rastreio.md)
- [Documentos del Envío](api-documentos.md)
- [Programación de Recogida](api-agendamento.md)

### Apoyo
- [Caja](api-caixa-v1.md)
- [Artículos](api-itens-v1.md)
- [Contrato](api-contrato-v1.md)
- [Plataforma](api-plataforma.md)

## Referencia rápida de endpoints

### Cotización
| Método | Ruta | Página |
|---|---|---|
| `POST` | `/v1/quotation` | [Cotización](api-cotacao-v1.md) |
| `GET` | `/v1/quotation/{quotation}/{level}` | [Cotización](api-cotacao-v1.md) |
| `POST` | `/v2/quotation` | [Cotización V2](api-cotacao-v2.md) |

### Envío
| Método | Ruta | Página |
|---|---|---|
| `POST` | `/v1/shipping` | [Envío](api-remessa-v1.md) |
| `POST` | `/v2/shipping` | [Envío V2](api-remessa-v2.md) |
| `PUT` | `/v1/shipping/{shipping}` | [Envío](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}` | [Envío](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{channel_identifier_number}/{channel_code}` | [Envío](api-remessa-v1.md) |
| `DELETE` | `/v1/shipping/{shipping}/docs` | [Envío](api-remessa-v1.md) |
| `POST` | `/v1/manifest` | [Manifiesto](api-manifesto.md) |
| `POST` | `/v1/draft` | [Borrador](api-rascunho.md) |
| `PUT` | `/v1/draft/{code}` | [Borrador](api-rascunho.md) |
| `GET` | `/v1/draft` | [Borrador](api-rascunho.md) |
| `DELETE` | `/v1/draft` | [Borrador](api-rascunho.md) |

### Seguimiento y documentos
| Método | Ruta | Página |
|---|---|---|
| `GET` | `/v1/shipping/{shipping}/status` | [Seguimiento](api-rastreio.md) |
| `GET` | `/v1/shipping/status/{channel_identifier_number}/{channel_code}` | [Seguimiento](api-rastreio.md) |
| `GET` | `/v1/shipping/{shipping}/docs` | [Documentos](api-documentos.md) |
| `GET` | `/v1/shipping/docs/{channel_identifier_number}/{channel_code}` | [Documentos](api-documentos.md) |

### Recogida
| Método | Ruta | Página |
|---|---|---|
| `POST` | `/v1/schedule` | [Programación](api-agendamento.md) |
| `GET` | `/v1/schedule/{shipping}` | [Programación](api-agendamento.md) |
| `DELETE` | `/v1/schedule/{shipping}/{reason}` | [Programación](api-agendamento.md) |

### Apoyo
| Método | Ruta | Página |
|---|---|---|
| `GET` `POST` `DELETE` | `/v1/box` | [Caja](api-caixa-v1.md) |
| `GET` `POST` `PUT` `DELETE` | `/v1/item` | [Artículos](api-itens-v1.md) |
| `GET` | `/v1/contract` | [Contrato](api-contrato-v1.md) |
| `GET` | `/v1/platform/version` | [Plataforma](api-plataforma.md) |

## Entornos

| Entorno | URL base | Documentación interactiva |
|---|---|---|
| Producción | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Homologación | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

> Desarrolla siempre en **homologación**. Las nuevas features aparecen allí primero.
