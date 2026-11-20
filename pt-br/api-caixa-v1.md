# Caixa

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Mantém o cadastro das embalagens que você usa com frequência (nome, dimensões e peso da embalagem vazia).
- Uma vez cadastrada, a caixa pode ser referenciada na cotação por `box_code` ou `box_identifier`,
  sem repetir dimensões e peso em cada requisição.
- A caixa também pode ser vinculada a um [item cadastrado](api-itens-v1.md) como caixa padrão do produto.
- O cadastro/atualização e a remoção são **assíncronos**: a API valida o formato, confirma o recebimento e processa em segundo plano.
- A listagem é **paginada (15 por página)** e mantida em cache por até **5 minutos**.

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
| `Content-Type` | sim em `POST` e `DELETE` | `application/json` | - |
| `lang` | não | `pt`, `en`, `es` | `pt` |

Cada operação exige uma permissão específica no seu perfil (listar, cadastrar e remover caixas).
Sem a permissão, a resposta é `403` com `Has no permission!`.

## Fluxo de uso
1. Cadastre as caixas com `POST /v1/box`.
2. Aguarde alguns instantes (processamento assíncrono). Acompanhe o resultado pelo relatório de importação no painel.
3. Confirme com `GET /v1/box` e anote o `box_code` gerado (lembre do cache de até 5 minutos).
4. Use `box_code` ou `box_identifier` nas [cotações](api-cotacao-v1.md) e no cadastro de [itens](api-itens-v1.md).
5. Remova as caixas obsoletas com `DELETE /v1/box` (preferencialmente pelo `box_code`).

## Conceitos essenciais

### Por que usar o cadastro de caixas

**Sem cadastro** — repetindo as dimensões em toda cotação:
```json
"boxes": [{ "name": "Caixa P", "height": 20, "width": 20, "depth": 20, "weight": 0.3 }]
```

**Com cadastro** — referenciando a caixa:
```json
"boxes": [{ "box_identifier": "CX0236" }]
```

Vantagens: payload menor, dimensões padronizadas e possibilidade de vincular cada item cadastrado a uma
caixa padrão (usado no agrupamento `default` da cotação `type: items`, exclusiva da v2).

### `box_code` x `box_identifier`

| Campo | Quem define | Tipo na listagem | Uso |
|---|---|---|---|
| `box_code` | **A plataforma** (gerado no cadastro) | integer | Código numérico da caixa. Forma mais confiável de referenciar a caixa em cotação, item e remoção |
| `box_identifier` | **Você** (no `POST`) | string | Seu próprio código de embalagem (ex.: `CX0236`). Deve ser único na sua conta |

> **Evite `box_identifier` composto apenas por números** (ex.: `"123"`). Nos filtros da listagem e na remoção,
> qualquer valor numérico é interpretado como `box_code`. Prefira identificadores com letras, como `CX0123`.

### Unidades de medida

As dimensões e o peso das caixas cadastradas são sempre interpretados em **centímetros** e **quilogramas**.
A conversão do sistema `imperial` (polegadas/libras) feita na cotação vale apenas para as medidas enviadas
no próprio payload da cotação — **não** é aplicada aos dados vindos do cadastro. Cadastre sempre em cm/kg.

## Endpoint: listar caixas

- **Método e rota:** `GET /v1/box`
- **Quando usar:** para consultar as caixas cadastradas, descobrir o `box_code` gerado e conferir se o cadastro assíncrono foi concluído.

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|---|
| `boxes[box_ids][]` | query | array de string/integer | não | - | Lista de `box_code` e/ou `box_identifier` (pode misturar). **Vazio ou ausente retorna todas** |
| `page` | query | integer | não | `1` | Página da listagem |

- **Itens por página:** 15 (fixo).
- Valores numéricos em `boxes[box_ids][]` são buscados como `box_code`; os demais, como `box_identifier`.
- Identificadores inexistentes são simplesmente ignorados (não geram erro).

> ⚠️ **Cache de 5 minutos:** o resultado fica em cache por até 5 minutos para cada combinação de conta e filtro.
> Nesse intervalo:
> - caixas recém-cadastradas, alteradas ou removidas podem não refletir na listagem;
> - **requisições para páginas diferentes com o mesmo filtro podem retornar a mesma página já consultada**
>   (o número da página não diferencia o cache). Se você tem mais de 15 caixas, prefira consultar por
>   `boxes[box_ids][]` ou aguarde a expiração do cache entre uma página e outra.

### Payload mínimo
Sem parâmetros (retorna todas as caixas, página 1):
```
GET /v1/box
```

### Payload máximo
```
GET /v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1&page=1
```

### Exemplo de request

