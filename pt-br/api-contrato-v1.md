# Contrato

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Lista os contratos de transporte disponíveis para o usuário autenticado, de forma paginada.
- Um "contrato" é a combinação **transportador + serviço + condições comerciais** da sua conta.
  É o que a cotação retorna em `carriers[]`.
- Use este endpoint para diagnóstico e para saber de antemão quais transportadores podem aparecer
  nas suas cotações.
- Somente leitura: contratos são configurados comercialmente, não via API.

> Você **não precisa** chamar este endpoint no fluxo normal de integração. A cotação já devolve os
> contratos elegíveis para aquele envio específico, com preço e prazo.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
lang: pt
```

> O seu perfil precisa ter a permissão de listar contratos. Sem ela, a resposta é `403` com `Has no permission!`.

## Fluxo de uso

```
1. GET /v1/contract                 → contratos ativos e vigentes (padrão)
2. GET /v1/contract?status=all      → incluir também os desativados
3. GET /v1/contract?status=all&page=2 → próximas páginas (repita o status em cada página)
```

### Quando usar

| Situação | Como ajuda |
|---|---|
| A cotação retorna `Nenhum transportador encontrado para este usuário!` | Confirma se a conta tem algum contrato ativo e vigente |
| A cotação retorna `Nenhum contrato disponível para este usuário!` | Confirma a vinculação de contratos |
| Você quer mostrar ao usuário os transportadores habilitados | Lista nomes e vigências |
| Você recebeu `Contrato especificado não foi listado para esta cotação!` | Ajuda a conferir o `code` esperado |

## Conceitos essenciais

| Conceito | Significado |
|---|---|
| Contrato vinculado | Contrato associado diretamente à sua conta. Se o seu usuário é vinculado a uma conta principal, valem os contratos da conta principal |
| Contrato padrão | Contrato padrão da plataforma, usado quando a conta não tem contratos vinculados |
| Vigência | Período entre `initial_date` e `final_date`. **Só contratos vigentes hoje são listados**, independentemente do filtro `status` |
| Ativo / inativo | `active` = contrato habilitado; `inactive` = contrato desativado (ainda dentro da vigência). **Não** se refere à vigência |

## Endpoint: listar contratos

- **Método e rota:** `GET /v1/contract`
- **Quando usar:** diagnóstico de cotação sem transportadores ou exibição dos transportadores habilitados.

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Valores | Descrição |
|---|---|---|---|---|---|---|
| `status` | query | string | não | `active` | `active`, `inactive`, `all` | Filtra por situação do contrato (habilitado/desativado). Valores vazios ou desconhecidos caem no padrão `active`, sem erro |
| `page` | query | integer | não | `1` | inteiro ≥ 1 | Página da listagem. Valores inválidos são tratados como `1` |

- **Itens por página:** 15 (fixo; não há parâmetro para alterar).
- **Ordenação:** não garantida. Não dependa da ordem dos itens.

### Payload mínimo

Sem corpo e sem query string:

```
GET /v1/contract
```

### Payload máximo

Todos os parâmetros aceitos:

```
GET /v1/contract?status=all&page=2
```

### Exemplo de request

```bash
# Contratos ativos (padrão)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'

# Todos os contratos, ativos e inativos, página 2
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all&page=2' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contrato padrao PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "DHL",
        "initial_date": "2026-01-06 00:00:00",
        "final_date": "2026-12-31 23:59:59",
        "direct_quotation": false,
        "active": "active"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/contract?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos da resposta

Itens de `data.data[]`:

| Campo | Tipo | Descrição |
|---|---|---|
| `code` | integer | **Código do contrato.** É o mesmo valor que aparece em `carriers[].code` na cotação e que você envia em `carrier_code` na remessa |
| `contract_description` | string \| null | Descrição comercial do contrato |
| `contract_name` | string | Nome do contrato |
| `country` | string | País de origem do contrato (ISO 3166-1 alpha-2) |
| `carrier` | string \| null | Nome do serviço do transportador |
| `initial_date` | string | Início da vigência (`AAAA-MM-DD HH:MM:SS`) |
| `final_date` | string | Fim da vigência (`AAAA-MM-DD HH:MM:SS`) |
| `direct_quotation` | boolean | `true` quando o contrato cota diretamente no transportador (em tempo real), em vez de usar tabela de frete |
| `active` | string | `active` (habilitado) ou `inactive` (desativado) |

