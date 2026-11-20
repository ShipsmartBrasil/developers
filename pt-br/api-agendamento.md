# Agendamento de Coleta

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md) e [Remessa (v1)](api-remessa-v1.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Solicita ao transportador que ele **retire** o volume no seu endereço, em vez de você levá-lo a um ponto
  de postagem.
- Permite consultar o andamento do agendamento e cancelá-lo com um motivo padronizado.
- Funciona para remessas criadas na **v1 e na v2** (os endpoints de agendamento existem apenas em `/v1`).
- O pedido é **assíncrono**: a API registra a solicitação e o envio ao transportador acontece em segundo
  plano. A confirmação chega depois, na consulta do agendamento.
- Cada remessa aceita **um único agendamento** pela API.

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

> `Content-Type` só é necessário no `POST`. Cada operação exige uma permissão própria no seu perfil
> (criar, consultar e cancelar agendamento). Sem a permissão, a resposta é `403` com `Has no permission!`.

## Fluxo de uso

```
1. GET    /v1/shipping/{shipping}/status    → confirmar que o status ATUAL é "shipped"
2. POST   /v1/schedule                      → registrar o agendamento (status "processing")
3. GET    /v1/schedule/{shipping}           → acompanhar até o status "success"
4. DELETE /v1/schedule/{shipping}/{reason}  → cancelar, se necessário (somente com status "success")
```

### Antes de agendar, confirme

| # | Condição | Como verificar |
|---|---|---|
| 1 | O status **atual** da remessa é **`shipped`** (processada no transportador, documentos disponíveis). Depois que a remessa avança (ex.: em trânsito), o agendamento deixa de ser aceito | [Rastreio](api-rastreio.md) |
| 2 | O transportador do contrato da remessa **possui coleta integrada** (ver [Transportadores com coleta integrada](#transportadores-com-coleta-integrada)) | Contrato escolhido na remessa |
| 3 | A remessa **ainda não tem** nenhum agendamento — nem mesmo um cancelado ou com erro | `GET /v1/schedule/{shipping}` deve responder `404` |
| 4 | A remessa foi criada pelo **mesmo usuário** do token | - |
| 5 | A sua conta **não** tem a coleta automática habilitada nas predefinições (ver [Coleta automática pelas predefinições da conta](#coleta-automática-pelas-predefinições-da-conta)) | Painel da conta |

## Conceitos essenciais

### Convenções

| Item | Formato | Exemplo |
|---|---|---|
| Data da coleta | `AAAA-MM-DD` (exatamente esse formato) | `2026-03-10` |
| Horários da janela | `HH:MM:SS` (com segundos) | `08:00:00` |
| Data "atual" | Considera o fuso horário da plataforma (horário de Brasília) | - |
| Identificador `{shipping}` | **UUID da cotação** ou **número do AWB** (remessas criadas nos últimos 90 dias) | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` |

> Diferente dos endpoints de remessa, **a chave da NF-e não é aceita** como identificador no agendamento.
> Um identificador que contém hífen (`-`) é sempre tratado como UUID; sem hífen, é tratado como AWB.

### Transportadores com coleta integrada

| Transportador | Agendamento pela API | Cancelamento pela API |
|---|---|---|
| DHL | sim | sim |
| FedEx | sim | sim |
| UPS | sim | sim |
| SkyPostal | sim | sim |
| CTBI Log | sim | **não** (o pedido de cancelamento termina em `error`; acione o suporte) |
| GLS | sim | **não** (o pedido de cancelamento termina em `error`; acione o suporte) |

> Lista vigente na data desta versão do guia; ela pode ser ampliada. Para qualquer outro transportador,
> o `POST` responde `400` com `Esta transportadora não possui serviço de coleta integrado`.

### Endereço da coleta

O endereço de retirada é definido de uma destas formas. **Não misture**: ou você envia `address`, ou deixa
a plataforma resolver.

| Forma | Como enviar | Endereço usado |
|---|---|---|
| **Forma A — automático** | Omita `address` (ou envie `null` / `{}`) | 1º a predefinição de coleta da sua conta para o país do remetente (se habilitada e com endereço cadastrado); 2º o endereço do remetente da remessa |
| **Forma B — endereço alternativo** | Envie o objeto `address` completo | O endereço informado, que precisa estar **no mesmo país do remetente** |

> Em contas do tipo **pessoa física** e **PJ mono-CD**, o objeto `address` é **ignorado** e a coleta usa
> sempre o endereço do remetente da remessa. Apenas o formato dos campos (`422`) é conferido; as regras de
> endereço (país, CEP, documento) não são aplicadas.

### Status do agendamento

| `code` | Significado | Próximo passo |
|---|---|---|
| `processing` | Agendamento registrado, aguardando envio/confirmação do transportador | Aguardar e consultar novamente |
| `success` | **Coleta confirmada pelo transportador.** `confirmation_code` disponível | Deixar o volume pronto na janela informada |
| `canceling` | Cancelamento solicitado, aguardando envio ao transportador | Aguardar |
| `terminating` | Cancelamento enviado ao transportador, aguardando confirmação | Aguardar |
| `canceled` | Cancelamento concluído | - |
| `error` | Falha no agendamento ou no cancelamento. O motivo vem em `status.description` | Ler a descrição e acionar o suporte |

```
POST                         DELETE
 │                             │
 ▼                             ▼
processing ──► success ──► canceling ──► terminating ──► canceled
 │                             │              │
 └──► error                    └──► error     └──► error
```

### Tipo de processamento

A plataforma classifica o agendamento automaticamente pela data informada. Não existe campo no payload.

| Tipo | Quando | Comportamento |
|---|---|---|
| Imediato | `date` = amanhã | Enviado ao transportador no próximo ciclo de processamento |
| Programado | `date` depois de amanhã ou mais tarde | Enviado ao transportador **1 dia antes** da data da coleta |

> Por isso um agendamento programado permanece em `processing` por alguns dias. Isso é esperado.
> Exemplo: pedido feito na sexta-feira para a segunda-feira seguinte é **programado** (enviado no domingo).

### Coleta automática pelas predefinições da conta

Se a sua conta tiver a **coleta automática** habilitada nas predefinições (configurada no painel), a
plataforma cria o agendamento sozinha assim que os documentos da remessa são gerados, sem chamada à API:

| Item | Valor aplicado |
|---|---|
| Data | Próximo dia útil (para remetente no Brasil, pula também um feriado nacional) |
| Janela | `08:00:00` a `18:00:00` |
| Instrução | A instrução cadastrada na predefinição |
| Endereço | Mesma regra da Forma A (predefinição de coleta e, na falta dela, o remetente) |

> Nesse caso, um `POST /v1/schedule` para a mesma remessa responde
> `400 Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso`.
> Use apenas o `GET /v1/schedule/{shipping}` para acompanhar.

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
> Use `006` ou `007` quando nenhum dos específicos se aplicar. Envie sempre os **3 dígitos** (`001`, não `1`).

## Endpoint: criar agendamento

- **Método e rota:** `POST /v1/schedule`
- **Quando usar:** com a remessa no status atual `shipped`, para pedir a retirada do volume.

### Parâmetros

| Campo | Tipo | Obrigatório | Mín. | Máx. | Formato / valores | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | string | **sim** se `awb_number` não for enviado | - | - | UUID de uma cotação existente | Código da cotação/remessa |
| `awb_number` | string | **sim** se `shipping` não for enviado | - | 50 | - | Número do AWB (remessas criadas nos últimos 90 dias) |
| `date` | string | **sim** | - | - | `AAAA-MM-DD` | Data da coleta. Ver [Regras de data](#regras-de-data) |
| `initial_time` | string | **sim** | - | - | `HH:MM:SS` | Início da janela de coleta |
| `final_time` | string | **sim** | - | - | `HH:MM:SS` | Fim da janela de coleta |
| `instruction` | string | **sim** | 5 | 80 | - | Instrução para o motorista (onde retirar, com quem falar) |
| `address` | object \| null | não | - | - | - | Endereço alternativo de coleta (Forma B). Ver tabela abaixo |

> Envie **`shipping` ou `awb_number`**, não os dois. Se os dois forem enviados, o `awb_number` prevalece.
>
> A API **não** compara `initial_time` com `final_time` nem valida horário comercial. Garanta que o fim seja
> posterior ao início e que a janela seja aceita pelo transportador; caso contrário, o agendamento pode
> terminar em `error`.

### Parâmetros — `address` (endereço alternativo de coleta)

Envie apenas se a coleta for em um endereço **diferente** do automático.

**Atenção: quando `address` é enviado, todas as 12 chaves abaixo precisam estar presentes no objeto.**
Uma chave ausente faz a requisição falhar com `500`. Quando não tiver o dado, use `null` nos campos que
aceitam nulo e `""` (texto vazio) em `complement` e `phone_extension`.

| Campo | Tipo | Aceita `null` | Mín. | Máx. | Regra para remetente no Brasil | Descrição |
|---|---|---|---|---|---|---|
| `street` | string | não | 5 | 70 | - | Logradouro |
| `number` | string | sim | - | 6 | **Obrigatório** (não pode ser vazio) | Número do imóvel |
| `district` | string | sim | - | 30 | **Obrigatório** (não pode ser vazio) | Bairro |
| `complement` | string | não (use `""`) | - | 20 | - | Complemento |
| `city` | string | não | 5 | 35 | - | Cidade |
| `state_code` | string | sim | - | 10 | - | Sigla do estado. Precisa pertencer ao país do remetente |
| `postal_code` | string | sim | - | 12 | - | CEP/ZIP. Se `state_code` vier vazio, o estado é descoberto a partir dele |
| `name` | string | não | 5 | 35 | - | Nome do contato no local |
| `email` | string | não | - | 100 | - | E-mail do contato (formato de e-mail válido) |
| `phone` | string | não | 10 | 25 | - | Telefone do contato |
| `phone_extension` | string | não (use `""`) | - | 5 | - | Ramal |
| `federal_tax_id` | string | sim | - | 20 | **Obrigatório: CPF ou CNPJ válido** | Documento do responsável pelo local |

Como o país e o estado são conferidos:

| Situação | Comportamento |
|---|---|
| `state_code` informado | Precisa ser um estado do país do remetente (ou o próprio código do país). Senão: `Endereço enviado não pertence ao pais da remessa` |
| `state_code` vazio e `postal_code` informado (remetente BR) | O estado é obtido pelo CEP. CEP não localizado: `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` |
| `state_code` vazio e `postal_code` informado (outros países) | O estado é obtido pelo código postal. Não localizado: `Informações de endereço de coleta insuficientes ou inválidas` |
| `state_code` e `postal_code` vazios | Não há conferência. Recomendamos sempre enviar os dois para evitar recusa do transportador |

### Payload mínimo

Forma A com `shipping` (UUID):

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirar na recepcao"
}
```

Forma A com `awb_number`:

```json
{
  "awb_number": "1234567890",
  "date": "2026-03-10",
  "initial_time": "08:00:00",
  "final_time": "18:00:00",
  "instruction": "Retirar na recepcao"
}
```

### Payload máximo

Forma B (endereço alternativo, remetente no Brasil), com todos os campos aceitos:

```json
{
  "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "date": "2026-03-10",
  "initial_time": "09:00:00",
  "final_time": "17:00:00",
  "instruction": "Retirar no galpao 2, portao lateral, falar com o Carlos",
  "address": {
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpao 2",
    "city": "Cidade Exemplo",
    "state_code": "SP",
    "postal_code": "01000000",
    "name": "Carlos Pereira",
    "email": "logistica@example.com",
    "phone": "11999999999",
    "phone_extension": "12",
    "federal_tax_id": "12345678000199"
  }
}
```

> Para usar o AWB, troque `shipping` por `awb_number`. O `federal_tax_id` do exemplo é fictício: em
> remessas do Brasil, a API exige um CPF ou CNPJ válido.

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
    "instruction": "Retirar na recepcao"
  }'
```

### Exemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

> A resposta confirma apenas o **registro** do pedido. A confirmação do transportador é assíncrona:
> consulte o agendamento até o status virar `success` e o `confirmation_code` aparecer.

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` quando o agendamento foi registrado |
| `message` | string | Mensagem traduzida conforme o header `lang` |

> Não há `data` nesta resposta. Para obter detalhes, use [consultar agendamento](#endpoint-consultar-agendamento).

### Regras de data

As regras são verificadas nesta ordem; a primeira que falhar define a mensagem (`400`).

| Regra | Detalhe | Mensagem quando falha |
|---|---|---|
| Data futura | `date` precisa ser **depois de hoje** (hoje não é aceito) | `A data de agendamento não pode ser menor que a data Atual` |
| Sem fim de semana | Sábado e domingo não são aceitos, para qualquer país | `Coleta não pode ser realizada em um final de semana` |
| Sem feriado nacional | Somente quando o remetente está no Brasil | `Coleta não pode ser realizada durante um feriado nacional` |
| Limite de 7 dias úteis | A data não pode passar do 7º dia útil depois de hoje (para remetente no Brasil, feriados nacionais não contam como dia útil) | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` |

Feriados nacionais brasileiros considerados (datas fixas): **01/01, 21/04, 01/05, 07/09, 12/10, 02/11,
15/11, 20/11 e 25/12**. Feriados móveis (ex.: Carnaval, Sexta-feira Santa, Corpus Christi) e feriados
estaduais/municipais **não** são bloqueados pela API — evite essas datas por conta própria.

### Ordem das validações

| # | Verificação | Status | Mensagem |
|---|---|---|---|
| 1 | Formato dos campos | `422` | Mensagens por campo em `data` |
| 2 | AWB localizado (se `awb_number` foi enviado) | `400` | `Agendamento não foi encontrado para esta remessa` |
| 3 | Data depois de hoje | `400` | `A data de agendamento não pode ser menor que a data Atual` |
| 4 | Remessa sem agendamento anterior | `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` |
| 5 | Remessa existente e com status atual `shipped` | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 6 | Transportador com coleta integrada | `400` | `Esta transportadora não possui serviço de coleta integrado` |
| 7 | Regras de data (fim de semana, feriado, 7 dias úteis) | `400` | Ver [Regras de data](#regras-de-data) |
| 8 | Remessa criada pelo usuário do token | `400` | `Remessa não está em pronta para realizar coleta ou não existe` |
| 9 | Endereço alternativo (se enviado) | `400` | Ver [Erros comuns](#erros-comuns) |

## Endpoint: consultar agendamento

- **Método e rota:** `GET /v1/schedule/{shipping}`
- **Quando usar:** para acompanhar o agendamento após o `POST` e antes de cancelar.

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `shipping` | path | string | **sim** | UUID da cotação ou número do AWB (remessas dos últimos 90 dias) |

### Payload mínimo

Sem corpo. Apenas o identificador no path:

```
GET /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944
```

### Payload máximo

Não há outros parâmetros. A alternativa é usar o AWB no lugar do UUID:

```
GET /v1/schedule/1234567890
```

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "confirmation_code": "000000",
    "date": "2026-03-10",
    "inicial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Retirar na recepcao",
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
| `data.confirmation_code` | string \| null | Número de confirmação da coleta no transportador. Só é definitivo quando `status.code` é `success`; antes disso pode vir `null` ou um valor provisório (ex.: `000000`) |
| `data.date` | string | Data da coleta (`AAAA-MM-DD`) |
| `data.inicial_time` | string | Início da janela (`HH:MM:SS`). ⚠️ **A chave é `inicial_time`** na resposta, enquanto no request o campo é `initial_time` |
| `data.final_time` | string | Fim da janela (`HH:MM:SS`) |
| `data.instruction` | string | Instrução informada na criação (ou a da predefinição, na coleta automática) |
| `data.status.code` | string | Código do status do agendamento (ver [Status do agendamento](#status-do-agendamento)). Use este campo na sua lógica |
| `data.status.description` | string | Texto informativo do último evento. **Não é traduzido na consulta**: pode vir no idioma em que o pedido foi feito, em inglês (ex.: `Pickup Scheduled Successfully!`, `Pickup Cancelled Successfully!`) ou com a mensagem devolvida pelo transportador, em caso de `error` |
| `data.status.reason` | string \| null | Código do motivo de cancelamento (`001`–`008`). `null` enquanto não houver pedido de cancelamento |

## Endpoint: cancelar agendamento

- **Método e rota:** `DELETE /v1/schedule/{shipping}/{reason}`
- **Quando usar:** quando a coleta já foi **confirmada** (status `success`) e precisa ser desmarcada.

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | UUID da cotação ou número do AWB (remessas dos últimos 90 dias) |
| `reason` | path | string | **sim** | `001` a `008` | Código do motivo (ver [Motivos de cancelamento](#motivos-de-cancelamento)). Repassado ao transportador |

### Payload mínimo

Sem corpo. Identificador e motivo no path:

```
DELETE /v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/006
```

### Payload máximo

Não há outros parâmetros. A alternativa é usar o AWB no lugar do UUID:

```
DELETE /v1/schedule/1234567890/001
```

### Exemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/schedule/9c06fd75-d5f4-40eb-a1a8-f12086f49944/001 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Operação de agendamento realizada com sucesso"
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` quando o pedido de cancelamento foi registrado |
| `message` | string | Mensagem traduzida conforme o header `lang` |

> Após o pedido, o status passa para `canceling` com a descrição
> `Solicitação de cancelamento recebida! Em breve será processada!`, depois `terminating` e, por fim,
> `canceled` (ou `error`). Consulte o agendamento para acompanhar.

### Regras do cancelamento

| Regra | Status | Mensagem quando falha |
|---|---|---|
| O `reason` precisa ser um dos códigos `001` a `008` | `422` | `O campo reason não contém um valor válido.` (em `data.reason`) |
| O AWB precisa ser localizado (se usado no path) | `400` | `Agendamento não foi encontrado para esta remessa` |
| O agendamento precisa existir | `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` |
| A remessa precisa continuar com status atual `shipped` | `400` | idem |
| O agendamento precisa estar em `success` (já confirmado). Em `processing`, `canceling`, `terminating`, `canceled` ou `error`, o cancelamento é recusado | `400` | idem |

## Regras de negócio

| Regra | Detalhe |
|---|---|
| Status da remessa | O status **atual** precisa ser `shipped`, tanto para agendar quanto para cancelar |
| Agendamento único | Cada remessa aceita um único agendamento pela API. Um agendamento cancelado ou com erro **também** impede um novo `POST`; para reagendar, acione o suporte |
| Transportador | O serviço da remessa precisa ter coleta integrada. Cancelamento integrado apenas para DHL, FedEx, UPS e SkyPostal |
| Data | Depois de hoje, dia útil, sem feriado nacional fixo (remetente BR), até 7 dias úteis |
| Janela | `initial_time` e `final_time` no formato `HH:MM:SS`; não há validação de ordem nem de horário comercial |
| Endereço alternativo | Todas as 12 chaves presentes; no Brasil exige `number`, `district` e `federal_tax_id` válido (CPF ou CNPJ) |
| País do endereço | Sempre o país do remetente da remessa. O `state_code` precisa pertencer a esse país |
| Tipo de conta | Pessoa física e PJ mono-CD: `address` ignorado; coleta no endereço do remetente |
| Coleta automática | Com a predefinição de coleta automática ativa, o agendamento é criado pela plataforma e o `POST` manual é recusado |
| Propriedade | Agende e consulte apenas remessas criadas pelo usuário do token |
| AWB | Resolve apenas remessas criadas nos **últimos 90 dias** (a mais recente com aquele número) |
| Processamento | Imediato para coleta amanhã; programado (enviado 1 dia antes) para datas posteriores |

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Já existe agendamento (manual, automático, cancelado ou com erro). Consulte com `GET`; para reagendar, acione o suporte |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Confirmar que o status atual é `shipped` e que a remessa é do usuário do token |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Levar o volume a um ponto de postagem |
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Usar uma data a partir de amanhã |
| `400` | `Coleta não pode ser realizada em um final de semana` | Escolher um dia útil |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Escolher outro dia útil |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Reduzir a antecedência |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Remetente BR: preencher `number` e `district`. Outros países: conferir `postal_code` ou enviar `state_code` |
| `400` | `O atributo federal_tax_id não é válido` | Remetente BR: informar um CPF ou CNPJ válido |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Usar um `state_code` do país do remetente |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Conferir o `postal_code` ou informar o `state_code` |
| `400` | `Agendamento não foi encontrado para esta remessa` | No `POST`/`DELETE` com AWB: conferir o número, a idade da remessa (90 dias) e se ela é da sua conta |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Só cancele agendamentos em `success` de remessas em `shipped` |
| `403` | `Has no permission!` | Solicitar ao suporte a permissão da operação (criar, consultar ou cancelar agendamento) |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Acionar o suporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Conferir o header `Authorization: Bearer ...` |
| `404` | `Agendamento não foi encontrado para esta remessa` | No `GET` com UUID: a remessa não tem agendamento ou não é da sua conta |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + mapa de campos em `data` | Corrigir os campos listados (ex.: `O campo data é obrigatório.`, `A data informada para o campo :attribute não respeita o formato :format.`, `O valor selecionado para o campo shipping é inválido.`, `O campo reason não contém um valor válido.`) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | `POST`: conferir se todas as chaves de `address` foram enviadas. `GET` com AWB: o AWB não foi localizado (inexistente, com mais de 90 dias ou de outra conta). Nos demais casos, repetir com backoff e acionar o suporte |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Falha inesperada no `DELETE`. Repetir com backoff; se persistir, acionar o suporte |

> Atenção: na **consulta por AWB** (`GET /v1/schedule/{awb}`), um AWB não localizado responde `500`, e não
> `404`. Prefira consultar pelo UUID da cotação, que responde `404` quando não há agendamento.

## Observações finais
- Sempre consulte o agendamento antes de criar ou cancelar: isso evita requisições redundantes e o erro
  `já tem um agendamento em progresso`.
- Atenção à diferença de nomenclatura: **`initial_time` no request** e **`inicial_time` na resposta**.
- Agendamentos programados permanecem em `processing` até 1 dia antes da coleta. Não é erro. Se o status
  não mudar depois da data da coleta, acione o suporte.
- Use `status.code` (e não `status.description`) para a lógica da sua integração.
- Status da remessa: [Rastreio](api-rastreio.md). Documentos para a coleta: [Documentos](api-documentos.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
