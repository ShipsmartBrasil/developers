# Itens

## Visão geral
- Gerencia itens cadastrados para uso em cotações, rascunhos e remessas.
- Os itens podem ser consultados, criados, atualizados e removidos em lote.
## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`

## Autenticação
- Bearer token no header `Authorization`.
- Header opcional de idioma: `lang`.

## Convenções
- `Content-Type: application/json`
- Consulta usa `items[skus][]` na query.
- Cadastro/atualização/removal são assíncronos.

## Fluxo de uso
1. Liste os itens já cadastrados.
2. Cadastre os itens base.
3. Atualize os itens quando houver mudança de dimensão, peso ou classificação.
4. Remova os itens obsoletos.

## Endpoints
### Listar itens
- Método e rota: `GET /v1/item`

### Criar itens
- Método e rota: `POST /v1/item`

### Atualizar itens
- Método e rota: `PUT /v1/item`

### Remover itens
- Método e rota: `DELETE /v1/item`

## Parâmetros
### Listagem
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `items[skus][]` | query | array | não | Lista de SKUs | `TSH-000-S` | Vazio retorna todos |

### Cadastro/atualização
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `items[]` | body | array | sim | Lista de itens | - | Processamento em lote |
| `name` | body | string | sim | Nome do item | `Tenis` | Obrigatório |
| `description` | body | string | sim | Descrição | `Nike 44` | Até 155 caracteres |
| `category` | body | string | sim | Categoria | `950450` | Valor livre na fonte |
| `country_code` | body | string | sim | País de origem | `BR` | ISO2 |
| `sku` | body | string | sim | SKU | `oLdTsW222` | Obrigatório |
| `ean` | body | string | não | EAN | `7894900010015` | Opcional |
| `height` | body | integer | sim | Altura | `5` | - |
| `width` | body | integer | sim | Largura | `5` | - |
| `depth` | body | integer | sim | Profundidade | `5` | - |
| `weight` | body | number | sim | Peso | `0.3` | - |
| `unit_value` | body | number | não | Valor unitário | `220.25` | Opcional |
| `selling_unit_value` | body | number | não | Valor de venda | `330.00` | Opcional |
| `hscode` | body | string | sim | HS code | `950450` | Obrigatório |
| `is_groupable` | body | boolean | sim | Pode agrupar | `true` | Obrigatório |
| `box_code` | body | integer | não | Caixa padrão | `3` | Opcional |
| `box_identifier` | body | string | não | Caixa padrão | `CX0236` | Opcional |

### Exclusão
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---|---|---|---|---|
| `items.skus` | body | array | sim | SKUs a remover | `["TSH-000-S"]` | Obrigatório |

## Payload mínimo e máximo
### Payload mínimo
```json
{
  "items": [
    {
      "name": "Teste prod 1",
      "description": "Teste prod 1",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "hscode": "673057",
      "is_groupable": false
    }
  ]
}
```

### Payload máximo
```json
{
  "items": [
    {
      "name": "Teste prod 1 update",
      "description": "Teste prod 1 update",
      "category": "outros",
      "country_code": "BR",
      "sku": "alsarew",
      "ean": "7894900010015",
      "height": 10,
      "width": 8,
      "depth": 5,
      "weight": 0.6,
      "unit_value": 220.25,
      "selling_unit_value": 330,
      "hscode": "673057",
      "is_groupable": false,
      "box_code": 129,
      "box_identifier": "CX0236"
    }
  ]
}
```

## Exemplo de request
```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/item?items[skus][]=TSH-000-S&items[skus][]=TSH-000-M' \
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
        "name": "Tenis",
        "description": "Nike 44",
        "category": "950450",
        "country_code": "BR",
        "sku": "oLdTsW222",
        "ean": "7894900010015",
        "height": 5,
        "width": 5,
        "depth": 5,
        "weight": 0.3,
        "hscode": "950450",
        "is_groupable": true
      }
    ]
  }
}
```

## Regras de negócio
- Os itens são processados em job assíncrono.
- A listagem usa coleção por usuário.
- `box_code` e `box_identifier` são opcionais e servem como vínculo padrão com caixa.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `422` | Validação | Corrigir campos obrigatórios |
| `500` | Erro inesperado | Acionar o time responsável |
