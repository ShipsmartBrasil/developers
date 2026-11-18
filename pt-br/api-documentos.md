# Documentos de Remessa

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Recupera os documentos gerados pela plataforma e pelo transportador após o processamento da remessa:
  AWB (etiqueta), invoice, packing list, declaração e outros.
- Também devolve os **anexos** que você enviou na criação/atualização da remessa.
- Cada documento vem com uma `url` para download e, opcionalmente, com o conteúdo em `base64`.
- Funciona para remessas criadas na **v1 e na v2**.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
lang: pt
```

## Fluxo de uso
1. Crie a [remessa](api-remessa-v1.md).
2. Consulte o [status](api-rastreio.md) até a remessa aparecer no grupo `shipped` — é nesse momento que
   os documentos passam a existir.
3. Consulte os documentos por `{shipping}` ou pelo canal.
4. Baixe o arquivo pela `url` (recomendado) ou decodifique o `base64`.

> **Antes de `shipped` a consulta pode retornar `400 Nenhum documento encontrado com o código especificado!`.**
> Isso é esperado: aguarde o processamento no transportador.

## Endpoints

### Consultar documentos por remessa
- **Método e rota:** `GET /v1/shipping/{shipping}/docs`

### Consultar documentos por canal
- **Método e rota:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- Útil quando você só guardou o número do seu próprio pedido.

## Parâmetros

| Nome | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | UUID da cotação, número do AWB (últimos 90 dias) ou chave da NF-e (44 dígitos) |
| `channel_identifier_number` | path | string | **sim** (variante por canal) | - | Identificador comercial informado na cotação |
| `channel_code` | path | integer | não | - | Código do canal de venda |
| `attachments` | query | boolean | não | `true` | Incluir os anexos enviados por você |
| `embedded` | query | boolean | não | `true` | Incluir documentos incorporados (AWB + invoice consolidados em um único arquivo). Requer permissão específica |
| `base64` | query | boolean | não | `true` | Incluir o conteúdo do arquivo em Base64, além da `url` |
| `documents` | query | string | não | todos | Lista de tipos separados por vírgula. Quando vazio ou ausente, retorna todos |

> **Dica de performance:** envie `base64=false` quando for baixar pela `url`. O payload fica muito menor
> e a resposta, bem mais rápida.

## Tipos de documento filtráveis (`documents`)

| Valor em `documents` | Chave em `data.docs` | Conteúdo |
|---|---|---|
| `awb_pdf` | `AWB` | Etiqueta/AWB em PDF |
| `awb_zpl` | `AWB_ZPL` | Etiqueta/AWB em ZPL (impressora térmica) |
| `awb_prc_pdf` | `AWB PRC` | AWB do processo de retorno/parceiro em PDF |
| `awb_prc_zpl` | `AWB PRC ZPL` | AWB do processo de retorno/parceiro em ZPL |
| `awb_prc_fedex_pdf` | `AWB PRC Fedex` | AWB FedEx do processo de retorno/parceiro em PDF |
| `awb_prc_fedex_zpl` | `AWB PRC Fedex ZPL` | AWB FedEx do processo de retorno/parceiro em ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice em PDF |
| `packing_list_pdf` | `Packing List` | Packing list em PDF |
| `declaration_pdf` | `Declaration` | Declaração de conteúdo em PDF |
| `non_dangerous_pdf` | `Non-Dangerous Object` | Declaração de mercadoria não perigosa em PDF |

Exemplo de filtro:
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> Nem todo documento existe em toda remessa: o conjunto depende do transportador, do país de destino e do
> tipo de objeto. Apenas os documentos efetivamente gerados aparecem em `data.docs`.

## Exemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=false&base64=false&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

Por canal:
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Exemplo de response

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
    "tracking_id": "1234567890123",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "Packing List": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Non-Dangerous Object": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML da Nota Fiscal", "base64": "PD94bWwg..." }
    ]
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.carrier` | string | Nome do transportador que processou a remessa |
| `data.tracking_id` | string | Número de rastreio/AWB |
| `data.docs` | object | Mapa `nome do documento → { url, base64 }`. As chaves são as da tabela acima |
| `data.docs.*.url` | string | URL para download direto do arquivo |
| `data.docs.*.base64` | string | Conteúdo em Base64. Presente apenas quando `base64=true` |
| `data.embedded_docs` | array | Documentos incorporados. `null`/ausente quando `embedded=false` ou sem permissão |
| `data.embedded_docs[].type` | string | `pdf` ou `zpl` |
| `data.embedded_docs[].base64` | string | Conteúdo em Base64 |
| `data.attachments` | array | Anexos enviados por você. `null`/ausente quando `attachments=false` |
| `data.attachments[].id` | integer | **ID do anexo.** Use para removê-lo em `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | `pdf`, `xml`, `jpeg` ou `png` |
| `data.attachments[].description` | string | Descrição informada na criação |
| `data.attachments[].base64` | string | Conteúdo em Base64 |

## Imprimindo etiquetas

| Formato | Quando usar |
|---|---|
| `awb_pdf` | Impressora comum (A4/Letter). Abra a `url` ou decodifique o `base64` |
| `awb_zpl` | Impressora térmica Zebra. Envie o conteúdo **puro** (após decodificar o Base64) direto para a fila da impressora |

> O conteúdo ZPL é texto. Não o converta nem reprocesse: envie exatamente como veio após o decode.

## Gerenciando anexos

| Operação | Rota | Página |
|---|---|---|
| Enviar anexos na criação | `POST /v1/shipping` (campo `attachments`) | [Remessa (v1)](api-remessa-v1.md) |
| Enviar anexos depois | `PUT /v1/shipping/{shipping}` | [Remessa (v1)](api-remessa-v1.md#endpoint-atualizar-remessa) |
| Listar anexos | `GET /v1/shipping/{shipping}/docs?attachments=true` | Esta página |
| Remover anexos | `DELETE /v1/shipping/{shipping}/docs` | [Remessa (v1)](api-remessa-v1.md#endpoint-remover-anexos) |

Limites dos anexos:

| Limite | Valor |
|---|---|
| Formatos aceitos | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamanho por arquivo | **5 MB** |
| Tamanho total da requisição | **90 MB** |

## Regras de negócio
- Os documentos só existem **após** o processamento no transportador (grupo `shipped` no rastreio).
- `embedded_docs` depende de **permissão específica** na sua conta. Sem ela, o campo não é retornado,
  mesmo com `embedded=true`. Mensagem relacionada:
  `Para obter os documentos incoporados a remessa precisa ter sido ao menos enviada.`
- Se não houver nenhum documento nem anexo, a resposta é `400` com
  `Nenhum documento encontrado com o código especificado!`.
- As `url` apontam para armazenamento externo e podem ter validade. **Baixe e armazene** os arquivos
  do seu lado se precisar mantê-los por muito tempo.
- A consulta por canal considera o `channel_identifier_number` informado na **cotação**.
  Sem `channel_code`, a busca usa apenas o identificador.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Aguardar o processamento (status `shipped`) ou conferir o identificador |
| `403` | Sem permissão para consultar documentos | Solicitar a permissão ao suporte |
| `404` | Remessa não encontrada | Conferir `{shipping}` ou o identificador do canal |
| `422` | `channel_code` não numérico | Enviar um inteiro em `channel_code` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- A consulta por `{shipping}` é a forma mais direta e estável.
- A consulta por canal é a melhor opção quando você quer trabalhar apenas com o número do seu pedido.
- Combine `documents` + `base64=false` para respostas rápidas em integrações de alto volume.
