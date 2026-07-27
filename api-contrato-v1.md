# Contrato

## Visão geral
- Lista os contratos disponíveis para o usuário autenticado.
- Os contratos são usados na cotação e na criação de remessas.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- A listagem pode filtrar por `status`.

## Fluxo de uso
1. Autentique o usuário.
2. Liste os contratos disponíveis.
3. Use o `code` do contrato na cotação ou remessa.

## Endpoints
### Listar contratos
- Método e rota: `GET /v1/contract`
- Quando usar: para identificar contratos ativos/inativos antes da cotação.

## Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `status` | query | string | não | Filtra contratos | `active` | `active`, `inactive`, `all` |

## Payload mínimo e máximo
- Não se aplica ao endpoint de listagem.

## Exemplo de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=active' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Exemplo de response
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
        "carrier": "Carrier Name",
        "initial_date": "2023-03-06 00:00:00",
        "final_date": "2023-06-30 23:59:59",
        "active": "active",
        "direct_quotation": false
      }
    ]
  }
}
```

## Regras de negócio
- Se não houver contratos no primeiro filtro, a API tenta uma segunda busca com comportamento alternativo do backend.
- O retorno depende da vinculação do usuário com os contratos.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `500` | Erro inesperado | Acionar o time responsável |
