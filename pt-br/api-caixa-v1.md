# Caixa

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Mantém o cadastro das embalagens que você usa com frequência.
- Uma vez cadastrada, a caixa pode ser referenciada na cotação por `box_code` ou `box_identifier`,
  sem repetir dimensões e peso em cada requisição.
- O cadastro e a remoção são **assíncronos**: a API confirma o recebimento e processa em segundo plano.

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

## Por que usar o cadastro de caixas

**Sem cadastro** — repetindo as dimensões em toda cotação:
```json
"boxes": [{ "name": "Caixa P", "height": 20, "width": 20, "depth": 20 }]
```

**Com cadastro** — referenciando a caixa:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Vantagens: payload menor, dimensões padronizadas e uso do agrupamento `smartbox`
(que encaixa os itens nas caixas cadastradas automaticamente).

## Fluxo de uso
1. Cadastre as caixas com `POST /v1/box`.
2. Aguarde alguns instantes (processamento assíncrono).
3. Confirme com `GET /v1/box`.
4. Use `box_code` ou `box_identifier` nas [cotações](api-cotacao-v1.md).
5. Remova as caixas obsoletas com `DELETE /v1/box`.

## Endpoint: listar caixas

- **Método e rota:** `GET /v1/box`

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array | não | - | Lista de `box_code` e/ou `box_identifier`. **Vazio ou ausente retorna todas** |
| `page` | query | integer | não | `1` | Página da listagem |

- **Itens por página:** 15
- A listagem é **cacheada por até 5 minutos**. Uma caixa recém-cadastrada pode levar esse tempo para aparecer.

```bash
# Todas as caixas
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'

# Filtrando por identificadores
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "width": "65",
        "height": "65",
        "depth": "65",
        "weight": "2.8",
        "created_at": "2026-01-27T10:05:05.000000Z",
        "updated_at": "2026-01-27T10:05:05.000000Z",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `box_code` | integer | **Código numérico da caixa.** Use em `boxes[].box_code` na cotação |
| `box_identifier` | string | **Identificador textual da caixa.** Use em `boxes[].box_identifier` na cotação |
| `name` | string | Nome da caixa |
| `height`, `width`, `depth` | number | Dimensões cadastradas |
| `weight` | number | Peso da embalagem vazia |
| `active` | boolean | Caixa ativa. **Caixas inativas não podem ser usadas na cotação** |
| `created_at`, `updated_at` | string | Datas de criação e última alteração |

## Endpoint: criar ou atualizar caixas

- **Método e rota:** `POST /v1/box`
- **Comportamento:** se o `box_identifier` (ou `box_code`) já existir, a caixa é **atualizada**.
  Caso contrário, é **criada**.
- **Processamento:** assíncrono. A resposta `200` confirma apenas o recebimento.

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `boxes` | array | **sim** | 1 item | - | Lista de caixas a criar/atualizar |
| `boxes[].box_identifier` | string | **sim** | - | 155 | Identificador único da caixa na sua base (ex.: `CX0236`) |
| `boxes[].name` | string | **sim** | - | 155 | Nome descritivo |
| `boxes[].height` | number | **sim** | 1 | - | Altura |
| `boxes[].width` | number | **sim** | 1 | - | Largura |
| `boxes[].depth` | number | **sim** | 1 | - | Profundidade |
| `boxes[].weight` | number | **sim** | - | - | Peso da embalagem vazia. Até 11 dígitos inteiros e **3 casas decimais** |
| `boxes[].activated` | boolean | **sim** | - | - | `true` para disponibilizar a caixa nas cotações |
| `boxes[].box_code` | string | não | - | 50 | Código da caixa. Informe para atualizar uma caixa existente pelo código |

> As dimensões seguem o sistema de medida da sua conta/cotação: **centímetros** em `metric`,
> **polegadas** em `imperial`. O peso é em **quilogramas** (`metric`) ou **libras** (`imperial`).

### Payload mínimo

```json
{
  "boxes": [
    {
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    }
  ]
}
```

### Payload máximo

```json
{
  "boxes": [
    {
      "box_code": "1",
      "box_identifier": "CX0236",
      "name": "Cartucho Individual",
      "height": 65,
      "width": 65,
      "depth": 65,
      "weight": 2.8,
      "activated": true
    },
    {
      "box_identifier": "CX0237",
      "name": "Caixa Master",
      "height": 40,
      "width": 30,
      "depth": 25,
      "weight": 0.85,
      "activated": true
    },
    {
      "box_identifier": "CX0100",
      "name": "Caixa descontinuada",
      "height": 10,
      "width": 10,
      "depth": 10,
      "weight": 0.2,
      "activated": false
    }
  ]
}
```

### Exemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "boxes": [
      {"box_identifier": "CX0236", "name": "Cartucho Individual", "height": 65, "width": 65, "depth": 65, "weight": 2.8, "activated": true}
    ]
  }'
```

```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

> **O processamento é assíncrono.** Confirme o resultado com `GET /v1/box` (lembrando do cache de 5 minutos)
> ou acompanhe o relatório de importação no painel.

## Endpoint: remover caixas

- **Método e rota:** `DELETE /v1/box`
- **Processamento:** assíncrono. Ao final, um relatório por e-mail informa quantas caixas foram removidas
  e quais falharam.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `boxes` | object | **sim** | Envelope |
| `boxes.box_ids` | array | **sim** | Lista de `box_code` e/ou `box_identifier` a remover |

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": ["CX0236", 1]}}'
```

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Regras da remoção

| Regra | Resultado quando falha |
|---|---|
| A caixa precisa existir na sua base | Erro no relatório: `box %s not found!` |
| A caixa **não pode ter itens vinculados** | Erro no relatório: `box %s cannot be deleted, has related item` |

> Para "desativar" uma caixa que já tem itens vinculados, use `POST /v1/box` com `activated: false`
> em vez de removê-la.

## Como usar as caixas na cotação

| Forma | Payload em `boxes[]` | Quando usar |
|---|---|---|
| Por código | `{ "box_code": 1 }` | Você guardou o código numérico |
| Por identificador | `{ "box_identifier": "CX0236" }` | Você usa o seu próprio código de embalagem |
| Manual | `{ "height": 20, "width": 20, "depth": 20 }` | Embalagem avulsa, sem cadastro |
| Kit | `{ "kit": "KIT-001" }` | Caixa **e** itens vêm do cadastro de kit |

Veja a seção `boxes[]` em [Cotação (v1)](api-cotacao-v1.md#parâmetros--boxes).

## Regras de negócio
- O cadastro e a remoção são processados em **fila assíncrona**.
- A listagem é **cacheada por 5 minutos** por usuário e página.
- Caixas com `activated: false` não aparecem como disponíveis nas cotações
  (`A caixa :name não existe ou está desativada!`).
- Caixas são **por conta**: cada usuário vê apenas as suas.
- `box_identifier` deve ser único na sua base — é o que permite atualizar em vez de duplicar.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Regra de negócio no processamento | Ler `message` e revisar o payload |
| `403` | Sem permissão para a ação | Solicitar a permissão ao suporte |
| `422` | Validação de campos | Corrigir os campos listados em `data` (dimensões ≥ 1, `box_identifier` obrigatório, `activated` booleano) |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Cadastrar as caixas é o pré-requisito para usar o agrupamento `smartbox` e o tipo de cotação `items`.
- Mantenha os `box_identifier` alinhados com o seu ERP/WMS: isso simplifica muito a integração.
- Para cadastrar os produtos que vão dentro das caixas, veja [Itens](api-itens-v1.md).
