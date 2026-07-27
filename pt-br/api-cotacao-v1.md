# Cotação

- [Voltar ao README do idioma](README.md)

## Visão geral
- Gera cotações internacionais com detalhamento de frete, seguro e impostos.
- Use quando precisar simular o envio antes de criar a remessa.
- A cotação é a base para remessa, rastreio e documentos.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Tipo: Bearer token.
- Header: `Authorization: Bearer SEU_TOKEN`
- Header adicional aceito: `lang: pt|en|es`.
- O idioma padrão é `pt`.

## Convenções
- `Content-Type: application/json`
- Datas no formato `Y-m-d H:i:s` nas respostas.
- Moedas em código ISO 4217 com 3 letras.
- `level` controla o detalhamento do resultado: `simple` ou `detailed`.
- Status HTTP comuns: `200`, `400`, `404`, `406`, `422`, `500`.

## Fluxo de uso
1. Envie uma cotação com origem, destino, objeto, tipo e itens/caixas.
2. Receba o `quotation` retornado pela API.
3. Consulte o resultado com `GET /v1/quotation/{quotation}/{level}`.
4. Se a cotação for elegível, use o mesmo `quotation` para criar a remessa.

## Diferenças de contrato
| Endpoint | Diferença principal | Impacto |
|---|---|---|
| `POST /v1/quotation` | Cria a cotação | Fluxo inicial |
| `GET /v1/quotation/{quotation}/{level}` | Consulta resultado | Permite revisar carriers |

## Tópicos funcionais
### Cotação simples, avançada e `items`
- `simple`: calcula por caixa com peso/valor direto. Para `doc`, é o fluxo natural.
- `advanced`: calcula com caixas e itens distribuídos por `box_id`.
- `items`: a validação do backend aceita esse tipo, mas o contrato OpenAPI não o expõe claramente. Validar com o time responsável pela API.

### `package`, `pallet` e `doc`
- `doc`: documento. A cotação simples para documentos usa caixa com peso e preço.
- `package`: pacote/caixa.
- `pallet`: palete. Pode usar `volume` em vez de `height/width/depth`.

### Cotação elegível para remessa
- A cotação precisa existir, não estar expirada, pertencer ao mesmo `api_version` da remessa e ainda não ter sido usada.
- `package`/`pallet` com tipo `simple` não são elegíveis para remessa.
- A remessa também exige contrato compatível e tipo de pagamento permitido.

## Endpoints
### Criar cotação
- Objetivo: calcular frete, seguro e impostos.
- Método e rota: `POST /v1/quotation`
- Quando usar: antes de criar remessa.
- Regras importantes: origem e destino obrigatórios; `currency_quote` e `currency_payment` obrigatórios; `measurement` aceita `metric` ou `imperial`.

#### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `channel_code` | body | integer | não | Canal de venda | `1` | Deve existir no cadastro |
| `channel_identifier_number` | body | string | não | Identificador do canal | `334455` | Livre |
| `object` | body | string | sim | Objeto enviado | `package` | `doc`, `package`, `pallet`, `not_doc` na fonte |
| `type` | body | string | sim | Tipo da cotação | `advanced` | `simple`, `advanced`, `items` na validação |
| `tax` | body | string | sim | Quem paga impostos | `sender` | `sender` ou `receiver` |
| `insurance` | body | boolean | sim | Seguro | `true` | - |
| `measurement` | body | string | não | Sistema de medida | `metric` | `metric` ou `imperial` |
| `residential_delivery` | body | boolean | não | Entrega residencial | `false` | - |
| `non_stackable` | body | boolean | não | Indica não empilhável | `false` | - |
| `ior` | body | boolean | não | Importer of Record | `false` | - |
| `currency_quote` | body | string | sim | Moeda da cotação | `BRL` | ISO 4217 |
| `currency_payment` | body | string | sim | Moeda de pagamento | `BRL` | ISO 4217 |
| `address_sender` | body | object | sim | Origem | - | Pode usar `address_code` ou campos manuais |
| `address_receiver` | body | object | sim | Destino | - | Pode usar `address_code` ou campos manuais |
| `boxes` | body | array | sim | Caixas/pacotes | - | Obrigatório salvo quando `type=items` |
| `items` | body | array | não | Itens da carga | - | Obrigatório em `advanced`/`items` |
| `manufacturer` | body | object | não | Fabricante na invoice | - | Opcional |
| `declared_freight` | body | object | não | Frete declarado | `{"declared":true,"value":0,"currency":"USD"}` | Só quando aplicável |
| `level` | query | string | não | Nível de detalhe | `simple` | `simple` ou `detailed` |

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
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
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
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "measurement": "metric",
  "residential_delivery": false,
  "non_stackable": false,
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema"
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida"
  },
  "boxes": [
    {
      "name": "Caixa 1",
      "height": 20,
      "width": 20,
      "depth": 20
    }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Test Product",
      "description": "Test Product",
      "country_code": "BR",
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
  --url https://api.shipsmart.global/v1/quotation \
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
    "address_sender": {
      "country_code": "BR",
      "state_code": "SP",
      "postal_code": "09941070",
      "city": "Diadema"
    },
    "address_receiver": {
      "country_code": "US",
      "state_code": "FL",
      "postal_code": "33101",
      "city": "Florida"
    },
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
    "description": null,
    "channel": "Domenic Dooley",
    "channel_identifier_number": "334455",
    "object": "package",
    "type": "advanced",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "EUR",
    "carriers": [
      {
        "code": 1,
        "name": "DHL",
        "valid_until": "2021-09-28 20:28:10",
        "transit_days": 5,
        "freight": "181.95",
        "insurance": "11.45",
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
| `400` | Falha de regra de negócio | Ajustar peso, CEP, elegibilidade ou tipo de cotação |
| `406` | Payload inconsistente | Verificar campos obrigatórios e tipos |
| `422` | Validação | Corrigir campos obrigatórios/formatos |
| `500` | Erro inesperado | Repetir e acionar o time responsável |

## Consultar cotação
### Método e rota
- `GET /v1/quotation/{quotation}/{level}`
- `level` obrigatório no path: `simple` ou `detailed`.

### Exemplo
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/quotation/9c06fd75-d5f4-40eb-a1a8-f12086f49944/detailed \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Response resumida
```json
{
  "status": "success",
  "message": "Resultado da cotação solicitada!",
  "data": {
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "finished": false,
    "object": "package",
    "type": "advanced",
    "carriers": []
  }
}
```

## Observações finais
- A resposta pode trazer `warnings` quando houver alertas de cadastro ou de transporte.
- A fonte analisada mostra divergência entre documentação OpenAPI e validação em alguns campos. Validar com o time responsável pela API antes de integrar em produção.
