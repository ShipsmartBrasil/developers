# Caixa

## Visão geral
- Gerencia o cadastro e a consulta de caixas usadas em cotações, rascunhos e remessas.
- Use para padronizar dimensões, peso e identificação das embalagens.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- Consulta usa `boxes[box_ids][]` na query.
- Cadastro/atualização ocorre de forma assíncrona.

## Fluxo de uso
1. Liste as caixas disponíveis.
2. Cadastre ou atualize caixas base.
3. Remova caixas quando necessário.
4. Reutilize os códigos nas cotações e rascunhos.

## Endpoints
### Listar caixas
- Método e rota: `GET /v1/box`
- Quando usar: para buscar caixas cadastradas ou filtrar por IDs.

### Criar ou atualizar caixas
- Método e rota: `POST /v1/box`
- Quando usar: ao manter a base de embalagens do integrador.

### Remover caixas
- Método e rota: `DELETE /v1/box`
- Quando usar: ao excluir uma ou mais caixas cadastradas.

## Parâmetros
### Listagem
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `boxes[box_ids][]` | query | array | não | Lista de códigos/identificadores | `1` | Vazio retorna todas |

### Cadastro
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `boxes[]` | body | array | sim | Lista de caixas | - | Cada item é processado em lote |
| `box_code` | body | string | não | Código da caixa | `1` | Opcional para atualização |
| `box_identifier` | body | string | sim | Identificador da caixa | `CX0236` | Obrigatório no cadastro |
| `name` | body | string | sim | Nome | `Cartucho Individual` | Até 155 caracteres |
| `height` | body | number | sim | Altura | `65` | >= 1 |
| `width` | body | number | sim | Largura | `65` | >= 1 |
| `depth` | body | number | sim | Profundidade | `65` | >= 1 |
| `weight` | body | number | sim | Peso | `2.8` | Regex com até 3 casas decimais |
| `activated` | body | boolean | sim | Ativada | `true` | - |

### Exclusão
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `boxes.box_ids` | body | array | sim | IDs das caixas | `[1, 2]` | Lista obrigatória |

## Payload mínimo e máximo
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
      "box_code": 1,
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

## Exemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/box \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
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
  }'
```

## Exemplo de response
```json
{
  "status": "success",
  "message": "Sua requisição será processada. Em breve te daremos um retorno."
}
```

## Regras de negócio
- O cadastro é processado em job assíncrono.
- A listagem pode ser cacheada por usuário e página.
- A exclusão aceita múltiplos IDs.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `422` | Validação | Corrigir campos obrigatórios |
| `400` | Requisição inválida | Verificar payload | 
| `500` | Erro inesperado | Acionar o time responsável |
