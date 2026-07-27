# Rastreio de Remessa

## Visão geral
- Consulta o status operacional e o histórico de uma remessa.
- Mostra recebimento, pagamento, processamento, transporte e cancelamento.
- Associado ao fluxo operacional principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- O campo `shipping` aceita UUID, AWB ou NF-e, conforme o endpoint.
- Os status usam códigos internos como `RCV0001`, `PPY0001`, `SPD0001`, `CCG0001` e `CCD0001`.

## Fluxo de uso
1. Crie a remessa.
2. Consulte o status pelo `shipping` ou pelo canal.
3. Acompanhe `received`, `processing_payment`, `processing_shipment`, `shipped` e, se aplicável, cancelamento.

## Endpoints
### Status por remessa
- Método e rota: `GET /v1/shipping/{shipping}/status`

### Status por canal
- Método e rota: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `shipping` | path | string | sim | Identificador da remessa | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Aceita UUID, AWB e NF-e |
| `channel_identifier_number` | path | string | sim | Número do canal | `334455` | - |
| `channel_code` | path | integer | não | Código do canal | `1` | Opcional |

### Exemplo de request
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response
```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2024-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        }
      ]
    },
    "received": [],
    "processing_payment": [],
    "shipped": []
  }
}
```

## Regras de negócio
- O pagamento precisa existir para a remessa retornar status completo.
- Se o pagamento ainda não foi aprovado, o status pode ficar em `processing_payment`.
- Quando a remessa é faturada e ainda não processada no transportador, pode aparecer `processing_shipment`.
- Quando há cancelamento, a resposta pode incluir `cancelling` ou `cancelled`.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Remessa não encontrada | Verificar identificador |
| `404` | Sem pagamento vinculado | Confirmar processamento da remessa |
| `500` | Erro inesperado | Acionar o time responsável |

## Observações finais
- O rastreio é a forma mais completa de monitorar a vida útil do envio.
- A timeline é derivada dos eventos internos e do retorno do pagamento/transportador.