Paginação (em `data`):

| Campo | Tipo | Descrição |
|---|---|---|
| `current_page` | integer | Página atual |
| `from` / `to` | integer \| null | Posição do primeiro e do último item da página (`null` quando a página está vazia) |
| `total` | integer | Total de contratos encontrados |
| `per_page` | integer | Itens por página (`15`) |
| `last_page` | integer | Número da última página |
| `first_page_url` / `last_page_url` | string | URLs da primeira e da última página |
| `prev_page_url` / `next_page_url` | string \| null | URLs da página anterior e da próxima (`null` quando não existem) |
| `path` | string | URL base da listagem |
| `links` | array | Links de navegação (`url`, `label`, `active`). O `label` é traduzido conforme o header `lang` |

> ⚠️ As URLs de paginação **não preservam o filtro `status`**. Ao navegar com `status=inactive` ou
> `status=all`, monte a URL da próxima página você mesmo (`?status=all&page=N`) em vez de seguir
> `next_page_url` diretamente.

## Relação com a cotação e a remessa

```
GET /v1/contract            → contratos habilitados na conta (visão geral)
        │
POST /v1/quotation          → carriers[] : contratos elegíveis PARA AQUELE ENVIO
        │                     (com code, slug, preço, prazo e valid_until)
        ▼
POST /v1/shipping           → carrier_code (ou slug) escolhido
```

| Ponto de atenção | Explicação |
|---|---|
| A cotação pode retornar **menos** contratos que este endpoint | A cotação considera só contratos ativos do país de origem do envio, com tabela de frete para o tipo de objeto (documento ou não documento) e que atendam àquele peso/dimensão/rota |
| A remessa aceita **apenas** contratos listados na cotação | Enviar um `carrier_code` ativo mas ausente na cotação retorna `404 Contrato especificado não foi listado para esta cotação!` |
| O `slug` não aparece nesta listagem | Use o `slug` retornado em `carriers[]` na cotação |

## Regras de negócio

| Regra | Detalhe |
|---|---|
| Vinculação | O retorno depende dos contratos vinculados à sua conta (ou à conta principal, para usuários vinculados) |
| Contratos padrão | Se a busca não encontrar nenhum contrato vinculado **na página pedida**, a plataforma devolve os contratos padrão da plataforma com o mesmo filtro. Por isso, pedir uma página além da última pode trazer contratos padrão |
| Vigência | Contratos fora da vigência (antes de `initial_date` ou depois de `final_date`) nunca são listados |
| Filtro `status` | `active` (padrão), `inactive` ou `all`. Qualquer outro valor é tratado como `active` |
| Cotação | Contratos `inactive` (desativados) **não** aparecem nas cotações |
| Cotação direta | `direct_quotation: true` indica cotação em tempo real no transportador. Esses contratos podem demorar um pouco mais para responder e dependem de credenciais válidas (`Credenciais inválidas para cotação direta`) |

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `403` | `Has no permission!` | Solicitar ao suporte a permissão de listar contratos |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Acionar o suporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Conferir o header `Authorization: Bearer ...` |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Repetir com backoff; se persistir, acionar o suporte |

> Filtros inválidos **não** geram erro: `status` desconhecido vira `active` e `page` inválida vira `1`.
> Uma lista vazia (`data.data: []`, `total: 0`) significa que não há contratos vigentes nem contratos padrão
> para aquele filtro.

## Observações finais
- Este endpoint é **somente leitura**: contratos são configurados comercialmente, não via API.
- Use-o como ferramenta de diagnóstico quando a cotação não retornar transportadores.
- Para entender por que um contrato específico não apareceu na cotação, veja
  **Restrições de carga** em [Cotação (v1)](api-cotacao-v1.md#restrições-de-carga).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
