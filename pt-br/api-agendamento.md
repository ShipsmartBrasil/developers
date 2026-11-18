# Agendamento de Coleta

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md) e [Remessa (v1)](api-remessa-v1.md)

## Visão geral
- Solicita ao transportador que ele **retire** o volume no seu endereço, em vez de você levá-lo a um ponto
  de postagem.
- Permite consultar o status do agendamento e cancelá-lo com um motivo padronizado.
- Funciona para remessas criadas na **v1 e na v2**.

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

## Pré-requisitos

Antes de agendar, confirme que:

1. A remessa já está no status **`shipped`** ([rastreio](api-rastreio.md)) — ou seja, já foi processada no
   transportador e os documentos existem.
2. O transportador do contrato escolhido **possui serviço de coleta integrado**.
3. A remessa **não tem** outro agendamento em andamento.

## Fluxo de uso

```
1. GET  /v1/shipping/{shipping}/status   → confirmar status "shipped"
2. POST /v1/schedule                     → criar o agendamento
3. GET  /v1/schedule/{shipping}          → acompanhar até status "success"
4. DELETE /v1/schedule/{shipping}/{reason} → cancelar (se necessário)
```

## Convenções
- Datas no formato `Y-m-d` (ex.: `2026-03-10`).
- Horários no formato `H:i:s` (ex.: `08:00:00`).
- O identificador `{shipping}` aceita o **UUID da cotação** ou o **número do AWB** (remessas dos
  últimos 90 dias). Diferente dos endpoints de remessa, **a chave da NF-e não é aceita aqui**.

## Endpoint: criar agendamento

- **Método e rota:** `POST /v1/schedule`
- **Quando usar:** após a remessa atingir `shipped`.

### Parâmetros

| Campo | Tipo | Obrigatório | Mín. | Máx. | Formato | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | string | **sim** se `awb_number` não for enviado | - | - | UUID | Código da cotação/remessa |
| `awb_number` | string | **sim** se `shipping` não for enviado | - | 50 | - | Número do AWB (remessas dos últimos 90 dias) |
| `date` | string | **sim** | - | - | `Y-m-d` | Data da coleta. Precisa ser **maior que hoje** |
| `initial_time` | string | **sim** | - | - | `H:i:s` | Início da janela de coleta |
| `final_time` | string | **sim** | - | - | `H:i:s` | Fim da janela de coleta |
| `instruction` | string | **sim** | 5 | 80 | - | Instrução para o motorista (ex.: onde retirar, com quem falar) |
| `address` | object | não | - | - | - | Endereço alternativo de coleta (ver tabela abaixo) |

> Envie **`shipping` ou `awb_number`**, não os dois.

### Parâmetros — `address` (endereço alternativo de coleta)

Envie apenas se a coleta for em um endereço **diferente** do remetente da remessa.
**O endereço precisa estar no mesmo país do remetente.**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `street` | string | **sim** se `address` for enviado | 5 | 70 | Logradouro |
| `city` | string | **sim** se `address` for enviado | 5 | 35 | Cidade |
| `name` | string | **sim** se `address` for enviado | 5 | 35 | Nome do contato no local |
| `email` | string | **sim** se `address` for enviado | - | 100 | E-mail do contato |
| `phone` | string | **sim** se `address` for enviado | 10 | 25 | Telefone do contato |
| `postal_code` | string | condicional | - | 12 | CEP/ZIP. **Obrigatório para coletas no Brasil** |
| `state_code` | string | não | - | 10 | Sigla do estado. Precisa pertencer ao país do remetente |
| `number` | string | condicional | - | 6 | Número do imóvel. **Obrigatório para coletas no Brasil** |
| `district` | string | condicional | - | 30 | Bairro. **Obrigatório para coletas no Brasil** |
| `complement` | string | não | - | 20 | Complemento |
| `phone_extension` | string | não | - | 5 | Ramal |
| `federal_tax_id` | string | condicional | - | 20 | CPF/CNPJ. **Obrigatório e validado para coletas no Brasil** |

> Quando `address` é omitido, a plataforma resolve o endereço automaticamente: primeiro a predefinição de
> coleta da sua conta e, na falta dela, o endereço do remetente da remessa.
>
> Em alguns tipos de conta (pessoa física e PJ mono-CD), o endereço alternativo é **ignorado** e a coleta
> usa sempre o endereço do remetente.

