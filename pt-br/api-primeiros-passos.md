# Primeiros Passos

- [Voltar ao README do idioma](README.md)

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
| `404` | `{"status":"warning","message":"..."}` | Token não encontrado/inválido |
| `403` | `{"status":"warning","message":"..."}` | Conta bloqueada |
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

> O header é opcional. Se não enviado (ou enviado com valor desconhecido), a API responde em português.

> **Sobre os exemplos deste guia:** todas as mensagens de resposta reproduzidas nas páginas seguintes
> usam o idioma padrão da API (`pt`). Enviando `lang: en` ou `lang: es`, o texto de `message` vem
> traduzido, mas o `status` e o código HTTP são sempre os mesmos.

## Convenções de requisição

| Item | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |
| Codificação | UTF-8 |
| Datas (resposta) | `Y-m-d H:i:s` |
| Datas (agendamento) | `Y-m-d` para data e `H:i:s` para hora |
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
remessa na v2. Misturar versões retorna `422` com a mensagem
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
| `warning` | `400`, `403`, `404`, `406`, `422` | Regra de negócio ou validação impediu a operação. **O payload precisa ser corrigido** |
| `error` | `500` | Falha inesperada do lado do servidor. Tente novamente; se persistir, acione o suporte |

## Erros de validação (`422`)

Quando um campo não passa na validação, a resposta traz `data` com o mapa `campo → lista de mensagens`.

```json
{
  "status": "warning",
  "message": "Validation Error",
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
| `400` | Regra de negócio bloqueou a operação (ex.: tipo de pagamento inexistente, cotação expirada, nenhum transportador elegível) | Ler `message` e corrigir o cenário |
| `403` | Sem permissão ou conta bloqueada | Solicitar liberação ao suporte |
| `404` | Recurso não encontrado (token, remessa, contrato não listado na cotação) | Conferir o identificador informado |
| `406` | Estrutura do payload incompatível com o esperado | Revisar tipos e aninhamento dos campos |
| `422` | Falha de validação de campos | Corrigir os campos listados em `data` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Predefinições da conta (presets)

Algumas opções podem ficar **pré-configuradas na sua conta** no painel. Quando configuradas, elas são
aplicadas automaticamente na cotação e na remessa, e você pode omitir o campo no payload.

| Preset | Afeta | Observação |
|---|---|---|
| Canal de venda (`channel_code`) | Cotação | Usado quando você não informa o canal |
| Imposto (`tax`) | Cotação | `sender` ou `receiver` |
| Seguro (`insurance`) | Cotação | - |
| Objeto (`object`) e tipo (`type`) | Cotação | - |
| Moedas (`currency_quote`, `currency_payment`) | Cotação | - |
| Sistema de medida (`measurement`) | Cotação | `metric` ou `imperial` |
| Entrega residencial / não empilhável | Cotação | - |
| Endereço do remetente | Cotação e remessa | Aplicado como `address_code` |
| Frete declarado (`declared_freight`) | Cotação | Valor + moeda |
| Faturante (`biller`) | Remessa | Endereço exibido na invoice |
| Agrupamento de caixas | Cotação | `default`, `all_in_one` ou `smartbox` |

**Precedência:** o que você envia no payload sempre vence o preset. Exceções:
- Se você informar `address_sender.country_code` na cotação, o preset de endereço do remetente é descartado.
- Se você informar `address_sender.street` na remessa, o preset de endereço do remetente é descartado.
- Se você informar `declared_freight`, o preset de frete declarado é descartado.

### Agrupamento de caixas
O agrupamento define como os itens são distribuídos nas caixas quando você usa `type: items`.
Ele **não** é um campo do payload: é uma configuração da sua conta.

| Modo | Comportamento |
|---|---|
| `default` | Agrupamento padrão da plataforma |
| `all_in_one` | Consolida tudo em uma única caixa |
| `smartbox` | Encaixa os itens nas caixas cadastradas considerando volume e dimensões (com 10% de folga) e respeitando `is_groupable` do item |

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
