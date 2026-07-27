# Remessa V2

## Visão geral
- Cria remessas com contrato simplificado.
- A v2 não expõe, na fonte analisada, endpoints de status, documentos, cancelamento ou atualização.
- Versão documentada: v2.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- `quotation` precisa ter sido criada na v2.
- `payment.type` continua definindo o fluxo financeiro.

## Fluxo de uso
1. Crie a cotação na v2.
2. Envie a remessa com o `quotation` retornado.
3. Acompanhe o restante do fluxo usando os endpoints da v1, se aplicável ao contrato legado.

## Endpoints
### Criar remessa
- Método e rota: `POST /v2/shipping`
- Quando usar: novo fluxo de shipping.
- Regras importantes: a cotação precisa ser v2; `payment.type` deve ser permitido para o usuário.

#### Diferenças em relação à v1
- A v2 mantém o endpoint de criação, mas a documentação pública não mostra os demais endpoints do ciclo.
- `attachments` na v2 aceita apenas `PDF` ou `XML` pela fonte analisada.
- O contrato é mais estrito para o uso de campos de endereço.

#### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `quotation` | body | string | sim | ID da cotação | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Precisa ser v2 |
| `carrier_code` | body | integer | sim | Contrato | `1` | Deve existir |
| `address_sender` | body | object | sim | Remetente | - | Aceita campos manuais ou `address_code` |
| `address_receiver` | body | object | sim | Destinatário | - | Aceita campos manuais ou `address_code` |
| `payment.type` | body | string | sim | Tipo de pagamento | `billed` | `billed`, `gateway`, `afterward` |
| `attachments` | body | array | não | Anexos | - | PDF/XML na fonte |
| `biller` | body | object | não | Faturante | - | Opcional |
| `nfkey` | body | string | não | Chave NF-e | - | 44 dígitos |

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
    "name": "John Doe",
    "email": "johndoe@example.com",
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
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
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
    "foreign": true
  },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
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
  },
  "attachments": [
    {
      "description": "Nota Fiscal",
      "base64": "JVBERi0xLjcK..."
    }
  ]
}
```

#### Exemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v2/shipping \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier_code": 1,
    "address_sender": {"street": "Main Street", "name": "Jhondoe", "email": "jhondoe@example.com", "phone": "11989677465", "type": "pj"},
    "address_receiver": {"street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
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

## Payload mínimo e máximo
- O payload mínimo exige apenas os campos indispensáveis para identificação da cotação, transportador, endereços e pagamento.
- O payload máximo inclui NF-e, anexos, faturante e referência comercial.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `422` | Validação | Ajustar contrato e campos obrigatórios |
| `400` | Regra de negócio | Verificar elegibilidade da cotação e do pagamento |
| `500` | Erro inesperado | Acionar o time responsável |

## Diferenças entre v1 e v2
| Endpoint | Principais diferenças | Campos alterados | Campos removidos | Campos adicionados | Impacto legado |
|---|---|---|---|---|---|
| `POST /shipping` | v2 mantém só a criação no contrato publicado | Endereços mais completos no request | Endpoints de status/docs/cancelamento não aparecem na v2 | `attachments` com regra própria | Integração nova tende a ficar mais restrita |

## Observações finais
- A v2 documentada na fonte é enxuta; para suporte operacional de documentos e rastreio, a fonte analisada aponta para a v1.
