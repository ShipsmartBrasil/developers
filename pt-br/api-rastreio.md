# Rastreio de Remessa

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Consulta o status operacional e o histórico de uma remessa, do recebimento na plataforma até a
  entrega (ou o cancelamento).
- A mesma resposta traz **três leituras**: a `timeline` cronológica, os eventos **agrupados por status** e o
  **último estado** (`last_update`).
- Há duas formas de consultar: pelo identificador da remessa (`{shipping}`) ou pelo identificador do seu
  canal de venda (`channel_identifier_number` + `channel_code`).
- Funciona para remessas criadas na **v1 e na v2** (as rotas de rastreio existem apenas na v1).

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
lang: pt
```

| Header | Obrigatório | Valores | Padrão | Observação |
|---|---|---|---|---|
| `Authorization` | **sim** | `Bearer <token>` | - | Token da sua conta. O perfil precisa da permissão de consulta de status de remessa |
| `lang` | não | `pt`, `en`, `es` | `pt` | Idioma das mensagens e das descrições geradas pela plataforma |

> O header `lang` traduz a `message` da resposta e as descrições dos eventos gerados pela **plataforma**
> (recebimento, pagamento, processamento, cancelamento e erro). As descrições dos eventos vindos do
> **transportador** são devolvidas como o transportador as enviou e **não** mudam com o idioma.
> Os códigos (`status_code`) e as chaves dos grupos nunca mudam com o idioma — use-os na lógica da integração.

## Fluxo de uso
1. Crie a remessa ([Remessa v1](api-remessa-v1.md) ou [Remessa v2](api-remessa-v2.md)) e guarde o `quotation`
   (UUID) usado na criação.
2. Consulte o status por `{shipping}` (recomendado) ou pelo identificador do canal.
3. Quando o grupo `shipped` aparecer, os documentos já existem: baixe-os em [Documentos de Remessa](api-documentos.md).
4. Continue consultando em intervalos maiores para acompanhar coleta, trânsito e entrega.
5. Pare de consultar quando `last_update` indicar um estado final (veja [Boas práticas de polling](#boas-práticas-de-polling)).

```
criar remessa ──► GET .../status ──► (shipped?) ──► GET .../docs
                        │                 │
                        │ não             │ sim
                        ▼                 ▼
                  aguardar e repetir   acompanhar coleta → trânsito → entrega
```

## Conceitos essenciais

### As três leituras da resposta

| Leitura | Onde está | O que contém | Use para |
|---|---|---|---|
| Cronológica | `data.carrier.timeline` | Todos os eventos montados pela plataforma e todos os eventos de rastreio do transportador, na ordem em que foram registrados | Exibir o histórico completo ao usuário final |
| Agrupada por status | `data.received`, `data.shipped`, `data.in_transit`... | Uma chave por status já alcançado. Cada chave é uma lista com **um único evento**: o mais recente daquele status | Verificar se a remessa **já passou** por determinada etapa |
| Último estado | `data.last_update` | O estado atual, no formato `{ "<grupo>": { ...evento } }` | Dashboards e sincronizações: uma leitura resolve "onde está a remessa" |

> **Atenção:** os grupos não guardam o histórico. Se o transportador enviar três eventos `in_transit`, o grupo
> `data.in_transit` trará apenas o último; os três aparecem em `data.carrier.timeline`.

### Grupos de status e códigos

| Grupo (chave) | `status_code` da plataforma | Significado | Onde pode aparecer |
|---|---|---|---|
| `received` | `RCV0001` | A remessa foi recebida na plataforma | Grupo e `timeline` (sempre presente) |
| `processing_payment` | `PPY0001` | Pagamento ainda não aprovado (também é o evento exibido para pagamento faturado, `billed`) | Grupo e `timeline` |
| `paid_out` | `POT0001` | Pagamento aprovado no meio de pagamento | Grupo e `timeline` |
| `pending_consolidation` | `PCD0001` | Aguardando consolidação (contratos com consolidação) | Reservado (ver [Regras de negócio](#regras-de-negócio)) |
| `consolidating` | `CDT0001` | Em consolidação | Reservado (ver [Regras de negócio](#regras-de-negócio)) |
| `processing_shipment` | `PSH0001` | Pagamento resolvido, aguardando o processamento no transportador | Grupo e `timeline` |
| `shipped` | `SPD0001` | **Processada no transportador. Documentos disponíveis** | Grupo e `timeline` |
| `collecting` | `CLG0001` | Transportador a caminho da coleta | Grupo, `timeline` e `last_update` |
| `collected` | `CLD0001` | Coletada pelo transportador | Grupo, `timeline` e `last_update` |
| `in_transit` | `ITR0001` | Em trânsito | Grupo, `timeline` e `last_update` |
| `delivering` | `DLG0001` | Saiu para entrega | Grupo, `timeline` e `last_update` |
| `delivered` | `DLD0001` | Entregue | Grupo, `timeline` e `last_update` |
| `error` | `ERR0001` | Não foi possível concluir o embarque | Somente `last_update` |
| `cancelling` | `CCG0001` | Cancelamento solicitado, em processamento | Somente `last_update` |
| `cancelled` | `CCD0001` | Cancelada | Somente `last_update` |
| `foc` | `FOC0001` | Finalizada pelo transportador | Somente `last_update` |
| `eol` | `EOL0001` | Fim do ciclo de vida do rastreio | Somente `last_update` |

> **`status_code` nem sempre é um código `XXX0001`.** Ele traz o código bruto da origem do evento quando
> esse código existe:
>
> - no evento de pagamento: o status informado pelo meio de pagamento (ex.: `approved`);
> - nos eventos de rastreio: o código próprio do transportador.
>
> Os códigos `XXX0001` são usados quando não há código externo. Para lógica de negócio, use a **chave do
> grupo** (`shipped`, `in_transit`, `delivered`, `cancelled`...) em vez do `status_code`.

### Descrições geradas pela plataforma

Mensagens literais em `pt` (mudam conforme `lang`):

| Grupo | `status_description` |
|---|---|
| `received` | `Remessa foi recebida em nosso sistema` |
| `processing_payment` | `Aguardando a finalização do processamento do pagamento junto ao gateway: :description ` |
| `paid_out` | `Pagamento aprovado junto ao gateway: :description ` |
| `processing_shipment` | `Remessa esta sendo processada no transportador especificado!` |
| `shipped` | `Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!` |
| `cancelling` (em `last_update`) | `Cancelando Remessa` |
| `cancelled` (em `last_update`) | `Remessa Cancelada` |
| `error` (em `last_update`) | `Não foi possível concluir o embarque desta remessa.` |

> Nas mensagens de pagamento, `:description` é a descrição enviada pelo meio de pagamento ou, na falta dela,
> o próprio código da plataforma (`PPY0001` / `POT0001`). As duas mensagens terminam com um **espaço** —
> considere isso se comparar strings (o recomendado é não comparar descrições).

### Fluxo típico de estados

```
received
   │
   ├── processing_payment ──► paid_out          (pagamento via gateway)
   │
   ├── processing_payment                       (pagamento billed: segue direto)
   │
   ▼
