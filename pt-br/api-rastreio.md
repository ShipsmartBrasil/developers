# Rastreio de Remessa

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Consulta o status operacional e o histórico completo de uma remessa, do recebimento na plataforma até a
  entrega (ou o cancelamento).
- A mesma resposta traz **dois formatos**: uma `timeline` cronológica e os eventos **agrupados por status**.
- Funciona para remessas criadas na **v1 e na v2**.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
lang: pt
```

> As descrições de status (`status_description`) são traduzidas conforme o header `lang`.
> Os códigos (`status_code`) **não** mudam com o idioma — use-os para lógica de integração.

## Endpoints

### Status por remessa
- **Método e rota:** `GET /v1/shipping/{shipping}/status`

### Status por canal
- **Método e rota:** `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}`

## Parâmetros

| Nome | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `shipping` | path | string | **sim** | UUID da cotação, número do AWB (últimos 90 dias) ou chave da NF-e (44 dígitos) |
| `channel_identifier_number` | path | string | **sim** (variante por canal) | Identificador comercial informado na cotação |
| `channel_code` | path | integer | não | Código do canal de venda |

## Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/9c06fd75-d5f4-40eb-a1a8-f12086f49944/status \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

Por canal:
```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/shipping/status/PED-2026-0001/1 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'lang: pt'
```

## Exemplo de response

```json
{
  "status": "success",
  "message": "Status da remessa encontrado!",
  "data": {
    "carrier": {
      "name": "Fedex Economy Startup Test BR",
      "tracking_number": "271249639538",
      "timeline": [
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:38",
          "status_code": "RCV0001",
          "status_description": "Remessa foi recebida em nosso sistema"
        },
        {
          "update_date": "2026-02-21",
          "update_time": "15:42:40",
          "status_code": "POT0001",
          "status_description": "Pagamento aprovado junto ao gateway: approved"
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
          "status_code": "CLD0001",
          "status_description": "Remessa coletada pelo transportador"
        },
        {
          "update_date": "2026-02-23",
          "update_time": "04:35:11",
          "status_code": "ITR0001",
          "status_description": "Remessa em trânsito internacional"
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
    "paid_out": [
      {
        "update_date": "2026-02-21",
        "update_time": "15:42:40",
        "status_code": "POT0001",
        "status_description": "Pagamento aprovado junto ao gateway: approved"
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
        "status_code": "CLD0001",
        "status_description": "Remessa coletada pelo transportador"
      }
    ],
    "in_transit": [
      {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    ],
    "last_update": {
      "in_transit": {
        "update_date": "2026-02-23",
        "update_time": "04:35:11",
        "status_code": "ITR0001",
        "status_description": "Remessa em trânsito internacional"
      }
    }
  }
}
```

## Estrutura da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.carrier.name` | string | Nome do contrato/transportador que processou a remessa |
| `data.carrier.tracking_number` | string | Número de rastreio (AWB) no transportador |
| `data.carrier.timeline` | array | **Todos os eventos em ordem cronológica.** Use este campo para montar uma linha do tempo na sua interface |
| `data.cost_center` | object | **Opcional.** Só aparece quando a remessa tem centro de custo |
| `data.cost_center.code` | integer | Código do centro de custo |
| `data.cost_center.name` | string | Nome do centro de custo |
| `data.<grupo_de_status>` | array | Eventos agrupados por status. As chaves existentes variam conforme o andamento |
| `data.last_update` | object | **Último evento**, no formato `{ "<grupo>": { ...evento } }`. Use para saber o estado atual com uma única leitura |

Cada evento (em `timeline`, nos grupos e em `last_update`) tem a mesma forma:

| Campo | Tipo | Descrição |
|---|---|---|
| `update_date` | string | Data do evento (`Y-m-d`) |
| `update_time` | string | Hora do evento (`H:i:s`) |
| `status_code` | string | Código interno do status (ver tabela) ou o código bruto do gateway/transportador |
| `status_description` | string | Descrição legível, traduzida pelo header `lang` |

### Os dois formatos, e quando usar cada um

| Formato | Onde está | Use para |
|---|---|---|
| Cronológico | `data.carrier.timeline` | Exibir o histórico completo ao usuário final |
| Agrupado por status | `data.received`, `data.shipped`, `data.in_transit`... | Verificar se a remessa **já passou** por determinada etapa (`if (isset(data.shipped))`) |
| Último estado | `data.last_update` | Dashboards e sincronizações: uma leitura resolve "onde está a remessa" |

## Grupos de status e códigos

| Grupo (chave em `data`) | `status_code` | Significado |
|---|---|---|
| `received` | `RCV0001` | A remessa foi recebida na plataforma |
| `processing_payment` | `PPY0001` | Aguardando a confirmação do pagamento no gateway |
| `paid_out` | `POT0001` | Pagamento aprovado |
| `pending_consolidation` | `PCD0001` | Aguardando consolidação (contratos com consolidação) |
| `consolidating` | `CDT0001` | Em consolidação |
| `processing_shipment` | `PSH0001` | Pagamento resolvido, aguardando o processamento no transportador |
| `shipped` | `SPD0001` | **Processada no transportador. Documentos disponíveis** |
| `collecting` | `CLG0001` | Coleta em andamento |
| `collected` | `CLD0001` | Coletada pelo transportador |
| `in_transit` | `ITR0001` | Em trânsito |
| `delivering` | `DLG0001` | Em rota de entrega |
| `delivered` | `DLD0001` | Entregue |
| `error` | `ERR0001` | Erro no processamento |
| `cancelling` | `CCG0001` | Cancelamento solicitado, em processamento |
| `cancelled` | `CCD0001` | Cancelada |
| `foc` | `FOC0001` | Finalizada pelo transportador |
| `eol` | `EOL0001` | Fim do ciclo de vida do rastreio |

> **`status_code` pode trazer o código bruto do gateway ou do transportador** quando ele existe
> (ex.: `approved`, `charge.succeeded`, códigos próprios da transportadora). Os códigos `XXX0001` são o
> de/para interno da plataforma, usados quando não há código externo. Para lógica de negócio, prefira a
> **chave do grupo** (`shipped`, `delivered`, `cancelled`...) em vez do `status_code`.

## Fluxo típico de estados

```
received
   │
   ├── processing_payment ──► paid_out        (pagamento via gateway)
   │
   └── (pagamento billed: segue direto)
                │
                ├── pending_consolidation ──► consolidating   (quando há consolidação)
                │
                ▼
         processing_shipment
                │
                ▼
             shipped          ◄── documentos disponíveis a partir daqui
                │
                ▼
    collecting ──► collected ──► in_transit ──► delivering ──► delivered
                │
                └── error | cancelling ──► cancelled | foc | eol
```

## Regras de negócio

- **Toda remessa precisa ter um registro de pagamento** para que o status seja montado. Se não houver,
  a resposta é `404` com `Nenhuma entrada de pagamento foi registrada para esta remessa!`.
- Com `payment.type: billed`, a remessa não passa por `processing_payment`: ela segue direto para
  `processing_shipment`/`shipped`.
- Com `payment.type: gateway`, a remessa fica em `processing_payment` até o gateway aprovar; só então
  avança para `processing_shipment`.
- O grupo muda de `processing_shipment` para `shipped` no momento em que os **documentos são gerados**.
  Essa é a sinalização para chamar [Documentos de Remessa](api-documentos.md).
- Quando a remessa está em `cancelling`, `cancelled` ou `error`, o `last_update` usa a descrição própria
  da plataforma em vez do texto bruto do transportador.
- Remessas que ficam **29 dias ou mais** sem evento de rastreio em status elegível podem ser canceladas
  automaticamente pela plataforma.
- A busca por AWB funciona para remessas criadas nos **últimos 90 dias**.

## Boas práticas de polling

| Situação | Intervalo recomendado |
|---|---|
| Logo após criar a remessa (aguardando `shipped`) | 10 a 30 segundos, com backoff |
| Remessa já em `shipped`/`in_transit` | 1 a 4 horas |
| Remessa em `delivered`, `cancelled`, `foc` ou `eol` | Pare o polling: são estados finais |

1. Leia `data.last_update` para decidir se houve mudança — é a leitura mais barata.
2. Pare o polling nos estados finais.
3. Use backoff exponencial em `500`.
4. Trate `400`/`404` como condição de negócio: não repita a mesma requisição em loop.

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Conferir `{shipping}` ou o identificador do canal |
| `403` | Sem permissão para consultar status | Solicitar a permissão ao suporte |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Aguardar o registro do pagamento; se persistir, acionar o suporte |
| `422` | `channel_code` não numérico | Enviar um inteiro em `channel_code` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- `data.carrier.timeline` é a visão mais completa; os grupos são um resumo operacional para facilitar a
  leitura programática.
- Guarde o `tracking_number` para oferecer ao cliente final o rastreio no site do transportador.
- Para baixar etiqueta e invoice assim que a remessa atingir `shipped`, veja
  [Documentos de Remessa](api-documentos.md).