### Payload mínimo

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirar na recepcao, falar com o Carlos"
}
```

### Payload máximo

```json
{
  "awb_number": "271249639538",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Retirar no galpao 2, portao lateral, falar com o Carlos",
  "address": {
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpao 2",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

### Exemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2026-03-10",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirar na recepcao, falar com o Carlos"
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> A resposta confirma o **recebimento**. A confirmação do transportador é assíncrona: consulte o
> agendamento até o status virar `success` e o `confirmation_code` aparecer.

### Regras de data e janela

| Regra | Detalhe | Mensagem quando falha |
|---|---|---|
| Data futura | `date` precisa ser **maior que a data atual** | `A data de agendamento não pode ser menor que a data Atual` |
| Sem fim de semana | Sábado e domingo não são aceitos | `Coleta não pode ser realizada em um final de semana` |
| Sem feriado nacional | Para coletas no Brasil | `Coleta não pode ser realizada durante um feriado nacional` |
| Limite de 7 dias úteis | A data não pode ultrapassar 7 dias úteis a partir de hoje | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Feriados nacionais brasileiros considerados: **01/01, 21/04, 01/05, 07/09, 12/10, 02/11, 15/11, 20/11 e 25/12**.

## Endpoint: consultar agendamento

- **Método e rota:** `GET /v1/schedule/{shipping}`

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `shipping` | path | string | **sim** | UUID da cotação ou número do AWB (últimos 90 dias) |

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirar na recepcao, falar com o Carlos",
    "status": {
      "code": "processing",
      "description": "Agendamento da Remessa Recebida. Em breve será processada!",
      "reason": null
    }
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.confirmation_code` | string | Número de confirmação da coleta no transportador. Preenchido quando o status chega a `success` |
| `data.date` | string | Data da coleta (`Y-m-d`) |
| `data.inicial_time` | string | Início da janela (`H:i:s`). ⚠️ **A chave é `inicial_time`** na resposta, enquanto no request o campo é `initial_time` |
| `data.final_time` | string | Fim da janela (`H:i:s`) |
| `data.instruction` | string | Instrução informada na criação |
| `data.status.code` | string | Código do status do agendamento (ver tabela) |
| `data.status.description` | string | Descrição traduzida conforme o header `lang` |
| `data.status.reason` | string \| null | Código do motivo de cancelamento (`001`–`008`). `null` enquanto não houver cancelamento |

### Status do agendamento

| `code` | Significado |
|---|---|
| `processing` | Agendamento recebido, aguardando envio ao transportador |
| `success` | **Coleta confirmada pelo transportador.** `confirmation_code` disponível |
| `canceling` | Cancelamento solicitado, aguardando processamento |
| `terminating` | Cancelamento em processamento no transportador |
| `canceled` | Cancelamento concluído |
| `error` | Falha no agendamento ou no cancelamento |

### Tipo de processamento

A plataforma classifica o agendamento automaticamente, sem campo no payload:

| Tipo | Quando | Comportamento |
|---|---|---|
| `automatico` | Coleta para o dia seguinte (até 1 dia) | Enviado ao transportador imediatamente |
| `agendado` | Coleta com mais de 1 dia de antecedência | Enviado ao transportador 1 dia antes da coleta |

> Por isso um agendamento futuro pode permanecer em `processing` por dias. Isso é esperado.

## Endpoint: cancelar agendamento

- **Método e rota:** `DELETE /v1/schedule/{shipping}/{reason}`
- **Quando usar:** quando a coleta já foi **confirmada** (status `success`) e precisa ser desmarcada.

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `shipping` | path | string | **sim** | UUID da cotação ou número do AWB (últimos 90 dias) |
| `reason` | path | string | **sim** | Código do motivo (ver tabela). Enviado ao transportador |

### Motivos de cancelamento

| Código | Motivo |
|---|---|
| `001` | Package not ready (volume não está pronto) |
| `002` | Rates too high (tarifas muito altas) |
| `003` | Transit time too slow (prazo de trânsito muito lento) |
| `004` | Take to service center or drop box (será levado a um ponto de postagem) |
| `005` | Commitment time not met (prazo acordado não cumprido) |
| `006` | Reason not given (motivo não informado) |
| `007` | Other (outro) |
| `008` | Pickup modified (coleta modificada) |

> Esses códigos são padronizados pelos transportadores e repassados a eles sem tradução.
> Use `006` ou `007` quando nenhum dos específicos se aplicar.

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Regras do cancelamento

| Regra | Mensagem quando falha |
|---|---|
| O `reason` precisa estar entre `001` e `008` | `Motivo do cancelamento não é compativel` |
| O agendamento precisa existir | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| A remessa precisa estar em `shipped` | idem |
| O agendamento precisa estar em `success` (já confirmado pelo transportador) | idem |

> Após a solicitação, o status passa para `canceling` com a descrição
> `Solicitação de cancelamento recebida! Em breve será processada!`, depois `terminating` e, por fim,
> `canceled` (ou `error`).

## Regras de negócio consolidadas

| Regra | Detalhe |
|---|---|
| Status da remessa | A remessa precisa estar em `shipped` |
| Agendamento único | Não é possível ter dois agendamentos ativos na mesma remessa |
| Transportador | O serviço precisa suportar coleta integrada |
| Data | Futura, dia útil, sem feriado nacional (BR), até 7 dias úteis |
| Endereço BR | Exige `postal_code`, `number`, `district` e `federal_tax_id` válido (CPF ou CNPJ) |
| País do endereço | O `state_code` precisa pertencer ao país do remetente da remessa |
| Propriedade | Só é possível agendar/consultar/cancelar remessas da sua própria conta |
| AWB | Resolve apenas remessas dos **últimos 90 dias** |

## Erros comuns

| Status | Mensagem | Como corrigir |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Consultar o agendamento existente antes de criar outro |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Aguardar o status `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Levar o volume a um ponto de postagem |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Usar uma data futura |
| `400` | `Coleta não pode ser realizada em um final de semana` | Escolher um dia útil |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Escolher outro dia útil |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Reduzir a antecedência |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Completar `number` e `district` (coletas no Brasil) |
| `400` | `O atributo federal_tax_id não é válido` | Informar um CPF ou CNPJ válido |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Usar um `state_code` do país do remetente |
| `400` | `Motivo do cancelamento não é compativel` | Usar um código entre `001` e `008` |
| `403` | Sem permissão | Solicitar a permissão de agendamento ao suporte |
| `404` | `Agendamento não foi encontrado para esta remessa` | Conferir o identificador ou criar o agendamento |
| `422` | Validação | Corrigir data/hora/instrução conforme os formatos |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Sempre consulte o agendamento antes de criar ou cancelar: isso evita requisições redundantes e o erro
  `já tem um agendamento em progresso`.
- Atenção à diferença de nomenclatura: **`initial_time` no request** e **`inicial_time` na resposta**.
- Agendamentos futuros permanecem em `processing` até 1 dia antes da coleta. Não é erro.