processing_shipment
   │
   ▼
shipped                 ◄── documentos disponíveis a partir daqui
   │
   ▼
collecting ──► collected ──► in_transit ──► delivering ──► delivered

Estados exibidos somente em last_update:
   error | cancelling ──► cancelled | foc | eol
```

## Endpoint: consultar status por remessa

- **Método e rota:** `GET /v1/shipping/{shipping}/status`
- **Quando usar:** é a forma recomendada. Use o UUID da cotação que você enviou como `quotation` na criação da remessa.

### Parâmetros

| Nome | Local | Tipo | Obrigatório | Tamanho | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `shipping` | path | string | **sim** | - | - | **Forma A:** UUID da cotação usada na remessa. **Forma B:** número de rastreio (AWB) da remessa |

> **Identificadores aceitos nesta rota:** UUID da cotação ou AWB. Se o AWB estiver em mais de uma remessa,
> é considerada a mais recente. **A chave da NF-e não é aceita nesta rota** (a consulta responde
> `400`), embora seja aceita na rota de documentos.

### Payload mínimo

Endpoint sem corpo. Basta o parâmetro de path:

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
```

### Payload máximo

Não há parâmetros de query. A forma máxima é a mesma da mínima, variando apenas o identificador:

```
GET /v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status
GET /v1/shipping/1234567890123/status
```

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK` — remessa com pagamento faturado (`billed`), já processada e em trânsito:

```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Contrato Exemplo Expresso",
      "tracking_number": "1234567890123",
      "timeline": [
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "PPY0001",
          "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:44:38",
          "status_code": "SPD0001",
          "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
        },
        {
          "update_date": "2026-02-22",
          "update_time": "09:10:02",
          "status_code": "PU",
          "status_description": "Shipment picked up"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "IT",
          "status_description": "In transit"
        }
      ]
    },
    "cost_center": {
      "code": 1,
      "name": "E-commerce"
    },
    "received": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "RCV0001",
        "status_description": "Remessa foi recebida em nosso sistema"
      }
    ],
    "processing_payment": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:38",
        "status_code": "PPY0001",
        "status_description": "Aguardando a finalização do processamento do pagamento junto ao gateway: PPY0001 "
      }
    ],
    "shipped": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:44:38",
        "status_code": "SPD0001",
        "status_description": "Remessa foi processada no transportador especificado. Documentos gerados e já disponíveis para download via portal ou API!"
      }
    ],
    "collected": [
      {
        "update_date": "2026-02-22",
        "update_time": "09:10:02",
        "status_code": "PU",
        "status_description": "Shipment picked up"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "IT",
        "status_description": "In transit"
      }
    }
  }
}
```

`200 OK` — trecho de `last_update` de uma remessa cancelada (substitui o último evento do transportador):

```json
{
  "last_update": {
    "cancelled": {
      "update_date": "2026-02-24",
      "update_time": "10:00:00",
      "status_code": "CCD0001",
      "status_description": "Remessa Cancelada"
    }
  }
}
```

`400 Bad Request` — remessa de outra conta:

```json
{
  "status": "warning",
  "message": "Nenhuma remessa encontrada com o código especificado!"
}
```

### Campos da resposta

| Campo | Tipo | Presença | Descrição |
|---|---|---|---|
| `status` | string | sempre | `success`, `warning` ou `error` |
| `message` | string | sempre | Mensagem traduzida pelo header `lang` |
| `data.carrier.name` | string | sempre | Nome do contrato que processou a remessa ou, sem contrato, o nome do transportador |
| `data.carrier.tracking_number` | string \| null | sempre | Número de rastreio (AWB) no transportador. Pode vir `null`/vazio antes do processamento |
| `data.carrier.timeline` | array | sempre | Todos os eventos, na ordem de registro: recebimento, pagamento, processamento e, depois, os eventos de rastreio do transportador |
| `data.cost_center` | object | opcional | Só aparece quando a remessa tem centro de custo |
| `data.cost_center.code` | integer | opcional | Código do centro de custo |
| `data.cost_center.name` | string \| null | opcional | Nome do centro de custo |
| `data.received` | array | sempre | Evento de recebimento (um item) |
| `data.processing_payment` ou `data.paid_out` | array | sempre um dos dois | Evento de pagamento (um item) |
| `data.processing_shipment` ou `data.shipped` | array | após pagamento aprovado ou faturado | Evento de processamento no transportador (um item) |
| `data.collecting`, `data.collected`, `data.in_transit`, `data.delivering`, `data.delivered` | array | conforme o rastreio | Último evento do transportador naquele status (um item) |
| `data.last_update` | object | opcional | Estado atual. **Só aparece** depois do primeiro evento de rastreio do transportador ou quando a remessa está em `cancelling`, `cancelled` ou `error` |

Cada evento (em `timeline`, nos grupos e em `last_update`) tem a mesma forma:

| Campo | Tipo | Formato | Descrição |
|---|---|---|---|
| `update_date` | string | `AAAA-MM-DD` | Data do evento |
| `update_time` | string | `HH:MM:SS` | Hora do evento. **Exceção:** no evento `processing_shipment`, este campo vem atualmente com a data (`AAAA-MM-DD`) em vez da hora |
| `status_code` | string | - | Código da plataforma (`XXX0001`) ou código bruto do meio de pagamento/transportador |
| `status_description` | string | - | Descrição legível. Traduzida por `lang` apenas nos eventos gerados pela plataforma |

## Endpoint: consultar status por canal

- **Método e rota:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`
- **Quando usar:** quando você guardou apenas o identificador do pedido no seu canal de venda (informado na cotação),
  e não o UUID da cotação.

