# Itens

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Mantém o cadastro dos produtos que você envia, com peso, dimensões, HS Code e valores.
- Com o item cadastrado, a cotação pode referenciá-lo apenas pelo `sku`, sem repetir todos os dados.
- É o que habilita a cotação `type: items` e o agrupamento automático `smartbox`.
- Todas as operações de escrita são **assíncronas**.

> ⚠️ **Importante:** `POST`, `PUT` e `DELETE` **não validam os campos de forma síncrona**.
> A API responde `200` confirmando o recebimento e valida cada item durante o processamento.
> Itens inválidos são **descartados silenciosamente** e o motivo fica registrado no relatório de
> importação, no painel. Sempre confirme o resultado com `GET /v1/item`.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
Content-Type: application/json
lang: pt
```

## Por que usar o cadastro de itens

**Sem cadastro** — repetindo tudo em cada cotação:
```json
"items": [
  { "box_id": 1, "name": "Camiseta algodao P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
]
```

**Com cadastro** — referenciando o SKU:
```json
"items": [
  { "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }
]
```

Com `type: items`, você nem precisa indicar a caixa: a plataforma escolhe entre as
[caixas cadastradas](api-caixa-v1.md).

## Fluxo de uso
1. Cadastre os itens com `POST /v1/item`.
2. Aguarde o processamento assíncrono.
3. Confirme com `GET /v1/item`.
4. Use o `sku` nas [cotações](api-cotacao-v1.md).
5. Atualize com `PUT /v1/item` e remova com `DELETE /v1/item`.

## Endpoint: listar itens

- **Método e rota:** `GET /v1/item`

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `items[skus][]` | query | array | não | - | Lista de SKUs. **Vazio ou ausente retorna todos** |
| `page` | query | integer | não | `1` | Página da listagem |

- **Itens por página:** 15

```bash
# Todos os itens
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'

# Filtrando por SKU
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "name": "Tenis",
        "description": "Tenis de corrida masculino 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "TSH-000-S",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "unit_value": 220.25,
        "selling_unit_value": 330.00,
        "hscode": "950450",
        "is_groupable": true,
        "is_foldable": false,
        "box_code": 3,
        "box_identifier": "CX0236"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

## Endpoint: criar itens

- **Método e rota:** `POST /v1/item`
- **Processamento:** assíncrono, em lote.

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `sku` | string | **sim** | - | - | Código único do produto na sua base. **Não pode repetir** |
| `name` | string | **sim** | - | - | Nome do produto (aparece na invoice) |
| `description` | string | **sim** | 6 | 155 | Descrição do produto. **Descrições distintas entre itens evitam alertas aduaneiros** |
| `category` | string | **sim** | 6 | 6 | Categoria do produto. **Exatamente 6 caracteres** |
| `country_code` | string | **sim** | 2 | 2 | País de origem/fabricação em ISO 3166-1 alpha-2 |
| `hscode` | string | **sim** | - | - | NCM/HS Code. Define a tabela de impostos aplicada no destino |
| `height` | number | **sim** | 0.01 | - | Altura do produto |
| `width` | number | **sim** | 0.01 | - | Largura do produto |
| `depth` | number | **sim** | 0.01 | - | Profundidade do produto |
| `weight` | number | **sim** | 0.01 | - | Peso unitário |
| `is_groupable` | boolean | **sim** | - | - | `true` permite agrupar o item com outros na mesma caixa. `false` força caixa exclusiva no `smartbox` |
| `ean` | string | não | - | - | Código EAN/GTIN |
| `unit_value` | number | não | - | - | Valor unitário padrão. Se ausente no cadastro **e** na cotação, a cotação falha |
| `selling_unit_value` | number | não | - | - | Valor de venda unitário. Necessário quando a cotação usa `ior: true` |
| `box_code` | integer | não | - | - | Caixa padrão do item ([Caixa](api-caixa-v1.md)). Precisa ser um **inteiro** |
| `box_identifier` | string | não | - | - | Caixa padrão do item, pelo identificador textual |
| `is_foldable` | boolean | não | - | - | Item dobrável. No `smartbox`, dobráveis são encaixados por volume; não dobráveis por dimensões |

> **Dimensões e peso** seguem o sistema de medida da conta/cotação: cm/kg em `metric`,
> in/lb em `imperial`.

### Payload mínimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao manga curta branca tamanho P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "is_groupable": true
    }
  ]
}
```

### Payload máximo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao manga curta branca tamanho P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3,
      "box_identifier": "CX0236"
    },
    {
      "sku": "SNK-044",
      "name": "Tenis esportivo 44",
      "description": "Tenis de corrida masculino numero 44 cor preta",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 199.9,
      "selling_unit_value": 349.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

### Exemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Camiseta algodao P", "description": "Camiseta 100% algodao manga curta branca tamanho P", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: atualizar itens

- **Método e rota:** `PUT /v1/item`
- **Chave de atualização:** o `sku`. Envie o SKU existente e os campos a alterar.
- **Processamento:** assíncrono, em lote.
- **Estrutura do payload:** idêntica ao `POST`.

> ⚠️ **SKU inexistente não é criado.** A atualização não faz "upsert": se o SKU não existir,
> nada acontece (e nada é reportado na resposta HTTP). Use `POST` para criar.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Camiseta algodao P", "description": "Camiseta premium 100% algodao manga curta branca P", "category": "610910", "country_code": "BR", "hscode": "610910", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

## Endpoint: remover itens

- **Método e rota:** `DELETE /v1/item`
- **Processamento:** assíncrono, em lote.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `items` | object | **sim** | Envelope |
| `items.skus` | array | **sim** | Lista de SKUs a remover |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Validação assíncrona — tabela de referência

Como a validação acontece no processamento, use esta tabela para garantir que os itens entrem.
A primeira regra violada descarta o item.

| Campo | Regra | Motivo registrado no relatório |
|---|---|---|
| (item) | Precisa ser um objeto | `not_array` |
| `name` | Obrigatório, texto | `name` |
| `description` | Obrigatório, texto | `description` |
| `category` | Obrigatório, texto, **exatamente 6 caracteres** | `category` |
| `country_code` | Obrigatório, texto, **exatamente 2 caracteres** | `country_code` |
| `sku` | Obrigatório, texto | `sku` |
| `ean` | Se enviado, precisa ser texto | `ean` |
| `height` | Obrigatório, numérico, **≥ 0.01** | `height` |
| `width` | Obrigatório, numérico, **≥ 0.01** | `width` |
| `depth` | Obrigatório, numérico, **≥ 0.01** | `depth` |
| `weight` | Obrigatório, numérico, **≥ 0.01** | `weight` |
| `hscode` | Obrigatório, texto | `hscode` |
| `is_groupable` | Se enviado, precisa ser booleano | `is_groupable` |
| `box_code` | Se enviado, precisa ser **inteiro** (não string) | `box_code` |
| `box_identifier` | Se enviado, precisa ser texto | `box_identifier` |
| `unit_value` | Se enviado, precisa ser numérico | `unit_value` |
| `selling_unit_value` | Se enviado, precisa ser numérico | `selling_unit_value` |

Outras situações registradas no relatório:

| Situação | Comportamento |
|---|---|
| `items` vazio ou não é array | Relatório: `Itens isn't an array or is empty` |
| SKU duplicado no cadastro | O registro é rejeitado (o SKU é índice único) |
| SKU inexistente no `PUT` | Nada é alterado e nada é criado |
| SKU inválido no `DELETE` | Relatório: `sku_invalid` |

