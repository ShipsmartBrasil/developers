# Cotação V2

## Visão geral
- Gera cotações com contrato simplificado para integrações novas.
- Use quando você quiser apenas criar a cotação na v2.
- A consulta detalhada do resultado não existe nesta versão na fonte analisada.
- Versão documentada: v2.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang: pt|en|es`.

## Convenções
- `Content-Type: application/json`
- `level` existe no request body/query da criação e controla o retorno do cálculo.
- HTTP mais comuns: `200`, `400`, `422`, `500`.

## Fluxo de uso
1. Envie a cotação na v2.
2. Receba o `quotation` e use esse código para criar a remessa na v2.
3. Não há endpoint de consulta de cotação na v2 na fonte analisada.

## Endpoints
### Criar cotação
- Método e rota: `POST /v2/quotation`
- Quando usar: novo fluxo de precificação.
- Regras importantes: `object`, `type`, `tax`, `insurance`, `currency_quote`, `currency_payment`, `address_sender`, `address_receiver`, `boxes` e/ou `items` conforme o tipo.

#### Diferenças funcionais em relação à v1
- A v2 documenta `items` como tipo de cotação explicitamente.
- A v2 simplifica os campos de endereço no contrato OpenAPI, mas a validação continua exigindo os dados necessários conforme a origem/destino.
- O resultado detalhado é entregue no próprio `POST`.

#### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `object` | body | string | sim | Objeto enviado | `package` | `doc`, `package`, `pallet`, `not_doc` na fonte |
| `type` | body | string | sim | Tipo da cotação | `items` | `simple`, `advanced`, `items` |
| `tax` | body | string | sim | Imposto | `sender` | `sender` ou `receiver` |
| `insurance` | body | boolean | sim | Seguro | `true` | - |
| `currency_quote` | body | string | sim | Moeda de cotação | `USD` | ISO 4217 |
| `currency_payment` | body | string | sim | Moeda de pagamento | `EUR` | ISO 4217 |
| `address_sender` | body | object | sim | Remetente | - | `address_code` ou manual |
| `address_receiver` | body | object | sim | Destinatário | - | `address_code` ou manual |
| `boxes` | body | array | sim | Caixas/paletes | - | Obrigatório salvo quando `type=items` |
| `items` | body | array | sim/condicional | Itens | - | Regras dependem do tipo |
| `manufacturer` | body | object | não | Fabricante | - | Opcional |
| `declared_freight` | body | object | não | Frete declarado | - | Opcional |

#### Payload mínimo
```json
{
  "object": "doc",
  "type": "simple",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "BRL",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ]
}
```

#### Payload máximo
```json
{
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "object": "package",
  "type": "items",
  "tax": "sender",
  "insurance": true,
  "measurement": "imperial",
  "residential_delivery": false,
  "non_stackable": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL"
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 8, "width": 8, "depth": 8 },
    { "box_code": 3, "height": 10, "width": 10, "depth": 10 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100
    }
  ],
  "manufacturer": {
    "name": "Factory LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "email": "factory@example.com",
    "phone": "11999999999",
    "type": "pj"
  }
}
```

#### Exemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/quotation \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "simple",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "BRL",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "state_code": "SP"},
    "address_receiver": {"country_code": "US", "state_code": "FL"},
    "boxes": [{"height": 10, "width": 10, "depth": 10, "weight": 1, "price": 100}]
  }'
```

#### Exemplo de response
```json
{
  "status": "success",
  "message": "Cotação Realizada com sucesso!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "date": "2021-09-28 14:28:10",
    "object": "package",
    "type": "advanced",
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "freight": "181.95",
        "tax": "28153.894",
        "currency_payment_amount": "24288.1"
      }
    ]
  }
}
```

#### Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Transportador indisponível ou regra negada | Ajustar país, peso, caixa ou contrato |
| `422` | Validação | Corrigir campos obrigatórios |
| `406` | Estrutura incompatível | Revisar payload e versão |
| `500` | Erro inesperado | Acionar o time responsável |

## Observações finais
- A documentação OpenAPI da v2 e a validação de request não são idênticas em todos os detalhes. Validar com o time responsável pela API.
- Para consulta detalhada da cotação, a fonte analisada não traz endpoint v2 equivalente ao `GET /v1/quotation/{quotation}/{level}`.
- Todas as ações de cotação e remessa na segunda versão podem ser acompanhadas pelas outras APIs da primeira versão.