### Parâmetros

| Nome | Local | Tipo | Obrigatório | Tamanho | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `channel_identifier_number` | path | string | **sim** | - | - | Identificador do pedido no canal, exatamente como informado na cotação |
| `channel_code` | path | integer | não | - | ausente | Código do canal de venda informado na cotação. Deve ser um número inteiro |

> **Como a busca funciona:**
>
> - **Com `channel_code`:** busca a remessa cuja cotação tem esse identificador **e** esse código de canal.
> - **Sem `channel_code`:** busca **apenas** remessas cuja cotação foi criada **sem** código de canal.
>   Se a sua cotação tinha `channel_code`, você precisa enviá-lo.
> - Remessas **canceladas não são consideradas**. Se houver mais de uma remessa válida, vale a mais recente.
> - Para omitir o `channel_code`, basta não enviar o último segmento. O valor `,` também é tratado como ausente.

### Payload mínimo

```
GET /v1/shipping/status/PED-2026-0001
```

### Payload máximo

```
GET /v1/shipping/status/PED-2026-0001/1
```

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK` — mesma estrutura de [consultar status por remessa](#endpoint-consultar-status-por-remessa).

`422 Unprocessable Entity` — `channel_code` não numérico:

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "channel_code": [
      "O campo channel code deve conter um número inteiro."
    ]
  }
}
```

### Campos da resposta

