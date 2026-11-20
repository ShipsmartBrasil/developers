# Códigos de Error

- [Volver al README del idioma](README.md)
- Requisito previo: [Primeros Pasos](api-primeiros-passos.md)

> ⚠️ **Ejemplos meramente ilustrativos:** todos los payloads, respuestas, identificadores, códigos, documentos y tokens
> de esta página son ficticios y **no son datos válidos**. Sustitúyelos por los datos reales de tu cuenta y de tu operación.

## Visión general
- Reúne, en un solo lugar, los mensajes de error y aviso que devuelve la API Pública, agrupados por módulo.
- Para cada mensaje: el código HTTP, el texto literal, la causa y cómo corregirlo.
- Los mensajes están en el idioma predeterminado de la API (`pt`). Con `lang: en` o `lang: es`, el texto viene traducido,
  pero el código HTTP y el `status` son los mismos.
- Placeholders como `:name`, `:sku` y `:address` se sustituyen por el valor real en la respuesta.
- El detalle de cada regla está en la página del módulo, enlazada en cada sección.

## URL base
- Producción: `https://api.shipsmart.global`
- Homologación: `https://hmlapi.shipsmart.com.br`
- Documentación interactiva: `https://api.shipsmart.global/doc` y `https://hmlapi.shipsmart.com.br/doc`

## Autenticación
```
Authorization: Bearer SU_TOKEN
Content-Type: application/json
lang: pt
```

## Flujo de uso

Cómo tratar una respuesta que no es `200`:

```
HTTP 422 ──► lee data (campo → mensajes)     ──► corrige el payload   ──► reenvía
HTTP 4xx ──► lee message                     ──► corrige el escenario ──► reenvía
HTTP 500 ──► espera (backoff exponencial)    ──► repite               ──► ¿persiste? contacta al soporte
```

1. Verifica primero el **código HTTP**: define la familia del problema.
2. En `422`, el `message` de nivel superior es genérico — los errores reales están en `data`.
3. En `400`, `401`, `403`, `404` y `406`, el `message` describe la regla de negocio que bloqueó la operación.
4. Repite automáticamente solo los errores `500`. Los errores `4xx` se repiten mientras el payload no sea corregido.

## Conceptos esenciales

### Envoltorio de error

```json
{
  "status": "warning",
  "message": "Cotação já foi utilizada em outra remessa!"
}
```

| Campo | Descripción |
|---|---|
| `status` | `warning` para regla de negocio/validación; `error` para fallo inesperado o rechazo del transportista |
| `message` | Texto del error, traducido según el header `lang` |
| `data` | Presente en `422` (mapa de campos) y en algunos errores del transportista (retorno bruto) |

