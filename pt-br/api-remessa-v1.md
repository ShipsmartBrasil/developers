# Remessa

- [Voltar ao README do idioma](README.md)

## Visão geral
- Cria, atualiza, cancela e consulta remessas.
- Também expõe consulta de documentos e status/rastreio.
- É a versão legada mais completa do fluxo de shipping.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- A remessa usa `quotation` como vínculo com a cotação.
- O `quotation` precisa ser da mesma versão da API de remessa.
- `payment.type` define o modelo financeiro do envio.

## Fluxo de uso
1. Gere a cotação.
2. Verifique se ela é elegível para remessa.
3. Envie a remessa com o `quotation` retornado.
4. Consulte `status` até a remessa ser processada.
5. Baixe os documentos em `/docs`.
6. Se necessário, agende coleta.

## Regras de negócio
- Cotação `package`/`pallet` com tipo `simple` não é elegível para remessa.
- A cotação não pode estar expirada.
- A remessa não pode ter sido criada anteriormente para a mesma cotação.
- O tipo de pagamento precisa estar habilitado para o usuário.
- Para DHL com remetente PJ no Brasil, `nfkey` pode ser obrigatória.
- Depois de criada, a remessa é irreversível na operação normal.
- Alterações posteriores só são admitidas em casos excepcionais: envio do número da nota fiscal quando ele não foi informado e envio de documentos para anexar à remessa quando eles não foram enviados durante a criação.

## Endpoints
### Criar remessa
- Método e rota: `POST /v1/shipping`
- Quando usar: após obter uma cotação elegível.

#### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sim | ID da cotação | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Precisa ser da primeira versão |
| `carrier_code` | body | integer | sim | Contrato/transportador | `1` | Deve existir para a cotação |
| `address_sender` | body | object | sim | Remetente | - | Pode usar `address_code` ou campos manuais |
| `address_receiver` | body | object | sim | Destinatário | - | Pode usar `address_code` ou campos manuais |
| `payment.type` | body | string | sim | Tipo de pagamento | `billed` | `billed`, `gateway`, `afterward` |
| `type` | body | string | não | Finalidade do envio | `Commercial` | Opcional |
| `nfkey` | body | string | não | Chave NF-e | `12345678901234567890123456789012345678901234` | 44 dígitos |
| `invoice_number` | body | string | não | Número da fatura | `NF-123` | Se não enviado, usa o código interno |
| `shipment_reference` | body | string | não | Referência do envio | `REF-2024-01` | Se não enviado, usa o ID interno |
| `attachments` | body | array | não | Anexos | - | PDF, XML, JPEG, PNG |
| `biller` | body | object | não | Faturante | - | Usado na invoice |

#### Payload mínimo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj"
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "Doe Jhon",
    "email": "doejhon@example.com",
    "phone": "11992494597",
    "type": "pf"
  },
  "payment": {
    "type": "billed"
  }
}
```

#### Payload máximo
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "nfkey": "12345678901234567890123456789012345678901234",
  "type": "Commercial",
  "invoice_number": "NF-123",
  "shipment_reference": "REF-2024-01",
  "carrier_code": 1,
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ],
  "address_sender": {
    "address_code": 1
  },
  "address_receiver": {
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Factory LTDA",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj",
    "foreign": false
  },
  "payment": {
    "type": "gateway"
  }
}
```

#### Exemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "Doe Jhon", "email": "doejhon@example.com", "phone": "11992494597", "type": "pf"},
    "payment": {"type": "billed"}
  }'
```

#### Exemplo de response
```json
{
  "status": "success",
  "message": "Recebemos sua remessa com sucesso! Acompanhe o status da mesma, usando o mesmo código da cotação!"
}
```

### Consultar status da remessa
- Método e rota: `GET /v1/shipping/{shipping}/status`
- `shipping` aceita: UUID da cotação, AWB, ou chave NF-e de 44 dígitos.

### Consultar status por canal
- Método e rota: `GET /v1/shipping/status/{channel_identifier_number}/{channel_code?}`

### Consultar documentos
- Método e rota: `GET /v1/shipping/{shipping}/docs`
- Retorna documentos, anexos e, se permitido, docs incorporados.

### Consultar documentos por canal
- Método e rota: `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code?}`

### Atualizar remessa
- Método e rota: `PUT /v1/shipping/{shipping}`
- Use para anexar documento e/ou vincular NF-e em caso excepcional.
- Os arquivos anexados devem estar em `base64` e aceitar apenas `PDF`, `XML`, `JPEG` ou `PNG`.
- O tamanho máximo é de `5 MB` por arquivo.

#### Exemplo de request
```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "nfkey": "12345678901234567890123456789012345678901234",
    "attachments": [
      {
        "description": "Nota Fiscal",
        "base64": "JVBERi0xLjcK..."
      }
    ]
  }'
```

#### Exemplo de response
```json
{
  "status": "success",
  "message": "Remessa atualizada com sucesso!"
}
```

### Cancelar remessa
- Método e rota: `DELETE /v1/shipping/{shipping}`
- `reason` é opcional no body e aceita texto livre.

#### Exemplo de request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{"reason":"Customer requested cancellation"}'
```

#### Exemplo de response
```json
{
  "status": "success",
  "message": "Requisição processada, dentro de instantes consulte o status para acompanhar o processo de cancelamento"
}
```

### Remover anexos
- Método e rota: `DELETE /v1/shipping/{shipping}/docs`
- Envie `attachments` com `id` ou `name`.

#### Exemplo de request
```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "attachments": [
      {"id": 123456},
      {"name": "Nota Fiscal"}
    ]
  }'
```

#### Exemplo de response
```json
{
  "status": "success",
  "message": "Documentos processados com sucesso!"
}
```

## Consultas e documentos
### Parâmetros de documentos
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `attachments` | query | boolean | não | Inclui anexos | `true` | Default `true` |
| `embedded` | query | boolean | não | Inclui docs incorporados | `true` | Default `true` |
| `base64` | query | boolean | não | Inclui base64 | `true` | Default `true` |
| `documents` | query | string | não | Filtra tipos | `invoice_pdf,awb_pdf` | Lista separada por vírgula |

### Tipos de documentos identificados
- `awb_pdf`
- `awb_zpl`
- `awb_prc_pdf`
- `awb_prc_zpl`
- `awb_prc_fedex_pdf`
- `awb_prc_fedex_zpl`
- `declaration_pdf`
- `invoice_pdf`
- `non_dangerous_pdf`
- `packing_list_pdf`

### Exemplo de consulta de documentos
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/docs?attachments=true&embedded=true&base64=true&documents=invoice_pdf,awb_pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Remessa não encontrada ou regra de negócio | Verificar `quotation`, `carrier_code` e elegibilidade |
| `401` | Pagamento não autorizado | Ajustar a configuração da conta |
| `404` | Recurso não encontrado | Confirmar o identificador informado |
| `422` | Validação | Corrigir payload |
| `500` | Erro inesperado | Acionar o time responsável |

## Observações finais
- A remessa v1 é o ponto de integração mais completo do ciclo operacional.
- Consulte `api-documentos.md` e `api-rastreio.md` para exemplos dedicados de consulta e acompanhamento.