Idênticos aos de [consultar status por remessa](#campos-da-resposta).

## Regras de negócio

- **Somente remessas da sua conta.** A remessa precisa ter sido criada pela conta do token. Remessas de outra
  conta respondem `400` com `Nenhuma remessa encontrada com o código especificado!`.
- **Toda remessa precisa ter um registro de pagamento** para que o status seja montado. Se não houver,
  a resposta é `404` com `Nenhuma entrada de pagamento foi registrada para esta remessa!` (mensagem fixa,
  não muda com `lang`).
- **Pagamento e grupo exibido:**

  | `payment.type` da remessa | Evento de pagamento | Evento de processamento |
  |---|---|---|
  | `billed` | `processing_payment` com `PPY0001` | `processing_shipment` → `shipped` |
  | `gateway` | `processing_payment` (código do meio de pagamento) até a aprovação; depois `paid_out` | Só aparece após `paid_out` |
  | `afterward` | `processing_payment` até o pagamento ser aprovado | Só aparece após a aprovação do pagamento |

  Com `afterward`, a remessa pode já ter documentos sem que o grupo `shipped` apareça. Nesse caso, consulte
  [Documentos de Remessa](api-documentos.md) diretamente.
- O grupo muda de `processing_shipment` para `shipped` no momento em que os **documentos são gerados**; a data
  e a hora de `shipped` são as da geração do primeiro documento.
- Os grupos de rastreio (`collecting` a `delivered`) só aparecem depois que o transportador começa a enviar eventos.
- Quando a remessa está em `cancelling`, `cancelled` ou `error`, o `last_update` usa a descrição própria
  da plataforma (tabela [Descrições geradas pela plataforma](#descrições-geradas-pela-plataforma)) em vez do texto
  bruto do transportador.
- **Consolidação:** enquanto a remessa aguarda consolidação ou está em consolidação, a consulta de status pode
  responder `500`. Aguarde a conclusão da consolidação e consulte novamente.
- **Cancelamento automático:** remessas que, **29 dias após a criação**, ainda não têm número de rastreio no
  transportador (e não passaram de `shipped`) podem ser canceladas automaticamente pela plataforma.
- **Fim do ciclo de vida:** remessas com mais de **2 meses** que não chegaram a um estado final
  (`delivered`, `cancelled`, `foc` ou `eol`) são encerradas pela plataforma e deixam de receber atualizações.
- A busca por AWB nesta rota considera a remessa mais recente com aquele número.
- A consulta por canal **ignora remessas canceladas**: para acompanhar um cancelamento, consulte por `{shipping}`.

### Boas práticas de polling

| Situação | Intervalo recomendado |
|---|---|
| Logo após criar a remessa (aguardando `shipped`) | 10 a 30 segundos, com backoff |
| Remessa já em `shipped`/`in_transit` | 1 a 4 horas |
| `last_update` em `delivered`, `cancelled`, `foc` ou `eol` | Pare o polling: são estados finais |
| Remessa com mais de 2 meses | Pare o polling: a remessa não recebe mais atualizações |

1. Leia `data.last_update` para decidir se houve mudança — é a leitura mais simples. Lembre-se de que ela só
   existe após o primeiro evento do transportador (ou em cancelamento/erro); antes disso, use os grupos.
2. Pare o polling nos estados finais.
3. Use backoff exponencial em `500`.
4. Trate `400`/`404` como condição de negócio: não repita a mesma requisição em loop.

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `400` | `remessa.status.status_not_found: ` — nenhuma remessa encontrada pelo UUID nem pelo AWB (consulta por `{shipping}`; a mensagem vem nesse formato, sem tradução). Também ocorre ao enviar uma chave de NF-e | Conferir o `{shipping}`. Usar o UUID da cotação ou o AWB |
| `400` | `Nenhuma remessa encontrada com o código especificado!` — remessa de outra conta (por `{shipping}`) ou nenhuma remessa encontrada (por canal) | Conferir o identificador, o `channel_code` e o token usado |
| `403` | `Has no permission!` | Solicitar ao suporte a permissão de consulta de status |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Conta bloqueada: acionar o suporte |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Conferir o token enviado em `Authorization` |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Aguardar o registro do pagamento; se persistir, acionar o suporte |
| `422` | `O campo channel code deve conter um número inteiro.` (em `data.channel_code`) | Enviar um inteiro em `channel_code` ou omitir o segmento |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- `data.carrier.timeline` é a visão mais completa; os grupos são um resumo operacional para facilitar a
  leitura programática.
- Guarde o `tracking_number` para oferecer ao cliente final o rastreio no site do transportador.
- Para baixar etiqueta e invoice assim que a remessa atingir `shipped`, veja
  [Documentos de Remessa](api-documentos.md).
- Para criar e cancelar remessas, veja [Remessa (v1)](api-remessa-v1.md) e [Remessa (v2)](api-remessa-v2.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
