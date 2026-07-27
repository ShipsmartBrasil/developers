# Documentos de Remessa

- [Voltar ao README do idioma](README.md)

## Visão geral
- Consulta documentos gerados após o processamento da remessa.
- Pode retornar PDF, ZPL, anexos e, quando permitido, documentos incorporados.
- Este módulo está publicado na v1.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- `attachments`, `embedded` e `base64` aceitam booleanos via query string.
- `documents` recebe lista separada por vírgula.

## Fluxo de uso
1. Crie a remessa.
2. Aguarde o status indicar processamento/conclusão.
3. Consulte os documentos por `shipping` ou por canal.
4. Baixe o conteúdo em `url` ou `base64`.

## Endpoints
### Consultar documentos por remessa
- Método e rota: `GET /v1/shipping/{shipping}/docs`
- `shipping` aceita UUID da cotação, AWB ou NF-e.

### Consultar documentos por canal
- Método e rota: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sim | Identificador da remessa | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Também aceita AWB e NF-e |
| `channel_identifier_number` | path | string | sim | Identificador do canal | `334455` | - |
| `channel_code` | path | integer | não | Código do canal | `1` | Opcional |
| `attachments` | query | boolean | não | Inclui anexos | `true` | Default `true` |
| `embedded` | query | boolean | não | Inclui docs incorporados | `true` | Default `true` |
| `base64` | query | boolean | não | Inclui base64 | `true` | Default `true` |
| `documents` | query | string | não | Filtra documentos | `invoice_pdf,awb_pdf` | Quando vazio, retorna tudo |

### Exemplo de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response
```json
{
  "status": "success",
  "message": "Documentos da remessa encontrado!",
  "data": {
    "carrier": "DHL - Teste",
    "tracking_id": "000000",
    "docs": {
      "Invoice": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_invoice.pdf",
        "base64": "string"
      },
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    },
    "embedded_docs": [
      { "type": "pdf", "base64": "string" }
    ],
    "attachments": [
      { "id": 1, "type": "pdf", "description": "Nota Fiscal", "base64": "string" }
    ]
  }
}
```

## Regras de negócio
- Se não houver documentos nem anexos, a API pode retornar `warning`.
- Documentos incorporados dependem de permissão específica.
- Os tipos de documentos filtráveis são os listados na resposta da remessa.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Nenhum documento encontrado | Confirmar se a remessa já foi processada |
| `500` | Erro inesperado | Repetir e acionar o time responsável |

## Observações finais
- Para integrações, a consulta por `shipping` é a forma mais direta.
- A consulta por canal é útil quando o integrador grava o identificador comercial no envio.
