# Agendamento de Coleta

## Visão geral
- Cria, consulta e cancela agendamentos de coleta vinculados à remessa.
- A coleta pode ser feita pela cotação/remessa ou por AWB.
- Associado ao fluxo operacional principal.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Convenções
- Datas no formato `Y-m-d`.
- Horários no formato `H:i:s`.
- O agendamento deve respeitar o país do remetente.

## Fluxo de uso
1. Garanta que a remessa esteja pronta para agendamento.
2. Crie a coleta com janela de horário e instrução.
3. Consulte o status do agendamento.
4. Se necessário, cancele usando um dos motivos permitidos.

## Endpoints
### Criar agendamento
- Método e rota: `POST /v1/schedule`
- Regras importantes: informe `shipping` ou `awb_number`.

### Consultar agendamento
- Método e rota: `GET /v1/schedule/{shipping}`

### Cancelar agendamento
- Método e rota: `DELETE /v1/schedule/{shipping}/{reason}`

### Parâmetros
| Nome | Local | Tipo | Obrigatório | Descrição | Exemplo | Observações |
|---|---|---:|---:|---|---|---|
| `shipping` | body/path | string | condicional | UUID da cotação/remessa | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Obrigatório se `awb_number` não vier |
| `awb_number` | body | string | condicional | AWB | `271249639538` | Obrigatório se `shipping` não vier |
| `date` | body | date | sim | Data da coleta | `2023-11-21` | Formato `Y-m-d` |
| `initial_time` | body | time | sim | Início da janela | `08:00:00` | Formato `H:i:s` |
| `final_time` | body | time | sim | Fim da janela | `18:00:00` | Formato `H:i:s` |
| `instruction` | body | string | sim | Instrução para coleta | `Deixar na portaria` | 5 a 80 caracteres |
| `address` | body | object | não | Endereço alternativo | - | Deve ser do mesmo país do remetente |
| `reason` | path | string | sim | Motivo do cancelamento | `001` | Código fixo |

### Motivos de cancelamento
| Código | Significado |
|---|---|
| `001` | Package not ready |
| `002` | Rates too high |
| `003` | Transit time too slow |
| `004` | Take to service center or drop box |
| `005` | Commitment time not met |
| `006` | Reason not given |
| `007` | Other |
| `008` | Pickup modified |

### Exemplo de request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2023-11-21",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Deixar na portaria"
  }'
```

### Exemplo de response
```json
{
  "status": "success",
  "message": "Agendamento de coleta registrado com sucesso"
}
```

## Regras de negócio
- Só é possível agendar se a remessa estiver pronta para isso.
- Já pode haver um agendamento em progresso, e neste caso a API retorna warning.
- O endereço alternativo precisa respeitar o mesmo país do remetente.

## Erros comuns
| Status | Significado | Como corrigir |
|---|---|---|
| `400` | Agendamento já existe ou remessa não está pronta | Ajustar o fluxo | 
| `422` | Validação | Corrigir data, hora ou campos obrigatórios |
| `500` | Erro inesperado | Acionar o time responsável |

## Observações finais
- Consulte o status antes de cancelar para evitar solicitações redundantes.
