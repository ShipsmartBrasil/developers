# Primeiros Passos

- [Voltar ao README do idioma](README.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
Esta página reúne tudo que é comum a **todos** os endpoints da API Pública da Shipsmart Global:
ambientes, autenticação, idioma, formato de resposta, versionamento e tratamento de erros.

Leia esta página antes de qualquer outra. As demais páginas do guia assumem que você já conhece
estas convenções e não as repetem em detalhe.

## Ambientes

| Ambiente | Base URL | Documentação interativa |
|---|---|---|
| Produção | `https://api.shipsmart.global` | `https://api.shipsmart.global/doc` |
| Homologação (staging) | `https://hmlapi.shipsmart.com.br` | `https://hmlapi.shipsmart.com.br/doc` |

- Use **homologação** para desenvolver e testar a integração. Nenhuma remessa criada lá é despachada de verdade.
- Features novas aparecem primeiro em homologação. Se um campo descrito aqui ainda não existir em produção,
  valide em `https://hmlapi.shipsmart.com.br/doc`.
- Acessar a raiz (`/`) redireciona para `/doc`.

## Autenticação

Toda chamada (exceto a documentação) exige um token de acesso pessoal no header `Authorization`,
no esquema **Bearer**.

```
Authorization: Bearer SEU_TOKEN
```

| Item | Valor |
|---|---|
| Tipo | Bearer token |
| Header | `Authorization` |
| Onde obter | Painel da Shipsmart Global, na área de configurações da sua conta |
| Expiração | O token não expira por tempo; ele é revogado se a conta for bloqueada ou o token regenerado |

### Erros de autenticação

| Status | Resposta | Significado |
|---|---|---|
| `404` | `{"status":"warning","message":"Nenhum usuário encontrado para o token especificado!"}` | Token ausente, não encontrado ou inválido |
| `403` | `{"status":"warning","message":"Usuário especificado encontra-se bloqueado em nosso sistema!"}` | Conta bloqueada |
| `403` | `{"status":"warning","message":"Has no permission!"}` | O token é válido, mas o seu perfil não tem permissão para a ação |

> As permissões são granulares por ação (cotar, remeter, cancelar, listar documentos, agendar coleta etc.).
> Se você receber `Has no permission!`, solicite a liberação da permissão correspondente ao suporte.

## Idioma das respostas

As mensagens de sucesso, aviso e erro são traduzidas conforme o header `lang`.

| Header | Valores aceitos | Padrão |
|---|---|---|
| `lang` | `pt`, `en`, `es` | `pt` |

```
lang: en
```

> O header é opcional. Se não for enviado, a API responde em português. Se for enviado com um valor
> não suportado (ex.: `fr`), as mensagens vêm em inglês.

> **Sobre os exemplos deste guia:** todas as mensagens de resposta reproduzidas nas páginas seguintes
> usam o idioma padrão da API (`pt`). Enviando `lang: en` ou `lang: es`, o texto de `message` vem
> traduzido, mas o `status` e o código HTTP são sempre os mesmos.

## Convenções de requisição

| Item | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Codificação | UTF-8 |
| Datas (resposta) | `AAAA-MM-DD HH:MM:SS` |
| Datas (agendamento) | `AAAA-MM-DD` para data e `HH:MM:SS` para hora |
| Moedas | Código ISO 4217 com 3 letras (ex.: `BRL`, `USD`, `EUR`) |
| Países | Código ISO 3166-1 alpha-2 com 2 letras (ex.: `BR`, `US`, `PT`) |
| Separador decimal | Ponto (`.`) — nunca vírgula |
| Pesos | Quilogramas quando `measurement` é `metric`; libras quando é `imperial` |
| Dimensões | Centímetros quando `measurement` é `metric`; polegadas quando é `imperial` |
| Volume | m³ quando `measurement` é `metric`; ft³ quando é `imperial` |

## Versionamento

A versão faz parte do path: `/v1/...` e `/v2/...`.

- `POST /v1/quotation` → cotação na primeira versão
- `POST /v2/quotation` → cotação na segunda versão

### Chamadas sem prefixo de versão
Se você chamar um recurso **sem** o prefixo de versão, a API responde `308 Permanent Redirect`
apontando para o mesmo recurso em `/v1`, preservando a query string.

```
GET /quotation  →  308  →  GET /v1/quotation
```

> Para integrações novas, sempre informe a versão explicitamente. Não dependa do redirecionamento.

### Compatibilidade entre versões
Uma cotação criada na v1 **só** pode gerar remessa na v1, e uma cotação criada na v2 **só** pode gerar
remessa na v2. Misturar versões retorna `422` com a mensagem (em `data.quotation`)
`Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.`

Veja [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md) para escolher a versão.

## Formato padrão das respostas

Todas as respostas seguem o mesmo envelope:

```json
{
  "status": "success",
  "message": "Mensagem legível para humanos",
  "data": {}
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success`, `warning` ou `error` |
| `message` | string | Mensagem já traduzida conforme o header `lang` |
| `data` | object/array | Conteúdo da resposta. Ausente em respostas que só confirmam o recebimento |

### Relação entre `status` e HTTP

| `status` | HTTP típico | Leitura |
|---|---|---|
| `success` | `200` | A operação foi aceita/concluída |
| `warning` | `400`, `401`, `403`, `404`, `406`, `422` | Regra de negócio ou validação impediu a operação. **O payload precisa ser corrigido** |
| `error` | `500` (e algumas recusas de peso/transportador com `400`/`404`) | Falha inesperada ou recusa do transportador. Leia `message`; erros `500` podem ser repetidos |

## Erros de validação (`422`)

Quando um campo não passa na validação, a resposta traz `data` com o mapa `campo → lista de mensagens`.
A `message` de topo é **genérica** (não descreve o erro): use sempre o conteúdo de `data`.

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "object": ["O campo object é obrigatório."],
    "address_sender.country_code": [
      "O campo address sender.country code é obrigatório quando address sender.address code não está presente."
    ],
    "boxes": ["O campo boxes é obrigatório a menos que type esteja presente em items."]
  }
}
```

> Em campos aninhados, a chave usa ponto (`address_sender.country_code`) e, em arrays,
> usa o índice (`items.0.hscode`). Use isso para apontar o erro no seu formulário/integração.

## Tabela geral de status HTTP

| Status | Quando acontece | O que fazer |
|---|---|---|
| `200` | Requisição aceita/processada | Seguir o fluxo |
| `308` | Chamada sem prefixo de versão | Usar `/v1/...` ou `/v2/...` |
| `400` | Regra de negócio bloqueou a operação (ex.: documento fiscal inválido, caixa sem item, CEP inválido) | Ler `message` e corrigir o cenário |
| `401` | Tipo de pagamento não habilitado para a conta | Habilitar a opção de pagamento ou usar outro tipo |
| `403` | Sem permissão, conta bloqueada, cotação simples ou expirada na remessa | Solicitar liberação ao suporte / refazer a cotação |
| `404` | Recurso não encontrado (token, cotação de outra conta, remessa, contrato não listado na cotação, endereço) | Conferir o identificador informado |
| `406` | Conflito de negócio (cotação já usada, endereço cadastrado incompatível, carga que exige atendimento personalizado) | Ler `message` e ajustar o fluxo |
| `422` | Falha de validação de campos, ou regra de negócio sobre dados (ex.: HS Code sem tabela de imposto, conta só DDP) | Corrigir os campos listados em `data` / ler `message` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Predefinições da conta (presets)

Algumas opções podem ficar **pré-configuradas na sua conta** no painel. Elas são aplicadas pela API depois da
validação do payload, com regras diferentes por campo:

| Predefinição | Afeta | Como é aplicada |
|---|---|---|
| Imposto (`tax`) | Cotação, Manifesto, Rascunho | **Substitui** o `tax` enviado. Países de destino configurados como exceção recebem o oposto do padrão. Se a conta estiver **travada em DDP**, o imposto é sempre `sender` |
| Seguro automático | Cotação, Manifesto | Contrata o seguro quando o valor declarado (convertido para USD) atinge o mínimo configurado, mesmo com `insurance: false` |
| Canal de venda (`channel_code`) | Cotação, Manifesto, Rascunho | Usado quando o campo não é enviado |
| Sistema de medida (`measurement`) | Cotação, Manifesto, Rascunho | Usado quando o campo não é enviado; na falta dele, `metric` |
| Entrega residencial / não empilhável | Cotação, Rascunho | Usados quando os campos não são enviados |
| Frete declarado (`declared_freight`) | Cotação, Manifesto, Rascunho | Usado quando o campo não é enviado. Exige a permissão de frete declarado |
| Agrupamento de caixas | Cotação v2 (`type: items`) | Define como as caixas são montadas (ver abaixo). Não é um campo do payload |
| Centro de custo (`cost_center_code`) | Remessa v1/v2, Manifesto, Rascunho | Usado quando o campo não é enviado **ou é enviado vazio/nulo** |
| Finalidade do envio (`type` da remessa) | Remessa v1/v2, Rascunho | Usada quando o campo não é enviado |
| Faturante (`biller`) | Remessa v1/v2, Manifesto, Rascunho | Usado quando o campo não é enviado (ou vem vazio) e a conta está configurada para exibir faturante |
| Notificação de cobrança (`payment_notify_shipping_billing`) | Remessa v1/v2 | Usada quando o campo não é enviado |
| Coleta automática | Remessa | Cria o agendamento de coleta automaticamente (ver [Agendamento de Coleta](api-agendamento.md)) |

**Precedência:** exceto pelo **imposto** (que sempre segue a predefinição, quando configurada), o que você envia
no payload vence a predefinição. Campos obrigatórios do payload (`object`, `type`, `insurance`, moedas,
`address_sender`) precisam ser enviados sempre — a predefinição não os dispensa.

> O endereço de remetente pré-configurado no painel **não** substitui o `address_sender` da API.
> Para reaproveitar um endereço cadastrado, envie `address_sender.address_code`.

### Agrupamento de caixas
O agrupamento define como os itens são distribuídos nas caixas quando você usa `type: items`
(cotação por itens, **exclusiva da v2**).
Ele **não** é um campo do payload: é uma configuração da sua conta.

| Modo | Comportamento |
|---|---|
| `default` | Cada item usa a caixa vinculada ao seu cadastro (`box_code`/`box_identifier`); itens agrupáveis com a mesma caixa são reunidos e itens não agrupáveis recebem uma caixa por unidade |
| `all_in_one` | Consolida os itens agrupáveis em uma única caixa virtual com o volume total (a partir das dimensões cadastradas); itens não agrupáveis recebem uma caixa por unidade |
| `smartbox` | Encaixa os itens nas caixas **enviadas em `boxes[]`** (obrigatório), considerando volume e dimensões com 10% de folga e respeitando `is_groupable`/`is_foldable` do item |

Exemplos de cada modo em [Cotação de Itens](api-cotacao-v2.md#cotação-de-itens).

## Cadastros feitos no painel

Alguns códigos usados nos payloads vêm de cadastros mantidos **no painel da Shipsmart Global** (não há endpoint
público para criá-los). Eles precisam existir, estar ativos e pertencer à sua conta (ou às contas autorizadas):

| Cadastro | Campo no payload | Onde é usado |
|---|---|---|
| Endereço | `address_sender.address_code`, `address_receiver.address_code`, `manufacturer.address_code`, `biller.address_code` | Cotação, Remessa, Manifesto, Rascunho |
| Canal de venda | `channel_code` | Cotação, Manifesto, Rascunho, consultas e cancelamento por canal |
| Centro de custo | `cost_center_code` | Remessa (criação e atualização), Manifesto, Rascunho |
| Kit | `boxes[].kit` | Cotação, Manifesto, Rascunho |

Caixas e itens também podem ser cadastrados pela API: veja [Caixa](api-caixa-v1.md) e [Itens](api-itens-v1.md).

## Moedas aceitas

Mesma lista para `currency_quote` (moeda da cotação) e `currency_payment` (moeda do pagamento):

`USD`, `CAD`, `NZD`, `TWD`, `HKD`, `AUD`, `SGD`, `BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `BOB`,
`PEN`, `PYG`, `EUR`, `CNY`, `GBP`, `JPY`, `CHF`, `THB`, `DKK`, `NOK`, `SEK`, `AED`, `TRY`, `ILS`,
`ZAR`, `SAR`, `RUB`, `INR`, `PLN`

