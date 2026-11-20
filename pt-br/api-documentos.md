# Documentos de Remessa

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Recupera os documentos gerados pela plataforma e pelo transportador após o processamento da remessa:
  AWB (etiqueta), invoice, packing list, declaração e outros.
- Também devolve os **anexos** que você enviou na criação/atualização da remessa.
- Cada documento vem com uma `url` para download e, opcionalmente, com o conteúdo em `base64`.
- Contas com permissão específica recebem também os **documentos incorporados** (todos os PDFs em um único arquivo e todas as etiquetas ZPL em um único conteúdo).
- Funciona para remessas criadas na **v1 e na v2** (as rotas de documentos existem apenas na v1).

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
lang: pt
```

| Header | Obrigatório | Valores | Padrão | Observação |
|---|---|---|---|---|
| `Authorization` | **sim** | `Bearer <token>` | - | Token da sua conta. O perfil precisa da permissão de consulta de documentos de remessa |
| `lang` | não | `pt`, `en`, `es` | `pt` | Idioma das mensagens |

## Fluxo de uso
1. Crie a remessa ([Remessa v1](api-remessa-v1.md) ou [Remessa v2](api-remessa-v2.md)).
2. Consulte o [status](api-rastreio.md) até a remessa aparecer no grupo `shipped` — é nesse momento que
   os documentos passam a existir.
3. Consulte os documentos por `{shipping}` ou pelo canal.
4. Baixe o arquivo pela `url` **imediatamente** ou decodifique o `base64`, e armazene-o do seu lado.

> **Antes de `shipped` a consulta pode retornar `400 Nenhum documento encontrado com o código especificado!`.**
> Isso é esperado: aguarde o processamento no transportador. Se a remessa tiver anexos, a resposta pode ser
> `200` trazendo apenas os anexos (veja [Regras de negócio](#regras-de-negócio)).

## Conceitos essenciais

### Tipos de documento filtráveis (`documents`)

| Valor em `documents` | Chave em `data.docs` | Conteúdo |
|---|---|---|
| `awb_pdf` | `AWB` | Etiqueta/AWB em PDF |
| `awb_zpl` | `AWB_ZPL` e `AWB_ZPL_UPS` | Etiqueta/AWB em ZPL (impressora térmica). Para alguns transportadores a chave é `AWB_ZPL_UPS` |
| `awb_prc_pdf` | `AWBPRC` | Etiqueta AWB PRC em PDF, emitida para alguns transportadores |
| `awb_prc_zpl` | `AWBPRCZPL` | Etiqueta AWB PRC em ZPL |
| `awb_prc_fedex_pdf` | `AWBPRCFedex` | Etiqueta AWB PRC FedEx em PDF |
| `awb_prc_fedex_zpl` | `AWBPRCFedexZPL` | Etiqueta AWB PRC FedEx em ZPL |
| `invoice_pdf` | `Invoice` | Commercial Invoice em PDF |
| `packing_list_pdf` | `PackingList` | Packing list em PDF |
| `declaration_pdf` | `Declaration` | Declaração de conteúdo em PDF |
| `non_dangerous_pdf` | `NonDangerousObject` | Declaração de mercadoria não perigosa em PDF |

> **As chaves de `data.docs` não têm espaços** (`PackingList`, `NonDangerousObject`, `AWBPRC`...), embora a
> documentação interativa as mostre com espaços (`Packing List`, `Non-Dangerous Object`, `AWB PRC`...).
> Programe a leitura pelas chaves desta tabela.

Exemplo de filtro:
```
?documents=invoice_pdf,awb_pdf,awb_zpl
```

> Nem todo documento existe em toda remessa: o conjunto depende do transportador, do país de destino e do
> tipo de objeto. Apenas os documentos efetivamente gerados aparecem em `data.docs`, e cada chave aparece
> **uma única vez**.

### Documentos incorporados (`embedded_docs`)

| `type` | Conteúdo | Condição |
|---|---|---|
| `pdf` | Todos os documentos PDF retornados (após o filtro `documents`) unidos em um único PDF. A quantidade de cópias de cada documento e a inclusão de anexos PDF seguem a configuração de incorporação da sua conta | A remessa precisa já ter sido processada no transportador |
| `zpl` | Todas as etiquetas ZPL retornadas (após o filtro `documents`) concatenadas, separadas por quebra de linha | Existir ao menos um documento ZPL |

Os documentos incorporados só são gerados quando **todas** as condições abaixo são verdadeiras:
`embedded=true`, `base64=true`, a sua conta tem a permissão de documentos incorporados e a remessa tem
ao menos um documento retornado. Caso contrário, `embedded_docs` vem como lista vazia (`[]`).

### Imprimindo etiquetas

| Formato | Quando usar |
|---|---|
| `awb_pdf` | Impressora comum (A4/Letter). Abra a `url` ou decodifique o `base64` |
| `awb_zpl` | Impressora térmica Zebra. Envie o conteúdo **puro** (após decodificar o Base64) direto para a fila da impressora |

> O conteúdo ZPL é texto. Não o converta nem reprocesse: envie exatamente como veio após o decode.

### Gerenciando anexos

| Operação | Rota | Página |
|---|---|---|
| Enviar anexos na criação | `POST /v1/shipping` (campo `attachments`) | [Remessa (v1)](api-remessa-v1.md) |
| Enviar anexos depois | `PUT /v1/shipping/{shipping}` | [Remessa (v1)](api-remessa-v1.md#endpoint-atualizar-remessa) |
| Listar anexos | `GET /v1/shipping/{shipping}/docs?attachments=true&base64=true` | Esta página |
| Remover anexos | `DELETE /v1/shipping/{shipping}/docs` | [Remessa (v1)](api-remessa-v1.md#endpoint-remover-anexos) |

Limites dos anexos:

| Limite | Valor |
|---|---|
| Formatos aceitos | `PDF`, `XML`, `JPEG`, `PNG` |
| Tamanho por arquivo | **5 MB** |
| Tamanho total da requisição | **90 MB** |

> Os anexos **só são retornados com `base64=true`**. Com `base64=false`, `data.attachments` vem sempre vazio.

## Endpoint: consultar documentos por remessa

- **Método e rota:** `GET /v1/shipping/{shipping}/docs`
- **Quando usar:** é a forma mais direta. Aceita o UUID da cotação, o AWB ou a chave da NF-e.

### Parâmetros

| Nome | Local | Tipo | Obrigatório | Tamanho | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | - | Identificador da remessa. Veja as formas aceitas abaixo |
| `attachments` | query | boolean | não | - | `true` | Incluir os anexos enviados por você. Só tem efeito com `base64=true` |
| `embedded` | query | boolean | não | - | `true` | Incluir os documentos incorporados. Só tem efeito com `base64=true` e com a permissão específica |
| `base64` | query | boolean | não | - | `true` | Incluir o conteúdo dos arquivos em Base64, além da `url`. Com `false`, anexos e documentos incorporados não são retornados |
| `documents` | query | string | não | - | todos | Lista de tipos separados por vírgula (valores da tabela [Tipos de documento filtráveis](#tipos-de-documento-filtráveis-documents)). Vazio ou ausente: todos |

Formas de informar `{shipping}` (use apenas uma):

| Forma | Valor | Como é reconhecido | Observações |
|---|---|---|---|
| A | UUID da cotação | Contém hífen (`-`) | Forma recomendada |
| B | Chave da NF-e | Exatamente 44 dígitos numéricos | Considera a remessa mais recente com essa chave, desconsiderando remessas canceladas |
| C | AWB (número de rastreio) | Qualquer outro valor | Apenas remessas criadas nos **últimos 90 dias**; considera a mais recente com esse AWB |

> Como qualquer valor com hífen é tratado como UUID, um AWB que contenha `-` não é localizado por esta rota.
> Nesse caso, use o UUID da cotação.

Regras dos parâmetros de query:

| Regra | Comportamento |
|---|---|
| Valores booleanos | São considerados verdadeiros: `true`, `1`, `on`, `yes`. **Qualquer outro valor, inclusive vazio** (`attachments=`), é considerado `false` |
| `documents` com valor desconhecido | O valor é ignorado. Se **nenhum** valor for válido, o filtro não é aplicado e todos os documentos são retornados |
| `documents` com letras maiúsculas | Não é reconhecido (os valores diferenciam maiúsculas e minúsculas) |
| `documents` repetido (`documents=a&documents=b`) | Apenas o **último** é considerado. Envie todos em um único parâmetro, separados por vírgula |
| `documents[]=...` | Não suportado: a requisição falha com `500` |

> **Dica de performance:** envie `base64=false` quando for baixar pela `url`. O payload fica muito menor
> e a resposta, bem mais rápida. Lembre-se de que, assim, anexos e documentos incorporados não vêm na resposta.

### Payload mínimo

Endpoint sem corpo. Basta o parâmetro de path (todos os parâmetros de query assumem o padrão):

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs
```