### Error de validación (`422`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "items.0.hscode": ["O campo items.0.hscode deve conter no mínimo 6 caracteres."],
    "address_sender.city": ["O campo address sender.city é obrigatório quando address sender.address code não está presente."]
  }
}
```

> El `message` del `422` es siempre el mismo mensaje genérico. **Usa `data`** para mostrar el error al usuario.

### Mensajes de validación más frecuentes en `data`

| Mensaje (predeterminado) | Significado |
|---|---|
| `O campo :attribute é obrigatório.` | Campo obligatorio ausente |
| `O campo :attribute é obrigatório quando :other não está presente.` | Obligatoriedad condicional (ej.: dirección manual sin `address_code`) |
| `O campo :attribute não contém um valor válido.` | Valor fuera de la lista aceptada (enum) |
| `O valor selecionado para o campo :attribute é inválido.` | Código de registro inexistente (`channel_code`, `box_code`, `address_code`, `quotation`...) |
| `O campo :attribute deve conter no mínimo :min caracteres.` / `não pode conter mais de :max caracteres.` | Longitud de texto fuera del límite |
| `O campo :attribute deve conter um número inteiro.` / `deve conter um valor numérico.` | Tipo numérico incorrecto |
| `O campo :attribute não contém um endereço de email válido.` | E-mail inválido |
| `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` no habilitado |

## Errores por status HTTP

| Status | `status` típico | Familia | Qué hacer |
|---|---|---|---|
| `308` | - | Llamada sin prefijo de versión | Usar `/v1/...` o `/v2/...` |
| `400` | `warning` / `error` | Regla de negocio | Leer `message` y corregir el escenario |
| `401` | `warning` | Tipo de pago no habilitado | Habilitar la opción de pago en la cuenta |
| `403` | `warning` | Sin permiso, cuenta bloqueada, cotización no elegible | Solicitar permiso / rehacer la cotización |
| `404` | `warning` / `error` | Recurso no encontrado | Verificar identificadores y registros |
| `406` | `warning` | Conflicto de negocio | Leer `message` y ajustar el flujo |
| `422` | `warning` | Validación de campos (o regla sobre datos) | Corregir los campos listados en `data` |
| `500` | `error` | Fallo inesperado | Repetir con backoff; si persiste, contactar al soporte |

## Errores de autenticación y permiso

Valen para **todos** los endpoints.

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Header `Authorization` ausente o token inválido | Enviar `Authorization: Bearer SU_TOKEN` con un token válido |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Cuenta bloqueada | Contactar al soporte |
| `403` | `Has no permission!` | El perfil no tiene el permiso de la acción (cotizar, enviar, consultar documentos, etc.) | Solicitar el permiso al soporte |
| `400` | Mensaje variable (`status: "error"`) | Fallo inesperado al validar el token | Repetir; si persiste, contactar al soporte |

Detalles en [Primeros Pasos](api-primeiros-passos.md#autenticación).

## Errores de cotización

`POST /v1/quotation`, `POST /v2/quotation` y `GET /v1/quotation/{quotation}/{level}`.
Detalles en [Cotización (v1)](api-cotacao-v1.md#reglas-de-negocio).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Caja sin artículo en `advanced` o `items` (v2) | Asociar al menos un artículo (`box_id`) a cada caja |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` fuera del intervalo de cajas | Usar la posición (1, 2, 3...) de la caja en `boxes[]` |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Artículo manual sin `box_id` | Informar `box_id` o `sku` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | Peso superior al máximo de la caja registrada | Reducir el peso o cambiar de caja |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Suma de los artículos superior al máximo de la caja registrada | Redistribuir los artículos |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inexistente o de otra cuenta | Verificar el [registro de cajas](api-caixa-v1.md) |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` no registrado y artículo incompleto | Registrar el artículo o enviar los datos manuales |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | `unit_value` igual o menor que cero | Informar un valor positivo |
| `400` | `O kit :sku é inválido!` | Kit inexistente/inactivo | Revisar el kit en el panel |
| `400` | `A caixa do kit :sku é inválida!` | Caja del kit inexistente | Revisar el kit en el panel |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Artículo del kit eliminado | Revisar el kit en el panel |
| `400` | `Para usar SmartBox, informe ao menos uma caixa em boxes[].` | `type: items` (v2) con agrupación `smartbox` sin `boxes[]` | Enviar las cajas candidatas |
| `400` | `Nenhuma caixa comporta o item :name (Smart Box).` | Ninguna caja enviada admite el artículo | Enviar una caja más grande |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Origen y destino idénticos | Revisar las direcciones |
| `400` | `Nenhum endereco encontrado para o usuário especificado!` | `address_code` inexistente o de otra cuenta | Verificar el código de la dirección |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP brasileño inválido (v1) | Corregir el CEP o enviar `state_code` |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Ningún transportista retornó resultado | Revisar el escenario o hablar con el área comercial |
| `400` | `Não foi encontrada a cotação especificada!` | `GET` de cotización inexistente o de otra cuenta | Verificar el `quotation` |
| `400` | `Nivel de detalhe da cotação não existe` | `level` inválido en el `GET` | Usar `simple` o `detailed` |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` de otra cuenta | Usar un canal de tu cuenta |
| `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` | Dirección brasileña sin CEP (v1) | Informar `postal_code` |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Dirección internacional no localizada (v1) | Revisar ciudad/código postal/estado |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Estado/provincia no reconocido | Enviar un `state_code` válido |
| `404` | `O item com o sku :name não existe!` | `type: items` (v2) con SKU no registrado | Registrar el artículo |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Agrupación `default`: caja vinculada al artículo inexistente | Revisar el vínculo artículo → caja |
| `404` | `Nenhum transportador encontrado para este usuário!` | Cuenta sin contrato para la ruta/objeto | Hablar con el área comercial |
| `404` | `Não há contratos que atendem as especificações do pacote enviado. Se você possui duvidas sobre restrições de envios (como dimensões e pesos), por favor contate o suporte para atendimento!` | Ningún contrato cubre peso/dimensión/ruta | Revisar la carga o hablar con el soporte |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado. Fale com nossa equipe pelo chat para recebermos seu pedido especial.` | Carga fuera del rango automático | Hablar con el equipo comercial |
| `406` | `Codigo de endereço do remetente não encontrado!` | El tipo de cuenta exige remitente registrado | Enviar `address_sender.address_code` |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | HS Code sin tabla de impuesto en el destino | Revisar el `hscode` |
| `400` | `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³` | Pallet sin medidas y sin `volume` | Enviar las medidas o `volume` |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Fallo inesperado | Repetir con backoff |

> Alertas (no bloquean): `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` y
> `Valores declarados dos itens podem ocasionar problemas aduaneiros`, en `data.warnings`.

## Errores de envío

`POST /v1/shipping` y `POST /v2/shipping`. Detalles en [Envío (v1)](api-remessa-v1.md#cotizaciones-no-elegibles-para-envío)
y [Envío (v2)](api-remessa-v2.md#reglas-de-negocio).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inexistente o de otra versión (en `data.quotation`) | Usar el envío de la misma versión de la cotización |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | Cotización de otra cuenta | Usar el token de la cuenta propietaria |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` | Cuenta restringida a DDP con cotización DDU no pagada | Rehacer la cotización con `tax: sender` |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Documento federal ausente | Informar `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brasileño sin inscripción estatal | Informar `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ inválido | Corregir el CNPJ |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF inválido | Corregir el CPF |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | Persona física con IE exenta | Informar una IE válida |
| `406` | `Cotação já foi utilizada em outra remessa!` | Cotización reutilizada | Hacer una nueva cotización |
| `403` | `Cotação Simples é inelegível para remessa!` | `simple` de `package`/`pallet` | Cotizar con `advanced` (o `items`, en la v2) |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` superado | Hacer una nueva cotización |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fuera de `carriers[]` | Usar un contrato de la cotización |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` de contrato inactivo | Usar otro contrato |
| `401` | `Opção de pagamento não autorizado!` | Tipo de pago no habilitado | Habilitarlo en la cuenta o usar otro tipo |
| `400` | `Numero da nota fiscal é requerido` | El contrato exige `nfkey` | Informar la clave de la NF-e |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` inválido | Verificar el código |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` incompatible | Usar la dirección cotizada |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` incompatible | Usar la dirección cotizada |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` incompatible (en la v2, cotización hecha solo con país) | Usar la dirección cotizada / dirección manual |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP inválido (v2) | Corregir el CEP |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Dirección internacional no localizada (v2) | Revisar la dirección |
| `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Estado/provincia no reconocido (v2) | Enviar un `state_code` válido |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Fallo inesperado | Repetir con backoff |

## Errores de actualización, adjuntos y cancelación

`PUT /v1/shipping/{shipping}`, `DELETE /v1/shipping/{shipping}/docs` y `DELETE /v1/shipping/...`.
Detalles en [Envío (v1)](api-remessa-v1.md#endpoint-actualizar-envío).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | `PUT` sin `nfkey`, `attachments` ni `cost_center_code` | Enviar al menos un campo |
| `400` | `Status da remessa não permite atualização na NFE.` | Envío más allá de `shipped` | No es posible vincular la NF-e |
| `400` | `Erro ao salvar arquivo!` | `base64` inválido | Recodificar el archivo |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Adjunto superior a 5 MB | Reducir el archivo |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` inválido o de otra cuenta (`PUT`); AWB/NF-e no localizado | Verificar el identificador |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Envío cancelado/finalizado | No es posible adjuntar |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Formato no aceptado | Convertir el archivo |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML mal formado | Corregir el XML |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Fallo inesperado en el `PUT` | Repetir con backoff |
| `400` | `Remessa não encontrada!` | Eliminación de adjunto con UUID sin envío | Verificar el UUID |
| `400` | `Esta remessa não foi encontrada!` | Cancelación con identificador inválido o de otra cuenta | Verificar el identificador |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | El status actual no permite cancelar | Ver la tabla de cancelación en [Envío (v1)](api-remessa-v1.md#endpoint-cancelar-envío) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Fallo inesperado en la cancelación | Repetir con backoff |

> En la eliminación de adjuntos, los fallos individuales vienen en `data.failed[]` con `reason` (ej.: `Nenhum documento encontrado!`),
> incluso con HTTP `200`.

## Errores de rastreo y documentos

`GET /v1/shipping/.../status` y `GET /v1/shipping/.../docs`.
Detalles en [Rastreo de Envío](api-rastreio.md#errores-comunes) y [Documentos de Envío](api-documentos.md#errores-comunes).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `400` | `remessa.status.status_not_found: ` | Status: ningún envío para el UUID/AWB informado (texto sin traducción) | Usar el UUID de la cotización o el AWB correcto |
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Status: envío de otra cuenta o canal sin envío | Verificar el identificador/canal |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Envío aún sin registro de pago | Esperar; si persiste, contactar al soporte |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Fallo inesperado en el status | Repetir con backoff |
| `400` | `Remessa não encontrada!` | Documentos: UUID sin envío | Verificar el UUID |
| `404` | `Esta remessa não foi encontrada!` | Documentos: AWB/NF-e no localizado | Verificar o usar el UUID |
| `400` | `Nenhum documento encontrado com o código especificado!` | Documentos aún no generados, filtro sin resultado o canal sin envío | Esperar `shipped` / revisar `documents` |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Fallo inesperado en los documentos | Repetir con backoff |

## Errores de programación de recogida

`POST /v1/schedule`, `GET /v1/schedule/{shipping}` y `DELETE /v1/schedule/{shipping}/{reason}`.
Detalles en [Programación de Recogida](api-agendamento.md#errores-comunes).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Fecha de hoy o pasada | Usar una fecha a partir de mañana |
| `400` | `Coleta não pode ser realizada em um final de semana` | Sábado o domingo | Elegir un día hábil |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Feriado nacional | Elegir otro día hábil |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Fecha demasiado lejana | Reducir la antelación |
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Ya existe una programación para el envío | Consultar con `GET` |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Envío fuera de `shipped` | Esperar `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Transportista sin recogida integrada | Llevar el bulto a un punto de entrega |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Dirección de recogida incompleta | Completar la dirección |
| `400` | `O atributo federal_tax_id não é válido` | CPF/CNPJ inválido | Corregir el documento |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Estado de otro país | Usar una dirección del país del remitente |
| `400` | `Agendamento não foi encontrado para esta remessa` | AWB no localizado (`POST`/`DELETE`) | Verificar el AWB o usar el UUID |
| `404` | `Agendamento não foi encontrado para esta remessa` | `GET` sin programación | Verificar el identificador |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Cancelación fuera del status permitido | Cancelar solo programaciones confirmadas |
| `422` | `O campo reason não contém um valor válido.` (en `data`) | `reason` fuera de `001` a `008` | Usar un motivo válido |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Fallo inesperado (incluye clave ausente en `address`) | Enviar todas las claves de `address`; repetir |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Fallo inesperado en la cancelación | Repetir; contactar al soporte |

## Errores de manifiesto

`POST /v1/manifest`. Detalles en [Manifiesto](api-manifesto.md#errores-comunes).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `400` | `Tipo de pagamento não autorizado!` | Cuenta sin pago facturado | Habilitar la facturación |
| `400` | `O canal especificado não foi encontrado!` | `channel_code` de otra cuenta | Usar un canal de la cuenta |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` inactivo | Usar otro contrato |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Estado no resuelto | Enviar `state_code` |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | `address_code` no visible para la cuenta | Verificar el código |
| `404` | `A transportadora informada não está disponível para esta remessa!` | `carrier_code` sin manifiesto | Omitir o cambiar el `carrier_code` |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Contrato sin despacho síncrono | Contactar al soporte |
| `400` | Retorno del transportista (`status: "error"`, con `data`) | El transportista rechazó el envío | Corregir según el retorno |
| `400` | `MANIFEST WARNING - AWB cannot be generated, carrier not mapped` | Etiqueta no generada después de la aceptación; el envío existe | Buscarla en Documentos; **no** repetir |

> Los errores de la etapa de cotización dentro del manifiesto (cajas, artículos, kits, contratos) vuelven como `400 warning`,
> con los mismos mensajes de la [cotización](#errores-de-cotización).

## Errores de borrador

`POST /v1/draft`, `PUT /v1/draft/{code}`, `GET /v1/draft` y `DELETE /v1/draft`.
Detalles en [Borrador](api-rascunho.md#errores-comunes).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | `PUT` en borrador fuera de `incomplete` | Consultar el status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Código inexistente, de otra cuenta o ya convertido en envío | Verificar el código |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Fallo inesperado | Repetir con backoff |

> Los errores del procesamiento asíncrono del borrador no vuelven en la respuesta HTTP: aparecen en
> `status_descricao` del borrador (ej.: `Nenhum item foi enviado para este rascunho`).

## Errores de caja y artículos

`/v1/box` y `/v1/item`. Detalles en [Caja](api-caixa-v1.md#errores-comunes) y [Artículos](api-itens-v1.md#errores-comunes).

| Status | Mensaje | Causa | Cómo corregirlo |
|---|---|---|---|
| `422` | Validación (en `data`) | Campos de caja inválidos; filtros de listado inválidos | Corregir los campos listados |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Fallo inesperado | Repetir con backoff |
| `200` + informe | `Box Error: box :id not found!` / `Box Error: box :id cannot be deleted, has related item` | Eliminación de caja rechazada en el procesamiento | Usar `box_code` entero / desvincular artículos |
| `200` + informe | `sku => :sku, status => invalid, reason => :campo` | Artículo descartado en el procesamiento asíncrono | Corregir el campo indicado |

> Los registros de cajas y artículos son **asíncronos**: la respuesta `200` confirma la recepción, y los registros
> inválidos se descartan después, con un informe enviado al e-mail de la cuenta.

## Reglas de negocio

### Política de reintento
| Situación | ¿Repetir automáticamente? |
|---|---|
| `500` | Sí, con backoff exponencial (ej.: 2 s, 4 s, 8 s) y límite de intentos |
| `4xx` | No. Corrige antes el payload/escenario |
| Manifiesto con `400` después de la aceptación del transportista | **No.** El envío ya existe: consulta [Rastreo](api-rastreio.md) y [Documentos](api-documentos.md) |
| Envío (`POST /v*/shipping`) con timeout de red | Antes de repetir, consulta el status por el `quotation`: una cotización solo genera un envío |

### Rechazos con `status: "error"`
Algunos rechazos de negocio relacionados con el **peso** o con el **transportista** vienen con `status: "error"` y código
`400`/`404`. Trátalos siempre por el código HTTP y por el `message`, no solo por el `status`.

## Observaciones finales
- Convenciones generales de respuesta en [Primeros Pasos](api-primeiros-passos.md#formato-estándar-de-las-respuestas).
- Cotizaciones que no pueden convertirse en envío: [Cotización (v1)](api-cotacao-v1.md#cotizaciones-no-elegibles-para-envío).
- En caso de duda, informa al soporte el `quotation`, la hora de la llamada y el `message` recibido.
