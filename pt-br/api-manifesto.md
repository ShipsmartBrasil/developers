# Manifesto

## Visão geral
- Executa o fluxo síncrono de cotação + remessa.
- Após a criação, pode retornar os documentos finais da operação.
- Associado ao fluxo operacional principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- Query opcional `format`: `pdf` ou `zpl`.
- O fluxo usa `billed` como tipo de pagamento no contrato publicado.

## Fluxo de uso
1. Envie os dados completos do manifesto.
2. A API faz cotação, cria a remessa e gera documentos.
3. Se `format` for `pdf` ou `zpl`, a resposta favorece esse formato.

## Endpoints
### Criar manifesto
- Método e rota: `POST /v1/manifest`
- Query params: `format`.

### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `format` | query | string | não | Tipo de retorno | `pdf` | `pdf` ou `zpl` |
| `object` | body | string | sim | Objeto | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | body | string | sim | Finalidade | `Commercial` | Vem do fluxo de negócio |
| `tax` | body | string | sim | Imposto | `sender` | `sender` ou `receiver` |
| `insurance` | body | boolean | sim | Seguro | `true` | - |
| `residential_delivery` | body | boolean | sim | Entrega residencial | `false` | - |
| `non_stackable` | body | boolean | sim | Não empilhável | `false` | - |
| `measurement` | body | string | sim | Sistema de medida | `metric` | `metric` ou `imperial` |
| `ior` | body | boolean | sim | IOR | `false` | - |
| `currency_quote` | body | string | sim | Moeda da cotação | `USD` | - |
| `currency_payment` | body | string | sim | Moeda de pagamento | `EUR` | - |
| `address_sender` | body | object | sim | Remetente | - | Manual ou `address_code` |
| `address_receiver` | body | object | sim | Destinatário | - | Manual ou `address_code` |
| `boxes` | body | array | sim | Caixas | - | Obrigatório |
| `items` | body | array | sim | Itens | - | Obrigatório exceto `doc` |
| `payment.type` | body | string | sim | Pagamento | `billed` | Manifesto só aceita faturado |
| `declared_freight` | body | object | não | Frete declarado | - | Opcional |

### Payload mínimo
```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Main Street",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Florida",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "foreign": true
  },
  "boxes": [
    {
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 1,
      "price": 100
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Payload máximo
```json
{
  "format": "pdf",
  "channel_code": 1,
  "channel_identifier_number": "334455",
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "EUR",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Manifesto de teste",
  "declared_freight": {
    "declared": true,
    "value": 0,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Warehouse",
    "street": "Main Street",
    "number": "33",
    "district": "Downtown",
    "complement": "Apto 3",
    "name": "Jhondoe",
    "email": "jhondoe@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678901",
    "state_tax_id": "1234567890",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Florida",
    "description": "Office",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "type": "pf",
    "federal_tax_id": "7418529630",
    "state_tax_id": "7418529586",
    "foreign": true
  },
  "boxes": [
    { "box_identifier": "CX0236", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "name": "Test Product",
      "country_code": "US",
      "hscode": "010190",
      "weight": 1,
      "quantity": 5,
      "unit_value": 100,
      "selling_unit_value": 110
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

### Exemplo de response
```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "000000",
    "docs": {
      "AWB": {
        "url": "https://example.s3.amazonaws.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "string"
      }
    }
  }
}
```

## Regras de negócio
- O manifesto só funciona com pagamento faturado.
- O documento final pode ser PDF, ZPL ou ambos, conforme o fluxo e o transportador.
- Se a cotação for incompatível com a versão, a execução falha.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Erro do transportador ou regra de negócio | Ajustar payload ou verificar retorno do transportador |
| `422` | Validação | Corrigir campos obrigatórios |
| `500` | Erro inesperado | Acionar o time responsável |

## Observações finais
- Este endpoint é síncrono e concentra mais responsabilidade do que os fluxos separados de cotação + remessa.
- Recomendado apenas para integrações que realmente precisam do fluxo completo em uma chamada.
