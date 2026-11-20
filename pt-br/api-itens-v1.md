# Itens

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Mantém o cadastro dos produtos que você envia, com peso, dimensões, HS Code, valores e identificadores (SKU, EAN e NSMP).
- Com o item cadastrado, a cotação pode referenciá-lo apenas pelo `sku`, sem repetir todos os dados.
- É o que habilita a [cotação por itens](api-cotacao-v2.md#cotação-de-itens) (`type: items`, **exclusiva da v2**) e os agrupamentos automáticos de caixas da conta.
- Todas as operações de escrita (`POST`, `PUT`, `DELETE`) são **assíncronas** e validadas apenas no processamento.
- A listagem é **paginada (20 por página)** e reflete o cadastro em tempo real (sem cache).

> ⚠️ **Importante:** `POST`, `PUT` e `DELETE` **não validam os campos de forma síncrona**.
> A API responde `200` confirmando o recebimento e valida cada item durante o processamento.
> Itens inválidos são **descartados silenciosamente** (sem erro na resposta HTTP) e o motivo fica registrado no
> relatório de importação de itens, no painel. Sempre confirme o resultado com `GET /v1/item`.

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

| Header | Obrigatório | Valores | Padrão |
|---|---|---|---|
| `Authorization` | **sim** | `Bearer SEU_TOKEN` | - |
| `Content-Type` | sim em `POST`, `PUT` e `DELETE` | `application/json` | - |
| `lang` | não | `pt`, `en`, `es` | `pt` |

Cada operação exige uma permissão específica no seu perfil (listar, cadastrar/atualizar e remover itens).
Sem a permissão, a resposta é `403` com `Has no permission!`.

## Fluxo de uso
1. (Opcional) Cadastre as [caixas](api-caixa-v1.md) e anote o `box_code` para vincular aos itens.
2. Cadastre os itens com `POST /v1/item`.
3. Aguarde o processamento assíncrono e confira o relatório de importação no painel.
4. Confirme com `GET /v1/item` que todos os SKUs entraram.
5. Use o `sku` nas [cotações](api-cotacao-v1.md).
6. Atualize com `PUT /v1/item` e remova com `DELETE /v1/item`.

## Conceitos essenciais

### Por que usar o cadastro de itens

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

Com `type: items` (**exclusivo da [Cotação v2](api-cotacao-v2.md#cotação-de-itens)**), você nem precisa posicionar o item em uma caixa: a plataforma monta as caixas conforme o
agrupamento configurado na conta (veja [Como usar os itens na cotação](#como-usar-os-itens-na-cotação)).

### Identificadores do produto

| Campo | O que é | Uso |
|---|---|---|
| `sku` | Seu código interno do produto | **Chave do cadastro**: único por conta, usado para referenciar, atualizar e remover |
| `ean` | Código de barras EAN/GTIN | Impresso na invoice e enviado como identificador padrão do fabricante nas informações regulatórias |
| `nsmp` | *Non Standard Manufacturer Product Id* — identificador de produto **não padronizado** do fabricante (ex.: o código de peça/modelo do fabricante) | Impresso na invoice (linha `Mfr:`) e enviado nas informações regulatórias exigidas por alguns transportadores para destinos na **União Europeia** |

### Unidades de medida

Dimensões e peso do cadastro são sempre interpretados em **centímetros** e **quilogramas**.
A conversão do sistema `imperial` feita na cotação vale apenas para as medidas enviadas no payload da cotação —
**não** é aplicada aos dados preenchidos a partir do cadastro. Cadastre sempre em cm/kg.

### Tipos de dados (o processamento é estrito)

A validação assíncrona confere o **tipo JSON** de cada campo. Valores com o tipo errado descartam o item:

| Campo | Envie como | Exemplo correto | Exemplo descartado |
|---|---|---|---|
| `sku`, `hscode`, `ean`, `nsmp`, `category` | texto (entre aspas) | `"hscode": "610910"` | `"hscode": 610910` |
| `box_code` | número inteiro | `"box_code": 3` | `"box_code": "3"` ou `3.0` |
| `height`, `width`, `depth`, `weight`, `unit_value`, `selling_unit_value` | número | `"weight": 0.25` | `"weight": "0,25"` |
| `is_groupable`, `is_foldable` | booleano JSON | `"is_groupable": false` | `"is_groupable": "false"` (é gravado, mas tratado como `true` na cotação) |

## Endpoint: listar itens

- **Método e rota:** `GET /v1/item`
- **Quando usar:** para consultar o cadastro, conferir se a importação assíncrona foi concluída e verificar os dados de um SKU.

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `items[skus][]` | query | array de string | não | - | Lista de SKUs (correspondência exata, diferencia maiúsculas/minúsculas). **Vazio ou ausente retorna todos** |
| `page` | query | integer | não | `1` | Página da listagem |

- **Itens por página:** 20 (fixo).
- SKUs inexistentes são ignorados (não geram erro).
- Sem cache: alterações processadas aparecem imediatamente.

### Payload mínimo
Sem parâmetros (retorna todos os itens, página 1):
```
GET /v1/item
```

### Payload máximo
```
GET /v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044&page=1
```

### Exemplo de request

```bash
# Todos os itens
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'

# Filtrando por SKU
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=SNK-044' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

> O `--globoff` evita que o curl interprete os colchetes da query string.

### Exemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "sku": "TSH-000-S",
        "name": "Camiseta algodao P",
        "description": "Camiseta 100% algodao manga curta branca tamanho P",
        "category": "610910",
        "country_code": "BR",
        "hscode": "610910",
        "ean": "7894900010015",
        "nsmp": "FAB-TSH-P-001",
        "height": 2,
        "width": 20,
        "depth": 25,
        "weight": 0.25,
        "is_groupable": true,
        "is_foldable": true,
        "box_code": 3,
        "box_identifier": "CX0236",
        "unit_value": 29.9,
        "selling_unit_value": 49.9
      },
      {
        "sku": "SNK-044",
        "name": "Tenis esportivo 44",
        "description": "Tenis de corrida masculino numero 44 cor preta",
        "category": "640411",
        "country_code": "BR",
        "hscode": "640411",
        "height": 15,
        "width": 22,
        "depth": 33,
        "weight": 0.8,
        "is_groupable": false,
        "nsmp": null,
        "unit_value": null,
        "selling_unit_value": null
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/item?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/item?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/item",
    "per_page": 20,
    "prev_page_url": null,
    "to": 2,
    "total": 2
  }
}
```

### Campos da resposta

**Item (`data.data[]`):**

| Campo | Tipo | Sempre presente | Descrição |
|---|---|---|---|
| `sku` | string | sim | Código único do produto |
| `name` | string | sim | Nome do produto |
| `description` | string | sim | Descrição do produto |
| `category` | string | sim | Categoria (6 caracteres) |
| `country_code` | string | sim | País de origem/fabricação |
| `hscode` | string | sim | NCM/HS Code |
| `height`, `width`, `depth` | number | sim | Dimensões (cm) |
| `weight` | number | sim | Peso unitário (kg) |
| `nsmp` | string \| null | **sim** | Identificador não padronizado do fabricante. Vem `null` quando não cadastrado |
| `unit_value` | number \| null | **sim** | Valor unitário padrão. `null` quando não cadastrado |
| `selling_unit_value` | number \| null | **sim** | Valor de venda unitário. `null` quando não cadastrado |
| `ean` | string | não | Código EAN/GTIN (só aparece se cadastrado) |
| `is_groupable` | boolean | não | Permite agrupar com outros itens (só aparece se cadastrado) |
| `is_foldable` | boolean | não | Item dobrável (só aparece se cadastrado) |
| `box_code` | integer | não | Caixa padrão vinculada (só aparece se cadastrado) |
| `box_identifier` | string | não | Caixa padrão vinculada, pelo identificador (só aparece se cadastrado) |

> Os campos são devolvidos como foram gravados (mesmo tipo e valor enviados). A ordem das chaves pode variar.
> Campos fora da lista documentada que forem enviados no cadastro também são gravados e podem aparecer na listagem:
> **envie apenas os campos documentados**.

**Paginação (`data`):** mesmos campos da [listagem de caixas](api-caixa-v1.md#campos-da-resposta)
(`current_page`, `from`, `to`, `per_page` = 20, `total`, `last_page`, URLs de navegação, `path` e `links`).

## Endpoint: criar itens

- **Método e rota:** `POST /v1/item`
- **Quando usar:** para cadastrar novos produtos, um ou vários por requisição.
- **Processamento:** assíncrono, em lote. Cada item é validado individualmente; os válidos são gravados e os inválidos
  são descartados e listados no relatório de importação.
- **SKU já existente não é atualizado:** use `PUT` para alterar.

### Parâmetros

**Raiz do payload:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `items` | array | **sim** | Lista de itens. Sem `items`, nada é gravado (a resposta continua `200`) |

**Parâmetros — `items[]`:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `sku` | string | **sim** | 1 | 50 (recomendado) | - | Código único do produto na sua conta. **Não pode repetir.** A cotação aceita SKU de até 50 caracteres |
| `name` | string | **sim** | 1 | - | - | Nome do produto (aparece na invoice) |
| `description` | string | **sim** | 1 (6 recomendado) | 155 (recomendado) | - | Descrição do produto. **Descrições distintas entre itens evitam alertas aduaneiros** |
| `category` | string | **sim** | 6 | 6 | - | Categoria do produto. **Exatamente 6 caracteres** (sem acentos: letras acentuadas contam como 2) |
| `country_code` | string | **sim** | 2 | 2 | - | País de origem/fabricação em ISO 3166-1 alpha-2, maiúsculo (ex.: `BR`) |
| `hscode` | string | **sim** | 1 | - | - | NCM/HS Code, **como texto**. Define a tabela de impostos aplicada no destino |
| `height` | number | **sim** | 0.01 | - | - | Altura do produto (cm) |
| `width` | number | **sim** | 0.01 | - | - | Largura do produto (cm) |
| `depth` | number | **sim** | 0.01 | - | - | Profundidade do produto (cm) |
| `weight` | number | **sim** | 0.01 | - | - | Peso unitário (kg) |
| `is_groupable` | boolean | não (**recomendado**) | - | - | `true` | `true` permite agrupar o item com outros na mesma caixa; `false` exige caixa exclusiva. No agrupamento `default`, item sem este campo causa erro na cotação |
| `ean` | string | não | - | 50 (recomendado) | - | Código EAN/GTIN, **como texto** |
| `nsmp` | string | não | - | 50 (recomendado) | `null` | *Non Standard Manufacturer Product Id*: identificador não padronizado do fabricante (ex.: código de modelo/peça). Impresso na invoice e usado nas informações regulatórias de alguns transportadores para destinos na União Europeia. A cotação aceita até 50 caracteres |
| `unit_value` | number | não | - | - | - | Valor unitário padrão. Se ausente no cadastro **e** na cotação, a cotação falha |
| `selling_unit_value` | number | não | - | - | - | Valor de venda unitário. Necessário quando a cotação usa `ior: true` |
| `box_code` | integer | não | - | - | - | Caixa padrão do item ([Caixa](api-caixa-v1.md)). Precisa ser **inteiro** JSON |
| `box_identifier` | string | não | - | 50 | - | Caixa padrão do item, pelo identificador textual |
| `is_foldable` | boolean | não | - | - | `false` | Item dobrável. No `smartbox`, dobráveis são encaixados por volume; não dobráveis, por dimensões |

> **Caixa padrão (Forma A / Forma B):** vincule a caixa por `box_code` (Forma A) **ou** por `box_identifier` (Forma B).
> Não envie os dois com valores de caixas diferentes. A existência da caixa **não** é conferida no cadastro do item —
> ela só é verificada na cotação. Prefira `box_code`: é o vínculo reconhecido na proteção contra remoção de caixas.

> Valores `null` em `unit_value` e `selling_unit_value` são ignorados (o campo não é gravado).

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
      "weight": 0.25
    }
  ]
}
```

> Este é o menor payload aceito. Para usar o item na cotação `type: items` com agrupamento `default`, inclua também
> `is_groupable` e a caixa padrão (`box_code` ou `box_identifier`).

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
      "nsmp": "FAB-TSH-P-001",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Tenis esportivo 44",
      "description": "Tenis de corrida masculino numero 44 cor preta",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": "FAB-SNK-44-BLK",
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
      {"sku": "TSH-000-S", "name": "Camiseta algodao P", "description": "Camiseta 100% algodao manga curta branca tamanho P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-001", "height": 2, "width": 20, "depth": 25, "weight": 0.25, "is_groupable": true, "box_code": 3}
    ]
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` (recebido) ou `error` (falha inesperada) |
| `message` | string | Mensagem traduzida conforme o header `lang` |

> O `200` confirma apenas o recebimento. Itens descartados aparecem no relatório de importação (veja
> [Validação assíncrona](#validação-assíncrona--tabela-de-referência)).

## Endpoint: atualizar itens

- **Método e rota:** `PUT /v1/item`
- **Quando usar:** para alterar dados de produtos já cadastrados, em lote.
- **Chave de atualização:** o `sku`. O item cadastrado com aquele SKU recebe os valores enviados.
- **Processamento:** assíncrono, em lote, com **a mesma validação do `POST`**.

> ⚠️ **Envie o item completo.** A validação do `PUT` é idêntica à do `POST`: `sku`, `name`, `description`, `category`,
> `country_code`, `hscode`, `height`, `width`, `depth` e `weight` são **obrigatórios em toda atualização**. Um item
> enviado só com o SKU e o campo alterado é **descartado** (motivo `name` no relatório).

> ⚠️ **SKU inexistente não é criado.** A atualização não faz "upsert": se o SKU não existir, nada acontece
> (e nada é reportado). Use `POST` para criar.

Regras de atualização dos campos opcionais:

| Situação no payload do `PUT` | Resultado no cadastro |
|---|---|
| Campo opcional **omitido** | Mantém o valor atual |
| Campo opcional enviado com valor | Substitui o valor atual |
| `ean`, `nsmp`, `box_code`, `box_identifier`, `is_foldable` enviados como `null` | O valor é apagado (passa a `null`) |
| `unit_value` ou `selling_unit_value` enviados como `null` | **Ignorado**: o valor atual é mantido (não é possível apagar o preço pelo `PUT`; envie um novo valor) |

### Parâmetros

Mesma estrutura e mesmas regras do [`POST /v1/item`](#endpoint-criar-itens): raiz `items` (array) e os campos de `items[]`.

### Payload mínimo

```json
{
  "items": [
    {
      "sku": "TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta premium 100% algodao manga curta branca P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27
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
      "description": "Camiseta premium 100% algodao manga curta branca P",
      "category": "610910",
      "country_code": "BR",
      "hscode": "610910",
      "ean": "7894900010015",
      "nsmp": "FAB-TSH-P-002",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.27,
      "unit_value": 34.9,
      "selling_unit_value": 54.9,
      "is_groupable": true,
      "is_foldable": true,
      "box_code": 3
    },
    {
      "sku": "SNK-044",
      "name": "Tenis esportivo 44",
      "description": "Tenis de corrida masculino numero 44 cor preta",
      "category": "640411",
      "country_code": "BR",
      "hscode": "640411",
      "ean": "7894900010022",
      "nsmp": null,
      "height": 15,
      "width": 22,
      "depth": 33,
      "weight": 0.8,
      "unit_value": 189.9,
      "selling_unit_value": 339.9,
      "is_groupable": false,
      "is_foldable": false,
      "box_identifier": "CX0237"
    }
  ]
}
```

> No segundo item, `"nsmp": null` apaga o NSMP cadastrado.

### Exemplo de request

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "items": [
      {"sku": "TSH-000-S", "name": "Camiseta algodao P", "description": "Camiseta premium 100% algodao manga curta branca P", "category": "610910", "country_code": "BR", "hscode": "610910", "nsmp": "FAB-TSH-P-002", "height": 2, "width": 20, "depth": 25, "weight": 0.27, "unit_value": 34.9, "is_groupable": true}
    ]
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Recebemos sua solicitação, e em breve ela irá ser processada! Você pode acompanhar a evolução do processamento pelo portal!"
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` (recebido) ou `error` (falha inesperada) |
| `message` | string | Mensagem traduzida conforme o header `lang` |

## Endpoint: remover itens

- **Método e rota:** `DELETE /v1/item`
- **Quando usar:** para excluir definitivamente produtos do cadastro.
- **Processamento:** assíncrono, em lote. SKUs inexistentes são ignorados sem aviso.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Mín. | Descrição |
|---|---|---|---|---|---|
| `items` | body | object | **sim** | - | Envelope |
| `items.skus` | body | array de string | **sim** | 1 item | Lista de SKUs a remover. Cada SKU precisa ser texto não vazio |

> A ausência de `items.skus` não gera erro na resposta HTTP (`200`); o problema é registrado no relatório.

### Payload mínimo

```json
{
  "items": {
    "skus": ["TSH-000-S"]
  }
}
```

### Payload máximo

```json
{
  "items": {
    "skus": ["TSH-000-S", "TSH-000-M", "SNK-044"]
  }
}
```

### Exemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/item \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"items": {"skus": ["TSH-000-S", "TSH-000-M"]}}'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` (recebido) ou `error` (falha inesperada) |
| `message` | string | Mensagem traduzida conforme o header `lang` |

## Validação assíncrona — tabela de referência

Como a validação acontece no processamento (`POST` e `PUT`), use esta tabela para garantir que os itens entrem.
As regras são verificadas **nesta ordem** e a **primeira** regra violada descarta o item.
No relatório de importação, cada item descartado aparece como:

```
sku => TSH-000-S, status => invalid, reason => category
```

| Ordem | Campo | Regra | `reason` no relatório |
|---|---|---|---|
| 1 | (item) | Precisa ser um objeto | `not_array` |
| 2 | `name` | Obrigatório, texto não vazio | `name` |
| 3 | `description` | Obrigatório, texto não vazio | `description` |
| 4 | `category` | Obrigatório, texto, **exatamente 6 caracteres** | `category` |
| 5 | `country_code` | Obrigatório, texto, **exatamente 2 caracteres** | `country_code` |
| 6 | `sku` | Obrigatório, texto não vazio | `sku` |
| 7 | `ean` | Se enviado (não nulo), precisa ser texto | `ean` |
| 8 | `nsmp` | Se enviado (não nulo), precisa ser texto | `nsmp` |
| 9 | `height` | Obrigatório, numérico, **≥ 0.01** | `height` |
| 10 | `width` | Obrigatório, numérico, **≥ 0.01** | `width` |
| 11 | `depth` | Obrigatório, numérico, **≥ 0.01** | `depth` |
| 12 | `weight` | Obrigatório, numérico, **≥ 0.01** | `weight` |
| 13 | `hscode` | Obrigatório, texto não vazio | `hscode` |
| 14 | `box_code` | Se enviado (não nulo), precisa ser **inteiro** (não string, não decimal) | `box_code` |
| 15 | `box_identifier` | Se enviado (não nulo), precisa ser texto | `box_identifier` |
| 16 | `unit_value` | Se enviado (não nulo), precisa ser numérico | `unit_value` |
| 17 | `selling_unit_value` | Se enviado (não nulo), precisa ser numérico | `selling_unit_value` |

Regras aplicadas silenciosamente (sem descarte e sem aviso):

| Situação | Comportamento |
|---|---|
| `is_groupable` com qualquer valor | Aceito e gravado como enviado. Use booleano JSON: `"false"` em texto é tratado como `true` na cotação |
| `is_foldable` com qualquer valor | Não é validado; gravado como enviado |
| `name`, `description`, `sku`, `ean`, `nsmp` acima dos limites recomendados | Gravados, mas podem gerar erro quando o item for usado na cotação/remessa |
| `country_code` com 2 caracteres que não é um país válido | Gravado; o erro aparece apenas na cotação |
| `box_code`/`box_identifier` de caixa inexistente | Gravado; o erro aparece apenas na cotação |
| Campos não documentados | Gravados como enviados |

Outras situações registradas no relatório:

| Operação | Situação | Comportamento |
|---|---|---|
| `POST`/`PUT` | Corpo vazio (`{}`) | Relatório: `Itens isn't an array or is empty` |
| `POST`/`PUT` | `items` ausente com outros campos no corpo, ou `items` não é lista | Processamento falha e é registrado como falha no relatório; nada é gravado |
| `POST`/`PUT` | `items: []` | Nada é gravado e nada é reportado |
| `POST` | SKU já cadastrado, ou repetido no mesmo lote | O item duplicado é rejeitado; os demais itens válidos do lote são gravados e o lote é marcado como falha no relatório |
| `PUT` | SKU inexistente | Nada é alterado, nada é criado e nada é reportado |
| `DELETE` | `items.skus` ausente ou vazio | Relatório: `SKUs isn't an array or is empty` |
| `DELETE` | SKU vazio ou que não é texto (ex.: número) | Relatório: `sku => ..., status => invalid, reason => sku_invalid` |
| `DELETE` | SKU inexistente | Ignorado sem aviso |

## Como usar os itens na cotação

| Cotação `type` | Payload em `items[]` | Comportamento |
|---|---|---|
| `advanced` (v1 e v2) | `{ "box_id": 1, "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | Você define em qual caixa (posição em `boxes[]`) o item vai |
| `items` (**só v2**) | `{ "sku": "TSH-000-S", "quantity": 3, "unit_value": 29.9 }` | A plataforma monta as caixas conforme o agrupamento da conta |
| qualquer | `{ "box_id": 1, "name": "Camiseta algodao P", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }` | Item avulso, sem cadastro |

Quando o item da cotação tem `sku` e falta algum dado (nome, país, HS Code, dimensões, peso, valores ou `nsmp`),
os dados ausentes são completados pelo cadastro. Os campos enviados na cotação têm prioridade sobre o cadastro.

Uso do cadastro conforme o agrupamento da conta (cotação `type: items`, v2):

| Agrupamento | Requisitos do cadastro | `is_groupable` |
|---|---|---|
| `default` | Cada item precisa ter caixa padrão (`box_code` ou `box_identifier`) e `is_groupable` cadastrados | `true`: agrupado com outros itens da mesma caixa padrão. `false`: uma caixa exclusiva por unidade |
| `smartbox` | As caixas candidatas vêm de `boxes[]` no payload da cotação | Ausente = `true`. `false`: caixa exclusiva. `is_foldable` (ausente = `false`) define encaixe por volume ou por dimensões |
| `all_in_one` | Apenas os dados do item | Ausente = `true`. `false`: o item fica fora da consolidação, em caixa própria |

Veja a seção `items[]` em [Cotação (v1)](api-cotacao-v1.md#parâmetros--items).

## Regras de negócio
- Todas as operações de escrita rodam em **segundo plano**; o `200` confirma apenas o recebimento.
- O `sku` é **único** por conta: o `POST` não sobrescreve um SKU existente.
- Itens são **por conta**: cada usuário vê e altera apenas os seus.
- O `PUT` exige o item completo (mesma validação do `POST`) e não cria SKUs novos.
- Dimensões e peso do cadastro são sempre em **cm** e **kg**.
- `nsmp` é opcional; quando preenchido, é impresso na invoice e enviado nas informações regulatórias de alguns
  transportadores para destinos na União Europeia. Sem `nsmp`, esse dado segue vazio.
- O cadastro precisa estar completo para a cotação funcionar. Mensagens relacionadas na cotação:

| Status | Mensagem | Causa |
|---|---|---|
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | SKU não cadastrado e item sem os dados mínimos no payload |
| `404` | `O item com o sku :name não existe!` | SKU não cadastrado na cotação `type: items` |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Item sem caixa padrão válida no agrupamento `default` |
| `400` | `O valor do item está vazio no cadastro. Preencha o valor unitário do item no cadastro ou informe o valor unitário na cotação (sku: :sku).` | Sem `unit_value` no cadastro e na cotação |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | Valor zerado |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | `hscode` incorreto para o destino |

> Os códigos HTTP das mensagens de cotação acima seguem a página [Cotação (v1)](api-cotacao-v1.md#erros-comuns).

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `200` com itens ausentes | Item descartado na validação assíncrona (`reason => campo` no relatório) | Conferir o relatório no painel e a [tabela de validação](#validação-assíncrona--tabela-de-referência) |
| `200` sem efeito no `PUT` | SKU inexistente ou item incompleto | Criar com `POST` ou enviar o item completo |
| `403` | `Has no permission!` — perfil sem permissão para listar, cadastrar/atualizar ou remover itens | Solicitar a permissão ao suporte |
| `422` | `O campo items.skus deve conter um array.` (somente `GET`) | Enviar `items[skus][]` como lista na query string |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Repetir com intervalo crescente; se persistir, acionar o suporte |
| Relatório | `Itens isn't an array or is empty` | Enviar `items` com ao menos um item |
| Relatório | `SKUs isn't an array or is empty` | No `DELETE`, enviar `items.skus` com ao menos um SKU |
| Relatório | `sku => ..., status => invalid, reason => sku_invalid` | No `DELETE`, enviar SKUs como texto não vazio |

> `POST`, `PUT` e `DELETE` **não retornam `422`**: problemas de campos aparecem apenas no relatório de importação.

## Observações finais
- Como a validação não é síncrona, **sempre confirme com `GET /v1/item`** após cadastrar, atualizar ou remover.
- Envie os tipos JSON corretos: `sku`, `hscode`, `ean` e `nsmp` como texto; `box_code` como inteiro; booleanos sem aspas.
- Mantenha descrições distintas entre produtos diferentes: descrições repetidas geram o warning
  `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` na cotação.
- Preencha `nsmp` para produtos enviados à União Europeia sempre que o fabricante tiver um código próprio do produto.
- Para padronizar as embalagens usadas com estes itens, veja [Caixa](api-caixa-v1.md).
- Para usar os itens na cotação, veja [Cotação (v1)](api-cotacao-v1.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
