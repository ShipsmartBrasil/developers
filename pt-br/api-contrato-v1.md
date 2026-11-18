# Contrato

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Lista os contratos de transporte disponíveis para o usuário autenticado.
- Um "contrato" é a combinação **transportador + serviço + condições comerciais** da sua conta.
  É o que a cotação retorna em `carriers[]`.
- Use este endpoint para diagnóstico e para saber de antemão quais transportadores podem aparecer
  nas suas cotações.

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

## Quando usar

| Situação | Como ajuda |
|---|---|
| A cotação retorna `Nenhum transportador encontrado para este usuário!` | Confirma se a conta tem algum contrato ativo |
| A cotação retorna `Nenhum contrato disponível para este usuário!` | Confirma a vinculação de contratos |
| Você quer mostrar ao usuário os transportadores habilitados | Lista nomes e vigências |
| Você recebeu `Contrato especificado não foi listado para esta cotação!` | Ajuda a conferir o `code`/`slug` esperado |

## Endpoint: listar contratos

- **Método e rota:** `GET /v1/contract`

| Parâmetro | Local | Tipo | Obrigatório | Padrão | Valores | Descrição |
|---|---|---|---|---|---|---|
| `status` | query | string | não | `active` | `active`, `inactive`, `all` | Filtra por vigência do contrato. Valores desconhecidos caem no padrão `active` |
| `page` | query | integer | não | `1` | - | Página da listagem |

- **Itens por página:** 15

### Exemplo de request

```bash
# Contratos ativos (padrão)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'

# Todos os contratos, ativos e inativos
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contrato padrão PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "DHL",
        "initial_date": "2026-01-06 00:00:00",
        "final_date": "2026-12-31 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
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
| `code` | integer | **Código do contrato.** É o mesmo valor que aparece em `carriers[].code` na cotação e que você envia em `carrier_code` na remessa |
| `contract_description` | string | Descrição comercial do contrato |
| `contract_name` | string | Nome do contrato |
| `country` | string | País de origem do contrato (ISO 3166-1 alpha-2) |
| `carrier` | string | Nome do transportador |
| `initial_date` | string | Início da vigência (`Y-m-d H:i:s`) |
| `final_date` | string | Fim da vigência (`Y-m-d H:i:s`) |
| `active` | string | `active` ou `inactive` |
| `direct_quotation` | boolean | `true` quando o contrato cota diretamente no transportador (em tempo real), em vez de usar tabela de frete |

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
| A cotação pode retornar **menos** contratos que este endpoint | O contrato existe, mas não atende àquele peso/dimensão/rota |
| A remessa aceita **apenas** contratos listados na cotação | Enviar um `carrier_code` ativo mas ausente na cotação retorna `404 Contrato especificado não foi listado para esta cotação!` |
| O `slug` não aparece nesta listagem | Use o `slug` retornado em `carriers[]` na cotação |

## Regras de negócio
- O retorno depende da **vinculação do seu usuário** aos contratos.
- Se nenhum contrato for encontrado com o filtro informado, a plataforma faz uma segunda busca
  considerando os contratos padrão da conta.
- Contratos `inactive` (fora da vigência) **não** aparecem nas cotações.
- `direct_quotation: true` indica cotação em tempo real no transportador. Esses contratos podem demorar
  um pouco mais para responder e dependem de credenciais válidas
  (`Credenciais inválidas para cotação direta`).

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `403` | Sem permissão para listar contratos | Solicitar a permissão ao suporte |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Este endpoint é **somente leitura**: contratos são configurados comercialmente, não via API.
- Use-o como ferramenta de diagnóstico quando a cotação não retornar transportadores.
- Para entender por que um contrato específico não apareceu na cotação, veja
  **Restrições de carga** em [Cotação (v1)](api-cotacao-v1.md#restrições-de-carga).
