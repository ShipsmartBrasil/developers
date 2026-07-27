# Rascunho

## Visão geral
- Cria, lista, atualiza e remove rascunhos de envio.
- O rascunho é processado assíncronamente e pode evoluir para cotação + remessa.
- Associado ao fluxo operacional principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- O rascunho aceita payloads parciais.
- `status` do rascunho pode ser `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` ou `error`.

## Fluxo de uso
1. Envie o rascunho com os dados que já possui.
2. Aguarde o processamento assíncrono.
3. Liste ou consulte os rascunhos.
4. Atualize apenas rascunhos em status `incomplete`.
5. Remova rascunhos quando necessário.

## Endpoints
### Criar rascunho
- Método e rota: `POST /v1/draft`

### Atualizar rascunho
- Método e rota: `PUT /v1/draft/{code}`
- Só aceita edição de rascunho `incomplete`.

### Listar rascunhos
- Método e rota: `GET /v1/draft`
- Pode filtrar por `drafts.drafts_ids`.

### Remover rascunho
- Método e rota: `DELETE /v1/draft`

### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | não | Canal | `1` | - |
| `channel_identifier_number` | body | string | não | Identificador comercial | `334455` | - |
| `carrier_code` | body | integer | não | Transportador preferido | `1` | - |
| `object` | body | string | não | Objeto | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | não | Tipo | `Commercial` | - |
| `tax` | body | string | não | Imposto | `sender` | - |
| `insurance` | body | boolean | não | Seguro | `true` | - |
| `measurement` | body | string | não | Medidas | `metric` | Default pode vir de preset |
| `currency_quote` | body | string | não | Moeda da cotação | `USD` | - |
| `currency_payment` | body | string | não | Moeda do pagamento | `BRL` | - |
| `address_sender` | body | object | não | Remetente | - | Aceita preenchimento parcial |
| `address_receiver` | body | object | não | Destinatário | - | Aceita preenchimento parcial |
| `boxes` | body | array | não | Caixas | - | Opcional |
| `items` | body | array | não | Itens | - | Opcional |
| `payment.type` | body | string | não | Tipo de pagamento | `billed` | `billed`, `afterward` |

### Exemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP", "city": "Diadema"},
    "address_receiver": {"country_code": "US", "city": "Florida"},
    "boxes": [{"height": 10, "width": 10, "depth": 10}],
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno"
}
```

## Regras de negócio
- Um rascunho sem payload útil retorna warning.
- O backend usa presets de usuário quando disponíveis.
- Rascunhos podem ser convertidos em cotação e remessa pelo processamento assíncrono.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Rascunho vazio, não encontrado ou não editável | Enviar dados válidos ou escolher outro rascunho |
| `422` | Validação | Corrigir payload |
| `500` | Erro inesperado | Acionar o time responsável |

## Observações finais
- O rascunho é a melhor opção quando a integração ainda não tem todos os dados finais.
- Consulte `api-cotacao-v1.md` e `api-remessa-v1.md` para entender o que o processamento assíncrono vai gerar.