## Como usar os itens na cotação

| Cotação `type` | Payload em `items[]` | Comportamento |
|---|---|---|
| `advanced` | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Você define em qual caixa o item vai |
| `items` | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | A plataforma escolhe as caixas (agrupamento da conta) |
| qualquer | `{ "box_id": 1, "name": "...", "country_code": "BR", "hscode": "...", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Item avulso, sem cadastro |

Veja a seção `items[]` em [Cotação (v1)](api-cotacao-v1.md#parâmetros--items).

## Regras de negócio
- Todas as operações de escrita rodam em **fila assíncrona**.
- O `sku` é **índice único** por conta.
- Itens são **por conta**: cada usuário vê apenas os seus.
- O cadastro precisa estar completo para a cotação funcionar. Mensagens relacionadas:
  - `Nenhum item foi encontrado com o SKU :sku especificado !`
  - `Por favor verifique o cadastro do item :sku, pois poder haver informações imcompletas!`
  - `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).`
  - `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero...`
- `is_groupable: false` garante caixa exclusiva no agrupamento `smartbox`.
- `hscode` incorreto gera erro de imposto na cotação:
  `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item).`

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `200` com itens ausentes | Item descartado na validação assíncrona | Conferir o relatório no painel e a tabela de validação acima |
| `403` | Sem permissão para a ação | Solicitar a permissão ao suporte |
| `422` | `items` ausente | Enviar o array `items` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Como a validação não é síncrona, **sempre confirme com `GET /v1/item`** após cadastrar ou atualizar.
- Mantenha descrições distintas entre produtos diferentes: descrições repetidas geram o warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` na cotação.
- Para padronizar as embalagens usadas com estes itens, veja [Caixa](api-caixa-v1.md).