> `currency_payment` com pagamento via gateway tem disponibilidade dependente do provedor:
> `BRL` é processado por um provedor e as demais moedas por outro. Se a moeda escolhida não estiver
> habilitada para a sua conta, a remessa retorna `warning` no momento do pagamento.

## Primeira chamada (teste de conectividade)

O endpoint mais simples para validar token e conectividade:

```bash
curl --request GET \
  --url https://hmlapi.shipsmart.com.br/v1/platform/version \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Accept: application/json'
```

Resposta:

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "staging",
    "version": "1.0.0"
  }
}
```

## Boas práticas de integração

1. **Sempre guarde o `quotation`** retornado na cotação: ele é a chave de todo o restante do fluxo
   (remessa, status, documentos, agendamento).
2. **Grave também o seu identificador comercial** em `channel_identifier_number`. Isso permite consultar
   status, documentos e cancelar usando o número do seu pedido, sem depender do `quotation`.
3. **Nunca faça polling agressivo.** Consultas de status/documentos devem ter intervalo de alguns
   segundos e backoff exponencial.
4. **Trate `warning` como erro de negócio** (não repita a mesma requisição) e `error` como transitório
   (pode repetir).
5. **Respeite os limites de tamanho de campo** descritos em cada página. Textos maiores são rejeitados
   com `422`, não truncados.
6. **Cotações expiram.** Crie a remessa dentro da validade informada em `valid_until` de cada transportador.

## Próximos passos

- [Cotação](api-cotacao-v1.md) ou [Cotação V2](api-cotacao-v2.md)
- [Remessa](api-remessa-v1.md) ou [Remessa V2](api-remessa-v2.md)
- [Cotação & Remessa: v1 vs v2](api-comparativo-cotacao-remessa.md)
- [Códigos de Erro](api-codigos-de-erro.md)