### Payload máximo

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Exemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf,awb_zpl' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

Somente as URLs, sem conteúdo Base64:

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?base64=false' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK` — com `base64=true`, anexos e permissão de documentos incorporados:

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "Invoice": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      },
      "PackingList": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_packing_list.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "Declaration": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_declaration.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "NonDangerousObject": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_non_dangerous_object.pdf",
        "base64": "JVBERi0xLjcK..."
      }
    },
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." },
      { "id": 2, "type": "xml", "description": "XML da Nota Fiscal", "base64": "PD94bWwg..." }
    ],
    "embedded_docs": [
      { "type": "pdf", "base64": "JVBERi0xLjcK..." },
      { "type": "zpl", "base64": "XlhBDQpeTUNZ..." }
    ],
    "carrier": "Contrato Exemplo Expresso",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — com `base64=false`:

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": null
      }
    },
    "attachments": [],
    "embedded_docs": [],
    "carrier": "Contrato Exemplo Expresso",
    "tracking_id": "1234567890123"
  }
}
```

`200 OK` — remessa ainda sem documentos, mas com anexos (`base64=true`):

```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "docs": [],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "JVBERi0xLjcK..." }
    ],
    "embedded_docs": []
  }
}
```

`400 Bad Request` — nenhum documento nem anexo:

```json
{
  "status": "warning",
  "message": "Nenhum documento encontrado com o código especificado!"
}
```

### Campos da resposta

| Campo | Tipo | Presença | Descrição |
|---|---|---|---|
| `status` | string | sempre | `success`, `warning` ou `error` |
| `message` | string | sempre | Mensagem traduzida pelo header `lang` |
| `data.docs` | object \| array | sempre | Mapa `chave do documento → { url, base64 }`. As chaves são as da tabela [Tipos de documento filtráveis](#tipos-de-documento-filtráveis-documents). **Quando não há documentos, vem como lista vazia (`[]`)** |
| `data.docs.*.url` | string | sempre | URL para download do arquivo. Pode ser temporária (veja [Regras de negócio](#regras-de-negócio)) |
| `data.docs.*.base64` | string \| null | sempre | Conteúdo em Base64. `null` quando `base64=false` |
| `data.attachments` | array | sempre | Anexos enviados por você. Lista vazia quando `attachments=false`, `base64=false` ou sem anexos |
| `data.attachments[].id` | integer | - | **ID do anexo.** Use para removê-lo em `DELETE /v1/shipping/{shipping}/docs` |
| `data.attachments[].type` | string | - | Extensão do arquivo: `pdf`, `xml`, `jpg` ou `png` |
| `data.attachments[].description` | string | - | Descrição informada no envio do anexo |
| `data.attachments[].base64` | string | - | Conteúdo em Base64 |
| `data.embedded_docs` | array | sempre | Documentos incorporados. Lista vazia quando não se aplicam (veja [Documentos incorporados](#documentos-incorporados-embedded_docs)) |
| `data.embedded_docs[].type` | string | - | `pdf` ou `zpl` |
| `data.embedded_docs[].base64` | string | - | Conteúdo em Base64 |
| `data.carrier` | string | quando há documentos | Nome do contrato que processou a remessa ou, sem contrato, o nome do transportador |
| `data.tracking_id` | string \| null | quando há documentos | Número de rastreio/AWB |

## Endpoint: consultar documentos por canal

- **Método e rota:** `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}`
- **Quando usar:** quando você só guardou o número do seu próprio pedido (informado na cotação).

### Parâmetros

| Nome | Local | Tipo | Obrigatório | Tamanho | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **sim** | - | - | Identificador do pedido no canal, exatamente como informado na cotação |
| `channel_code` | path | integer | não | - | ausente | Código do canal de venda informado na cotação. Deve ser um número inteiro |
| `attachments` | query | boolean | não | - | `true` | Igual a [consultar documentos por remessa](#endpoint-consultar-documentos-por-remessa) |
| `embedded` | query | boolean | não | - | `true` | Igual a [consultar documentos por remessa](#endpoint-consultar-documentos-por-remessa) |
| `base64` | query | boolean | não | - | `true` | Igual a [consultar documentos por remessa](#endpoint-consultar-documentos-por-remessa) |
| `documents` | query | string | não | - | todos | Igual a [consultar documentos por remessa](#endpoint-consultar-documentos-por-remessa) |

> **Como a busca funciona:**
>
> - **Com `channel_code`:** busca a cotação mais recente com esse identificador **e** esse código de canal.
> - **Sem `channel_code`:** busca a cotação mais recente com esse identificador, **com ou sem** código de canal.
>   Se você usa o mesmo identificador em canais diferentes, envie sempre o `channel_code`.
> - Diferente da [consulta de status por canal](api-rastreio.md#endpoint-consultar-status-por-canal), esta busca
>   **também considera remessas canceladas**.
> - Para omitir o `channel_code`, basta não enviar o segmento. O valor `,` também é tratado como ausente.

### Payload mínimo

```
GET /v1/shipping/docs/PED-2026-0001
```

### Payload máximo

```
GET /v1/shipping/docs/PED-2026-0001/1?attachments=true&embedded=true&base64=true&documents=awb_pdf,awb_zpl,awb_prc_pdf,awb_prc_zpl,awb_prc_fedex_pdf,awb_prc_fedex_zpl,invoice_pdf,packing_list_pdf,declaration_pdf,non_dangerous_pdf
```

### Exemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/docs/PED-2026-0001/1?base64=false' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK` — mesma estrutura de [consultar documentos por remessa](#endpoint-consultar-documentos-por-remessa).

`422 Unprocessable Entity` — `channel_code` não numérico:

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "channel_code": [
      "O campo channel code deve conter um número inteiro."
    ]
  }
}
```

### Campos da resposta

Idênticos aos de [consultar documentos por remessa](#campos-da-resposta).

## Regras de negócio
- Os documentos só existem **após** o processamento no transportador (grupo `shipped` no rastreio).
- **Ordem de decisão da resposta:**

  | Documentos encontrados (após o filtro) | Anexos (`attachments=true` e `base64=true`) | Resposta |
  |---|---|---|
  | Sim | Sim ou não | `200` com `docs`, `attachments`, `embedded_docs`, `carrier` e `tracking_id` |
  | Não | Sim | `200` com `docs: []`, `attachments` e `embedded_docs: []` (sem `carrier` e `tracking_id`) |
  | Não | Não | `400` `Nenhum documento encontrado com o código especificado!` |

- O filtro `documents` também vale para a decisão acima: se os tipos filtrados ainda não existem e não há
  anexos, a resposta é `400`.
- Documentos cujo arquivo não está disponível são omitidos em silêncio. Se nenhum sobrar e não houver
  anexos, a resposta é `400`.
- `embedded_docs` depende de **permissão específica** na sua conta. Sem ela, a lista vem vazia, mesmo com
  `embedded=true`.
- **Validade das URLs:** quando a sua conta está configurada para proteger os links dos documentos, cada `url`
  é **temporária e vale 5 minutos**. Nos demais casos o link é público, mas também pode ser temporário.
  **Baixe e armazene** os arquivos do seu lado logo após a consulta; para baixar de novo depois, consulte o
  endpoint novamente.
- Somente documentos de remessas da sua conta são retornados. Remessas de outra conta respondem
  `400 Nenhum documento encontrado com o código especificado!` (por UUID e por canal) ou
  `404 Esta remessa não foi encontrada!` (por AWB e por NF-e).
- A busca por AWB considera apenas remessas criadas nos **últimos 90 dias**.
- A busca por chave de NF-e desconsidera remessas canceladas.

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `400` | `Nenhum documento encontrado com o código especificado!` | Aguardar o processamento (status `shipped`), revisar o filtro `documents` ou conferir o identificador. Na consulta por canal, também indica que nenhuma remessa foi encontrada |
| `400` | `Remessa não encontrada!` — nenhuma remessa com o UUID informado (consulta por `{shipping}`) | Conferir o UUID da cotação |
| `403` | `Has no permission!` | Solicitar ao suporte a permissão de consulta de documentos |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Conta bloqueada: acionar o suporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Conferir o token enviado em `Authorization` |
| `404` | `Esta remessa não foi encontrada!` — AWB ou chave de NF-e sem remessa correspondente na sua conta (AWB com mais de 90 dias, NF-e de remessa cancelada) | Conferir o AWB/NF-e ou usar o UUID da cotação |
| `422` | `O campo channel code deve conter um número inteiro.` (em `data.channel_code`) | Enviar um inteiro em `channel_code` ou omitir o segmento |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- A consulta por `{shipping}` com o UUID da cotação é a forma mais direta e estável.
- A consulta por canal é a melhor opção quando você quer trabalhar apenas com o número do seu pedido.
- Combine `documents` + `base64=false` para respostas rápidas em integrações de alto volume.
- Para saber quando os documentos ficam disponíveis, veja [Rastreio de Remessa](api-rastreio.md).
- Para enviar e remover anexos, veja [Remessa (v1)](api-remessa-v1.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