```bash
# Todas as caixas
curl --request GET \
  --url 'https://api.shipsmart.global/v1/box' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'

# Filtrando por identificador e por codigo
curl --request GET \
  --globoff \
  --url 'https://api.shipsmart.global/v1/box?boxes[box_ids][]=CX0236&boxes[box_ids][]=1' \
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
        "box_code": 1,
        "box_identifier": "CX0236",
        "name": "Cartucho Individual",
        "height": 65,
        "width": 65,
        "depth": 65,
        "weight": 2.8,
        "created_at": "2026-01-27 10:05:05",
        "updated_at": "2026-01-27 10:05:05",
        "active": true
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/box?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/box?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/box",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos da resposta

**Caixa (`data.data[]`):**

| Campo | Tipo | Descrição |
|---|---|---|
| `box_code` | integer | **Código numérico da caixa**, gerado pela plataforma. Use em `boxes[].box_code` na cotação e em `box_code` do item |
| `box_identifier` | string \| null | **Seu identificador textual da caixa**. Use em `boxes[].box_identifier` na cotação. Pode vir `null` em caixas criadas sem identificador pelo painel |
| `name` | string | Nome da caixa |
| `height`, `width`, `depth` | number | Dimensões cadastradas (cm) |
| `weight` | number | Peso da embalagem vazia (kg) |
| `created_at`, `updated_at` | string | Datas de criação e última alteração, no formato `AAAA-MM-DD HH:MM:SS` |
| `active` | boolean | Indica se a caixa está ativa. Trate também `1`/`0` como verdadeiro/falso |

**Paginação (`data`):**

| Campo | Tipo | Descrição |
|---|---|---|
| `current_page` | integer | Página atual |
| `from`, `to` | integer \| null | Posição do primeiro e do último registro da página |
| `per_page` | integer | Registros por página (15) |
| `total` | integer | Total de registros encontrados |
| `last_page` | integer | Número da última página |
| `first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url` | string \| null | URLs de navegação (mantêm os filtros da query) |
| `path` | string | URL base da listagem |
| `links` | array | Links de navegação prontos (`url`, `label`, `active`) |

## Endpoint: criar ou atualizar caixas

- **Método e rota:** `POST /v1/box`
- **Quando usar:** para cadastrar novas caixas ou alterar caixas existentes, em lote.
- **Comportamento:** para **atualizar** uma caixa existente, informe o `box_identifier` (ou o `box_code`) dela.
  Caso a caixa não exista, ela é **criada**.
- **Processamento:** a API valida o formato de todos os campos (erro `422` se algo estiver errado) e, se estiver tudo certo,
  responde `200` confirmando **apenas o recebimento**. A gravação acontece em segundo plano e o resultado fica no
  relatório de importação de caixas, no painel.

### Parâmetros

**Raiz do payload:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `boxes` | array | **sim** | 1 item | - | - | Lista de caixas a criar/atualizar. Array vazio é rejeitado |

**Parâmetros — `boxes[]`:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `box_identifier` | string | **sim** | - | 155 (use até **50**) | - | Seu identificador único da caixa (ex.: `CX0236`). A validação aceita até 155 caracteres, mas identificadores com mais de 50 caracteres **não são gravados corretamente** |
| `name` | string | **sim** | - | 155 | - | Nome descritivo da caixa |
| `height` | number | **sim** | 1 | - | - | Altura em cm |
| `width` | number | **sim** | 1 | - | - | Largura em cm |
| `depth` | number | **sim** | 1 | - | - | Profundidade em cm |
| `weight` | number | **sim** | - | 11 dígitos inteiros + 3 decimais | - | Peso da embalagem vazia em kg. Formato aceito: até 11 dígitos inteiros e até **3 casas decimais**, separador `.`, sem sinal (ex.: `0.85`, `2.8`, `12`). Informe valor maior que zero |
| `activated` | boolean | **sim** | - | - | - | `true` para caixa ativa, `false` para inativa. Aceita também `1`/`0` |
| `box_code` | **string** | não | - | 50 | - | Código da caixa já existente, para atualizá-la pelo código. **Envie como texto** (ex.: `"1"`): enviado como número, é rejeitado com `422` |

> **Atenção ao tipo de `box_code` no `POST`:** aqui ele é **string** (`"1"`). Na cotação, no cadastro de itens e na
> remoção, ele é **integer** (`1`).

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

### Exemplo de response

**Sucesso (`200`):**
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

**Erro de validação (`422`):**
```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "boxes.0.height": ["O campo boxes.0.height deve conter um número superior ou igual a 1."],
    "boxes.0.box_code": ["O campo boxes.0.box code deve ser uma string."]
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success`, `warning` (validação/regra) ou `error` (inesperado) |
| `message` | string | Mensagem traduzida conforme o header `lang` |
| `data` | object | Só em erros de validação: mapa `campo` → lista de mensagens. O campo vem com o índice da caixa (`boxes.0.*` = primeira caixa) |

> **O `200` não garante que a caixa foi gravada.** Confirme o resultado com `GET /v1/box` (lembrando do cache de
> até 5 minutos) ou pelo relatório de importação no painel.

## Endpoint: remover caixas

- **Método e rota:** `DELETE /v1/box`
- **Quando usar:** para excluir definitivamente caixas que não são mais usadas.
- **Processamento:** a API valida o formato e responde `200`; a remoção é feita em segundo plano.
  Ao final, você recebe **um relatório por e-mail** (no e-mail da conta) com o total de caixas processadas,
  o total removido e a lista de erros.
- A remoção é **definitiva**. Para apenas desativar, use `POST /v1/box` com `activated: false`.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Mín. | Descrição |
|---|---|---|---|---|---|
| `boxes` | body | object | **sim** | - | Envelope |
| `boxes.box_ids` | body | array de integer/string | **sim** | 1 item | Lista de caixas a remover. **Use o `box_code` como número inteiro** (ex.: `1`) |

> ⚠️ **Remova pelo `box_code` (inteiro).** Embora a documentação interativa aceite `box_identifier` nesta lista,
> a remoção por identificador textual **não é confiável** no momento:
> - a caixa pode ser reportada como `box CX0236 not found!` mesmo existindo;
> - a checagem de itens vinculados só reconhece o vínculo feito por `box_code` — uma caixa vinculada a itens apenas
>   por `box_identifier` pode ser removida.
>
> Envie também o `box_code` como **número** (`1`) e não como texto (`"1"`): com texto, a checagem de itens
> vinculados pode não identificar o vínculo. Consulte o `box_code` com `GET /v1/box` antes de remover.

### Payload mínimo

```json
{
  "boxes": {
    "box_ids": [1]
  }
}
```

### Payload máximo

```json
{
  "boxes": {
    "box_ids": [1, 2, 3, 15]
  }
}
```

### Exemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"boxes": {"box_ids": [1, 2]}}'
```

### Exemplo de response

**Recebido (`200`):**
```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

> Apesar do texto, a mensagem acima confirma apenas o **recebimento**. O resultado real chega no relatório por e-mail.

**Erro inesperado (`500`):**
```json
{
  "status": "error",
  "message": "Algo inesperado aconteceu. Tente novamente mais tarde!"
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success`, `warning` ou `error` |
| `message` | string | Mensagem traduzida conforme o header `lang` |
| `data` | object | Só em erros de validação: mapa `campo` → lista de mensagens |

### Regras da remoção (relatório por e-mail)

Cada caixa de `box_ids` é avaliada individualmente; uma falha não impede a remoção das demais.

| Regra | Mensagem no relatório quando falha |
|---|---|
| A caixa precisa existir na sua conta | `Box Error: box %s not found!` (`%s` = valor enviado) |
| A caixa **não pode ter itens vinculados** (`box_code` do item) | `Box Error: box %s cannot be deleted, has related item` |

> Para tirar de uso uma caixa que já tem itens vinculados, altere a caixa padrão desses itens
> ([`PUT /v1/item`](api-itens-v1.md#endpoint-atualizar-itens)) antes de remover, ou apenas desative a caixa
> com `POST /v1/box` e `activated: false`.

## Como usar as caixas na cotação

| Forma | Payload em `boxes[]` | Quando usar |
|---|---|---|
| Por código | `{ "box_code": 1 }` | Você guardou o código numérico (integer) |
| Por identificador | `{ "box_identifier": "CX0236" }` | Você usa o seu próprio código de embalagem |
| Manual | `{ "height": 20, "width": 20, "depth": 20, "weight": 0.3 }` | Embalagem avulsa, sem cadastro |
| Kit | `{ "kit": "KIT-001" }` | Caixa **e** itens vêm do cadastro de kit |

Use **uma** forma por caixa (não misture `box_code` com `box_identifier` ou com dimensões manuais na mesma posição).
Caixa referenciada que não existe gera erro na cotação:

| Situação | Status | Mensagem |
|---|---|---|
| `box_code`/`box_identifier` inexistente na plataforma | `422` | `O valor selecionado para o campo boxes.0.box code é inválido.` (ou `box identifier`) |
| Caixa não pertence à sua conta | `400` | `A caixa :name não foi encontrada para o usuário especificado!` |

Uso das caixas cadastradas conforme o agrupamento configurado na conta (cotação `type: items`, exclusiva da v2):

| Agrupamento | Como as caixas cadastradas são usadas |
|---|---|
| `default` | Cada item cadastrado precisa ter uma caixa padrão (`box_code` ou `box_identifier` no [cadastro do item](api-itens-v1.md)). Sem caixa vinculada: `A caixa com o box_code/box_identifier :name não existe ou está desativada` (HTTP `404`) |
| `smartbox` | Usa as caixas informadas com dimensões em `boxes[]` no payload da cotação. Sem caixa válida: `Para usar SmartBox, informe ao menos uma caixa em boxes[].` (HTTP `400`) |
| `all_in_one` | Consolida os itens; não depende da caixa padrão do item |

Veja a seção `boxes[]` em [Cotação (v1)](api-cotacao-v1.md#parâmetros--boxes).

## Regras de negócio
- Criação, atualização e remoção são processadas em **segundo plano**; o `200` confirma apenas o recebimento.
- A listagem fica em **cache por até 5 minutos** por conta e filtro (veja o alerta sobre paginação acima).
- Caixas são **por conta**: cada usuário lista, usa e remove apenas as suas.
- `box_identifier` deve ser único na sua conta — é o que permite atualizar em vez de duplicar. Use no máximo 50 caracteres e evite valores só numéricos.
- Dimensões e peso cadastrados são sempre em **cm** e **kg**, independentemente do `measurement` da cotação.
- O indicador `active` é informativo na listagem: **não referencie caixas inativas** em cotações. No momento, a cotação
  não bloqueia de forma garantida uma caixa inativa referenciada por `box_code`/`box_identifier`.
- Caixa com itens vinculados (pelo `box_code` do item) não é removida.
- A remoção é definitiva e é reportada por e-mail.

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `403` | `Has no permission!` — o perfil não tem permissão para listar, cadastrar ou remover caixas | Solicitar a permissão ao suporte |
| `422` | `O campo boxes é obrigatório.` — `boxes` ausente ou vazio (`POST`/`DELETE`) | Enviar ao menos uma caixa (`POST`) ou o envelope `boxes` (`DELETE`) |
| `422` | `O campo boxes deve conter um array.` — `boxes` não é lista/objeto | Corrigir a estrutura do payload |
| `422` | `O campo boxes.0.box identifier é obrigatório.` / `O campo boxes.0.name é obrigatório.` | Informar `box_identifier` e `name` em todas as caixas |
| `422` | `O campo boxes.0.box identifier não pode conter mais de 155 caracteres.` / `O campo boxes.0.name não pode conter mais de 155 caracteres.` | Encurtar o valor (para `box_identifier`, use até 50) |
| `422` | `O campo boxes.0.height deve conter um número superior ou igual a 1.` (idem `width`/`depth`) | Informar dimensões ≥ 1 cm |
| `422` | `O campo boxes.0.height deve conter um valor numérico.` (idem `width`/`depth`/`weight`) | Enviar números, sem unidade no valor |
| `422` | `O formato do valor informado no campo boxes.0.weight é inválido.` | Usar até 3 casas decimais, separador `.` e sem sinal |
| `422` | `O campo boxes.0.activated deve conter o valor verdadeiro ou falso.` | Enviar `true`/`false` |
| `422` | `O campo boxes.0.box code deve ser uma string.` | No `POST`, enviar `box_code` como texto (`"1"`) |
| `422` | `O campo boxes.0.box code não pode conter mais de 50 caracteres.` | Conferir o `box_code` |
| `422` | `O campo boxes.box ids é obrigatório.` / `O campo boxes.box ids deve conter um array.` (`DELETE`; no `GET`, só a segunda) | Enviar `box_ids` como lista não vazia |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Repetir com intervalo crescente; se persistir, acionar o suporte |
| Relatório (e-mail) | `Box Error: box %s not found!` | Conferir o `box_code` com `GET /v1/box` |
| Relatório (e-mail) | `Box Error: box %s cannot be deleted, has related item` | Desvincular a caixa dos itens ou apenas desativá-la |

> Nas mensagens de validação, o índice (`boxes.0`, `boxes.1`...) indica a posição da caixa no array, começando em `0`,
> e o `_` do nome do campo aparece como espaço (`box identifier`).

## Observações finais
- Cadastrar as caixas e vinculá-las aos itens é o pré-requisito para a [cotação por itens da v2](api-cotacao-v2.md#cotação-de-itens) (`type: items`) com agrupamento `default`.
- Mantenha os `box_identifier` alinhados com o seu ERP/WMS: isso simplifica muito a integração.
- Guarde o `box_code` retornado na listagem: ele é a referência mais confiável para cotação, itens e remoção.
- Para cadastrar os produtos que vão dentro das caixas, veja [Itens](api-itens-v1.md).
- Para referenciar as caixas na cotação, veja [Cotação (v1)](api-cotacao-v1.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
