# Rascunho

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Permite registrar um envio **incompleto**: você manda o que já tem, e a plataforma valida depois.
- Na entrada, **todos os campos são opcionais** (só o formato de cada campo é conferido). A validação
  rígida acontece depois, no **despacho**, que roda automaticamente em ciclos de poucos minutos.
- Quando o rascunho passa no despacho, a plataforma gera a **cotação + remessa** sozinha e o rascunho
  **deixa de existir** (é removido). Enquanto faltarem dados, ele fica `incomplete` e pode ser corrigido.
- É o fluxo indicado quando a integração recebe os dados do pedido em etapas (ex.: o pedido chega antes
  do endereço final ou da nota fiscal).
- As **predefinições da conta** (presets) são aplicadas ao rascunho, inclusive o responsável pelo
  imposto, que **substitui** o valor enviado quando estiver configurado.

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

O header `lang` aceita `pt`, `en` e `es` (padrão `pt`). Cada operação exige uma permissão própria
no perfil (criar, editar, listar e remover rascunhos). Sem ela, a resposta é `403 Has no permission!`.

## Fluxo de uso
1. `POST /v1/draft` com os dados que você já tem. A resposta apenas confirma o recebimento
   (**não devolve o código do rascunho**).
2. Aguarde alguns minutos: o rascunho é gravado e depois passa pelo despacho automático.
3. `GET /v1/draft` para localizar o rascunho (pelo seu `channel_identifier_number`), obter o `codigo`
   e ver o `status`. Se estiver `incomplete`, leia o `status_descricao`.
4. `PUT /v1/draft/{code}` enviando **o rascunho completo e corrigido** (o `PUT` substitui todo o conteúdo).
5. Repita 2 a 4 até o rascunho sumir da listagem: isso significa que a cotação e a remessa foram criadas.
6. Acompanhe a remessa pelo [rastreio](api-rastreio.md) e baixe os [documentos](api-documentos.md),
   usando o `channel_identifier_number` (e o `channel_code`, se houver).
7. `DELETE /v1/draft` para descartar rascunhos que não serão usados (ou que ficaram com status `error`).

## Conceitos essenciais

### Rascunho vs. outros fluxos

| Tema | Cotação + Remessa | Manifesto | Rascunho |
|---|---|---|---|
| Payload | Completo em 2 etapas | Completo em 1 chamada | **Parcial, em quantas etapas precisar** |
| Validação | Síncrona, por etapa | Síncrona, total | **Formato na entrada; regras completas no despacho (assíncrono)** |
| Resposta | Cotação com preços | Documentos prontos | Confirmação de recebimento (sem código) |
| Pagamento | `billed`, `gateway`, `afterward` | `billed` | **`billed` ou `afterward`** |
| Escolha do contrato | Você escolhe | Você escolhe | **Automática** (`carrier_code` preferido ou preferência da conta) |
| Correção posterior | Nova cotação | Não | **Sim, via `PUT` (só em `incomplete`)** |

### Ciclo de vida do rascunho

```
POST /v1/draft                      PUT /v1/draft/{code}
     │                                       │
     ▼                                       ▼
  received ──► validating ──► incomplete ──► edited ──► validating ──► ...
                    │
                    ▼
                 complete ──► processing ──► cotação + remessa criadas (rascunho removido)
                                  │
                                  ├──► incomplete  (regra de negócio recusou: motivo em status_descricao)
                                  └──► error       (falha inesperada: motivo em status_descricao)
```

| Status | Significado | Pode editar (`PUT`)? |
|---|---|---|
| `received` | Rascunho recebido, aguardando o próximo ciclo de despacho | Não |
| `edited` | Rascunho atualizado via `PUT`, aguardando o próximo ciclo de despacho | Não |
| `validating` | Em validação | Não |
| `incomplete` | **Faltam dados ou alguma regra recusou o envio.** O motivo está em `status_descricao` | **Sim** |
| `complete` | Passou na validação, pronto para gerar cotação e remessa | Não |
| `processing` | Gerando cotação e remessa | Não |
| `error` | Falha inesperada no processamento | Não (remova e crie de novo) |

| Detalhe | Comportamento |
|---|---|
| Frequência do despacho | Os rascunhos `received` e `edited` são validados automaticamente a cada ~3 minutos |
| Sucesso | A cotação e a remessa são criadas e **o rascunho é removido** (não aparece mais no `GET`) |
| Rascunho preso em `processing` | Após ~15 minutos sem conclusão, volta para `incomplete` com `status_descricao` vazio |
| Tamanho do `status_descricao` | Limitado a **254 caracteres**: a lista de pendências pode vir cortada |
| Formato do `status_descricao` | Na validação de campos, traz a primeira mensagem de cada campo, concatenadas sem separador |

> **Somente rascunhos com status `incomplete` podem ser editados.** Em qualquer outro status a resposta é
> `400 Apenas rascunhos com o status incomplete podem ser editados`.

### O `PUT` substitui o rascunho inteiro

O `PUT /v1/draft/{code}` **não** mescla com o conteúdo anterior: o conteúdo salvo é **trocado** pelo
payload enviado (já mesclado com os presets da conta). Campos que você não reenviar **são perdidos**.

- Sempre envie o rascunho completo, com as correções.
- Atenção ao montar o `PUT` a partir do `GET`: a listagem **não devolve** `measurement`,
  `residential_delivery`, `non_stackable`, `cost_center_code`, `declared_freight`, `biller`,
  `manufacturer` e `address_sender.address_code`. Reenvie-os a partir dos seus próprios dados.

### Predefinições da conta (presets) no rascunho

Os presets configurados no painel são mesclados ao payload no `POST` **e** no `PUT`
(veja [Predefinições da conta](api-primeiros-passos.md#predefinições-da-conta-presets)).

| Campo | Preset usado quando | Observação |
|---|---|---|
| `channel_code` | O campo **não** é enviado | Enviar o campo (mesmo `null`) impede o preset |
| `cost_center_code` | O campo não é enviado **ou** vem vazio/`null` | Único campo em que valor vazio também recebe o preset |
| `carrier_code` | O campo não é enviado | - |
| `tax` | **Sempre que houver preset de imposto** | O preset **substitui** o valor enviado (ver abaixo) |
| `insurance` | O campo não é enviado | - |
| `measurement` | O campo não é enviado ou vem `null` | Sem preset, assume `metric` |
| `currency_quote`, `currency_payment` | O campo não é enviado | - |
| `residential_delivery`, `non_stackable` | O campo não é enviado | - |
| `type` (finalidade do envio) | O campo não é enviado | - |
| `address_sender` | O objeto `address_sender` **não** é enviado | O preset entra como `address_code`. Enviar `address_sender` com qualquer campo descarta o preset |
| `biller` | O objeto não é enviado **ou** vem vazio, e a conta está configurada para exibir o faturante | Se você enviar `biller` preenchido, ele sempre vale |
| `declared_freight` | O objeto não é enviado | Frete declarado repassado à cotação |

> A mescla é feita **no primeiro nível**: um objeto enviado (`address_sender`, `biller`,
> `declared_freight`) substitui o objeto do preset por inteiro, sem combinar campo a campo.

### Responsável pelo imposto (`tax`)

Quando a conta tem preset de imposto, **o valor enviado em `tax` é ignorado** e a plataforma resolve
o responsável assim:

| Configuração do preset de imposto | Valor gravado no rascunho |
|---|---|
| Sem preset de imposto | O `tax` enviado. Se não enviar, o despacho acusa `tax` obrigatório |
| Padrão `sender` | `sender`, mesmo que você envie `receiver` |
| Padrão `receiver` | `receiver`, mesmo que você envie `sender` |
| Padrão definido e país de destino na lista de exceções | O **oposto** do padrão (`sender` ↔ `receiver`) |
| Modo travado | Sempre `sender` |

- O país de destino considerado é o `address_receiver.country_code` **do mesmo payload**. Se ele não
  for enviado, a exceção por país não é avaliada e vale o padrão.
- Como o `PUT` reaplica os presets, o imposto é recalculado a cada atualização.

## Endpoint: criar rascunho

- **Método e rota:** `POST /v1/draft`
- **Quando usar:** para registrar um envio, mesmo com dados faltando.
- **Processamento:** assíncrono. A resposta `200` só confirma o recebimento; o rascunho aparece no
  `GET` alguns instantes depois.

### Parâmetros — raiz do payload

"Entrada" = regra conferida na chamada (erro `422`). "Despacho" = regra conferida depois, que deixa
o rascunho `incomplete`.

| Campo | Tipo | Entrada | Despacho | Mín. | Máx. | Padrão | Valores / descrição |
|---|---|---|---|---|---|---|---|
| `channel_identifier_number` | string | opcional | opcional | - | 100 | - | Seu identificador do pedido. **Recomendado**: é por ele que você localiza o rascunho e a remessa |
| `channel_code` | integer | opcional | opcional | - | - | preset da conta | Canal de venda cadastrado. No despacho precisa pertencer à sua conta |
| `cost_center_code` | integer | opcional | opcional | - | - | preset da conta | Centro de custo cadastrado da sua conta (ou de contas autorizadas) |
| `carrier_code` | integer | opcional | opcional | - | - | preset da conta | Contrato preferido. Usado se estiver disponível no resultado da cotação |
| `object` | string | opcional | opcional | - | - | `package` | `doc`, `package`, `pallet`, `not_doc` |
| `type` | string | opcional | opcional | - | 30 | `Commercial` (ou preset) | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` |
| `tax` | string | opcional | **obrigatório** | - | - | preset da conta | `sender`, `receiver`. Ver [Responsável pelo imposto](#responsável-pelo-imposto-tax) |
| `insurance` | boolean | opcional | **obrigatório** | - | - | preset da conta | Contratar seguro |
| `measurement` | string | opcional | opcional | - | - | preset ou `metric` | `metric`, `imperial` |
| `ior` | boolean | opcional | opcional | - | - | `false` | Importer of Record |
| `residential_delivery` | boolean | opcional | opcional | - | - | `false` (ou preset) | Entrega residencial |
| `non_stackable` | boolean | opcional | opcional | - | - | `false` (ou preset) | Volumes não empilháveis |
| `currency_quote` | string | opcional | **necessário** (ver nota) | 3 | 3 | preset da conta | Moeda da cotação. Ver [moedas](api-primeiros-passos.md#moedas-aceitas) |
| `currency_payment` | string | opcional | **necessário** (ver nota) | 3 | 3 | preset da conta | Moeda do pagamento. Ver [moedas](api-primeiros-passos.md#moedas-aceitas) |
| `nfkey` | string | opcional | opcional | 44 | 44 | - | Chave da NF-e |
| `description` | string | opcional | opcional | - | 155 | - | Descrição livre |
| `address_sender` | object | opcional | **obrigatório** | - | - | preset da conta | Remetente (ver tabela) |
| `address_receiver` | object | opcional | **obrigatório** | - | - | - | Destinatário (ver tabela) |
| `boxes` | array | opcional | ver regras | - | - | - | Volumes (ver tabela) |
| `items` | array | opcional | ver regras | - | - | - | Itens (ver tabela) |
| `payment` | object | opcional | **obrigatório** | - | - | - | Pagamento (ver tabela) |
| `declared_freight` | object | opcional | opcional | - | - | preset da conta | Frete declarado: `value` (number) e `currency` (3 letras). Não é conferido na entrada; é repassado à cotação |
| `biller` | object | opcional | opcional | - | - | preset da conta | Faturante exibido na invoice (mesma estrutura do remetente) |
| `manufacturer` | object | opcional | opcional | - | - | - | Fabricante exibido na invoice (ver tabela) |

> **Nota sobre moedas:** a validação do despacho não acusa a falta de `currency_quote` e
> `currency_payment`, mas sem elas (no payload ou no preset) a geração da cotação falha e o rascunho
> vai para `error`, que **não pode ser editado**. Sempre informe as duas moedas ou configure o preset.

### Parâmetros — `address_sender` e `address_receiver`

Duas formas de informar o **remetente** (não misture):
- **Forma A — endereço cadastrado:** envie só `address_sender.address_code`. No despacho, o endereço
  cadastrado é copiado para o rascunho e validado com as mesmas regras abaixo.
- **Forma B — campos manuais:** envie os campos abaixo e **não** envie `address_code`.

O destinatário aceita **apenas a Forma B** (`address_receiver.address_code` não é usado).

| Campo | Tipo | Mín. | Máx. (entrada) | Máx. (despacho) | Obrigatório no despacho | Descrição |
|---|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | - | Código de endereço cadastrado (apenas `address_sender`) |
| `country_code` | string | 2 | 2 | 2 | **Sim** | País em ISO 3166-1 alpha-2 |
| `postal_code` | string | - | 12 | 12 | Sim, se `state` não for informado | CEP/ZIP |
| `state` | string | - | - | - | Sim, se `postal_code` e `state_code` não forem informados | Nome do estado |
| `state_code` | string | - | - | - | Ver nota | Sigla do estado, conferida contra a base de estados. Para país sem estados, use o próprio código do país |
| `city` | string | - | 100 | 100 | **Sim** | Cidade |
| `street` | string | - | 70 | 70 | **Sim** | Logradouro |
| `number` | string | - | 100 | **6** | - | Número do imóvel |
| `district` | string | - | 30 | 30 | - | Bairro |
| `complement` | string | - | 20 | 20 | - | Complemento |
| `description` | string | - | 155 | 155 | - | Rótulo interno do endereço |
| `name` | string | 5 | 35 | 35 | **Sim** | Nome do contato/razão social |
| `email` | string | - | 60 | 60 | **Sim** | E-mail válido |
| `phone` | string | - | 25 | 25 | **Sim** | Telefone |
| `phone_extension` | string | - | 5 | 5 | - | Ramal |
| `type` | string | - | - | - | **Sim** (sem padrão) | `pf` ou `pj` |
| `federal_tax_id` | string | - | 20 | 20 | Sim quando o país é `BR` | Documento federal. Para `BR` sem `foreign: true`, precisa ser um CPF ou CNPJ válido |
| `state_tax_id` | string | - | 20 | 20 | Sim quando o país é `BR` | Inscrição estadual |
| `foreign` | boolean | - | - | - | - | Indica pessoa estrangeira (dispensa a checagem de CPF/CNPJ) |

> **Localização do endereço:** informe `postal_code` **ou** `state`. `state_code` sozinho não basta,
> porque, sem `state`, o `postal_code` passa a ser exigido.
>
> **Atenção ao `number`:** a entrada aceita até 100 caracteres, mas o despacho só aceita até **6**.
> Valores maiores deixam o rascunho `incomplete`.

### Parâmetros — `boxes[]`

Quando você **envia `boxes`**, cada caixa precisa ser informada por **uma** das formas abaixo (não misture
na mesma caixa):

| Forma | Campos | O que acontece |
|---|---|---|
| A — kit | `kit` | A caixa e os itens do kit são preenchidos a partir do kit cadastrado |
| B — caixa cadastrada por código | `box_code` | Usa a [caixa cadastrada](api-caixa-v1.md) |
| C — caixa cadastrada por identificador | `box_identifier` | Usa a caixa cadastrada |
| D — caixa manual | `name`, `height`, `width`, `depth` | **Os quatro campos são obrigatórios no despacho** |

| Campo | Tipo | Mín. | Máx. | Descrição |
|---|---|---|---|---|
| `kit` | string | - | 50 | SKU do kit cadastrado |
| `box_code` | integer | - | - | Código da caixa cadastrada |
| `box_identifier` | string | - | - | Identificador da caixa cadastrada |
| `name` | string | 4 | 155 | Nome da caixa |
| `height` | number | 1 | - | Altura |
| `width` | number | 1 | - | Largura |
| `depth` | number | 1 | - | Profundidade |
| `volume` | number | 0.001 | - | Volume. Aceito, mas **não dispensa** as dimensões no despacho |
| `weight` | number | 0.1 | - | Peso do volume, até 3 casas decimais. **Obrigatório (já na entrada) quando `object` é `doc`**. Ignorado em caixa com `kit` |
| `price` | number | 0 | - | Valor declarado. **Obrigatório (já na entrada) quando `object` é `doc`**. Ignorado em caixa com `kit` |

> Com `boxes` e `object` diferente de `doc`, **toda caixa que não seja kit precisa ter pelo menos um item**
> apontando para ela em `items[].box_id`. Com `object: doc`, os `items` são descartados.

### Parâmetros — `items[]`

Duas formas de informar cada item:
- **Forma A — item cadastrado:** `sku` (+ `quantity`, `unit_value` e, com `boxes`, `box_id`). Os dados
  faltantes vêm do [cadastro de itens](api-itens-v1.md). O SKU precisa existir.
- **Forma B — item manual:** todos os campos marcados como obrigatórios sem `sku`.

| Campo | Tipo | Mín. | Máx. | Obrigatório no despacho | Descrição |
|---|---|---|---|---|---|
| `sku` | string | - | 50 | - | SKU do item cadastrado |
| `ean` | string | - | 50 | - | Código EAN |
| `nsmp` | string | - | 50 | - | *Non Standard Manufacturer Product Id*: identificador de produto não padronizado do fabricante. Impresso na invoice e usado nas informações regulatórias exigidas por alguns transportadores para destinos na União Europeia. Com `sku`, se não for enviado, vem do cadastro do item |
| `box_id` | integer | - | - | Sim, quando há `boxes` | **Posição** da caixa no array `boxes` (1 = primeira). Sem `boxes`, é ignorado |
| `name` | string | 6 | 35 | Sim, sem `sku` | Nome do item |
| `description` | string | 4 | 155 | - | Descrição do item (o mínimo de 4 é conferido no despacho) |
| `country_code` | string | 2 | 2 | Sim, sem `sku` | País de origem do item |
| `hscode` | string | 6 | 10 | Sim, sem `sku` | NCM/HS Code |
| `weight` | number | - | 99 | Sim, sem `sku` | Peso unitário (o máximo de 99 é conferido no despacho) |
| `quantity` | integer | 1 | 1000 | **Sim** | Quantidade |
| `unit_value` | number | - | - | **Sim** | Valor unitário |
| `selling_unit_value` | number | - | - | Exigido pela cotação quando `ior` é `true` | Valor de venda unitário |
| `height`, `width`, `depth` | number | 1 | - | Sim, quando **não** há `boxes` (salvo SKU cadastrado com dimensões) | Dimensões do item, usadas para calcular as caixas |
| `is_groupable` | boolean | - | - | - | Padrão `true`. Sem `boxes`: itens agrupáveis vão juntos em uma caixa calculada; `false` gera uma caixa por unidade |
| `is_foldable` | boolean | - | - | - | Aceito, mas sem efeito no rascunho |

> **Quando `items` é obrigatório:** sem `boxes`, `items` é sempre exigido (inclusive para `doc`). Com
> `boxes`, é exigido para `package`, `pallet` e `not_doc`, exceto quando as caixas usam `kit`.
>
> **Sem `boxes`:** a plataforma monta as caixas a partir das dimensões dos itens. Para `doc`, envie
> sempre `boxes` com `weight` e `price`.

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório no despacho | Valores | Descrição |
|---|---|---|---|---|
| `payment.type` | string | **Sim** | `billed`, `afterward` | O rascunho **não aceita** `gateway` (erro `422` na entrada) |

### Parâmetros — `biller`

Mesma estrutura e limites de `address_sender`, incluindo `address_code` (Forma A) ou campos manuais
(Forma B). No despacho, todos os campos do `biller` são opcionais; o `number` aceita até 6 caracteres.
Se você não enviar `biller` (ou enviar vazio), o faturante do preset da conta pode ser aplicado.

### Parâmetros — `manufacturer`

Envie `address_code` **ou** os campos manuais. O fabricante **não é conferido na entrada**: os limites
abaixo são aplicados no despacho.

| Campo | Tipo | Mín. | Máx. | Obrigatório no despacho | Descrição |
|---|---|---|---|---|---|
| `address_code` | integer | - | - | - | Código de endereço cadastrado. Quando usado, os demais campos são opcionais |
| `name` | string | 5 | 30 | Sim, sem `address_code` | Razão social/nome do fabricante |
| `country_code` | string | 2 | 2 | Sim, sem `address_code` | País |
| `state_code` | string | 1 | 10 | Sim, sem `address_code` | Sigla do estado |
| `state` | string | - | 155 | - | Nome do estado |
| `postal_code` | string | - | 12 | Sim, sem `address_code` | CEP/ZIP |
| `city` | string | - | 100 | Sim, sem `address_code` | Cidade |
| `street` | string | - | 70 | Sim, sem `address_code` | Logradouro |
| `number` | string | - | 6 | - | Número |
| `district` | string | - | 30 | - | Bairro |
| `complement` | string | - | 20 | - | Complemento |
| `email` | string | - | 60 | Sim, sem `address_code` | E-mail |
| `phone` | string | - | 25 | Sim, sem `address_code` | Telefone |
| `phone_extension` | string | - | 6 | - | Ramal |
| `type` | string | - | - | Sim, sem `address_code` | `pf` ou `pj` |
| `federal_tax_id` | string | - | 20 | - | Documento federal |
| `state_tax_id` | string | - | 20 | - | Inscrição estadual |

### Payload mínimo

O menor payload aceito é um objeto vazio: ele cria um rascunho que ficará `incomplete` até ser completado.

```json
{}
```

Na prática, envie ao menos o seu identificador, para conseguir localizar o rascunho depois:

```json
{
  "channel_identifier_number": "PED-2026-0001"
}
```

Payload mínimo para o rascunho **passar no despacho** sem depender de presets (remetente e destinatário
fora do Brasil, caixa manual):

```json
{
  "channel_identifier_number": "PED-2026-0001",
  "tax": "sender",
  "insurance": false,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "US",
    "postal_code": "10001",
    "city": "New York",
    "street": "Fifth Avenue",
    "name": "Loja Exemplo",
    "email": "contato@example.com",
    "phone": "12125550100",
    "type": "pj"
  },
  "address_receiver": {
    "country_code": "CA",
    "postal_code": "M5V2T6",
    "city": "Toronto",
    "street": "King Street",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "14165550100",
    "type": "pf"
  },
  "boxes": [
    { "name": "Caixa P", "height": 20, "width": 20, "depth": 20 }
  ],
  "items": [
    {
      "box_id": 1,
      "name": "Camiseta algodao",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9
    }
  ],
  "payment": { "type": "billed" }
}
```

### Payload máximo

Todos os campos aceitos. Os volumes mostram as formas manual (caixa 1), caixa cadastrada (caixa 2)
e kit (caixa 3); em uma integração real use a forma adequada a cada volume. O remetente usa a Forma B
(campos manuais); para a Forma A, envie apenas `"address_sender": {"address_code": 1}`.

```json
{
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 1,
  "carrier_code": 1,
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": true,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": true,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "nfkey": "12345678901234567890123456789012345678901234",
  "description": "Rascunho do pedido do marketplace",
  "declared_freight": {
    "value": 50,
    "currency": "USD"
  },
  "address_sender": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Centro de distribuicao SP",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Galpao 2",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "phone_extension": "12",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "state": "Florida",
    "state_code": "FL",
    "postal_code": "33101",
    "city": "Miami",
    "description": "Escritorio Miami",
    "street": "Main Avenue",
    "number": "33",
    "district": "Downtown",
    "complement": "Apt 3",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "11992494597",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": "987654321",
    "foreign": true
  },
  "boxes": [
    {
      "name": "Caixa master",
      "height": 20,
      "width": 20,
      "depth": 20,
      "volume": 0.008,
      "weight": 2.5,
      "price": 150
    },
    { "box_identifier": "CX0236" },
    { "kit": "KIT-001" }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7894900010015",
      "nsmp": "NSMP-TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao manga curta branca",
      "country_code": "BR",
      "hscode": "610910",
      "height": 2,
      "width": 20,
      "depth": 25,
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9,
      "is_groupable": true,
      "is_foldable": true
    },
    {
      "box_id": 2,
      "name": "Bermuda sarja M",
      "description": "Bermuda sarja bege",
      "country_code": "BR",
      "hscode": "620342",
      "weight": 0.4,
      "quantity": 1,
      "unit_value": 59.9,
      "selling_unit_value": 89.9
    }
  ],
  "payment": { "type": "billed" },
  "biller": {
    "country_code": "BR",
    "state": "Sao Paulo",
    "state_code": "SP",
    "postal_code": "09941070",
    "city": "Diadema",
    "description": "Matriz",
    "street": "Rua Principal",
    "number": "100",
    "district": "Centro",
    "complement": "Sala 1",
    "name": "Holding Exemplo LTDA",
    "email": "financeiro@example.com",
    "phone": "11999999999",
    "phone_extension": "10",
    "type": "pj",
    "federal_tax_id": "98765432000188",
    "state_tax_id": "987654321098",
    "foreign": false
  },
  "manufacturer": {
    "name": "Fabrica Exemplo LTDA",
    "country_code": "BR",
    "state_code": "SP",
    "state": "Sao Paulo",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": "200",
    "district": "Centro",
    "complement": "Bloco B",
    "email": "fabrica@example.com",
    "phone": "11999999999",
    "phone_extension": "20",
    "type": "pj",
    "federal_tax_id": "11222333000144",
    "state_tax_id": "111222333444"
  }
}
```

### Exemplo de request

```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Caixa P", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodao", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Recebemos o seu rascunho com sucesso. Em breve ele será processado!"
}
```

`422 Unprocessable Entity` (ex.: `payment.type` com `gateway`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "payment.type": ["O campo payment.type não contém um valor válido."]
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success`, `warning` ou `error` |
| `message` | string | Mensagem de confirmação ou de erro |
| `data` | object | Somente em erro `422`: mapa campo → lista de mensagens |

> A resposta de sucesso **não traz o código do rascunho**. Use `GET /v1/draft` para obtê-lo.

## Endpoint: listar rascunhos

- **Método e rota:** `GET /v1/draft`
- **Quando usar:** para descobrir o `codigo` de um rascunho e acompanhar o `status`/`status_descricao`.
- **Escopo:** somente rascunhos da sua conta. **Itens por página:** 20.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `page` | query | integer | não | Página da listagem (padrão 1) |
| `drafts` | body | object | não | Envelope do filtro |
| `drafts.drafts_ids` | body | array | não | Códigos de rascunho (inteiros ou strings). **Vazio ou ausente retorna todos** |

> Este `GET` aceita um corpo JSON para o filtro. Se o seu cliente HTTP não suportar body em `GET`,
> liste todos (paginando) e filtre do seu lado.

### Payload mínimo

Sem corpo e sem query: lista a primeira página de todos os rascunhos.

```
GET /v1/draft
```

### Payload máximo

```
GET /v1/draft?page=1
```

```json
{
  "drafts": {
    "drafts_ids": [30, 31]
  }
}
```

### Exemplo de request

```bash
curl --request GET \
  --url 'https://api.shipsmart.global/v1/draft?page=1' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30]}}'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "codigo": 30,
        "codigo_usuario": 1,
        "data": {
          "channel_identifier_number": "PED-2026-0001",
          "carrier_code": "",
          "nfkey": "",
          "object": "package",
          "type": "Commercial",
          "channel_code": "",
          "channel": "",
          "tax": "sender",
          "insurance": true,
          "ior": false,
          "currency_quote": "USD",
          "currency_payment": "BRL",
          "description": "",
          "address_sender": {
            "country_code": "",
            "state": "",
            "state_code": "",
            "postal_code": "",
            "city": "",
            "description": "",
            "street": "",
            "number": "",
            "district": "",
            "complement": "",
            "name": "",
            "email": "",
            "phone": "",
            "type": "",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "address_receiver": {
            "country_code": "US",
            "state": "",
            "state_code": "",
            "postal_code": "33101",
            "city": "Miami",
            "description": "",
            "street": "Main Avenue",
            "number": "",
            "district": "",
            "complement": "",
            "name": "John Doe",
            "email": "johndoe@example.com",
            "phone": "11992494597",
            "type": "pf",
            "phone_extension": "",
            "federal_tax_id": "",
            "state_tax_id": "",
            "foreign": ""
          },
          "items": [
            {
              "box_id": 1,
              "name": "Camiseta algodao",
              "country_code": "BR",
              "hscode": "610910",
              "weight": 0.25,
              "quantity": 3,
              "unit_value": 29.9
            }
          ],
          "boxes": [
            { "name": "Caixa P", "height": 20, "width": 20, "depth": 20 }
          ],
          "payment": { "type": "billed" }
        },
        "status": "incomplete",
        "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
        "created_at": "2026-02-20 19:04:00",
        "updated_at": "2026-02-20 19:08:14"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/draft?page=1",
    "links": [
      { "url": null, "label": "« Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/draft?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo »", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/draft",
    "per_page": 20,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.data[].codigo` | integer | **Código do rascunho.** Use em `PUT /v1/draft/{code}` e em `drafts.drafts_ids` |
| `data.data[].codigo_usuario` | integer | Código da conta dona do rascunho |
| `data.data[].data` | object | Conteúdo do rascunho (já mesclado com os presets), em formato fixo: campos ausentes vêm como `""` |
| `data.data[].data.channel` | string | Nome do canal de venda (quando há `channel_code`) |
| `data.data[].status` | string | Status atual (ver [ciclo de vida](#ciclo-de-vida-do-rascunho)) |
| `data.data[].status_descricao` | string/null | **Quando `incomplete` ou `error`, diz o que impediu o avanço** (até 254 caracteres) |
| `data.data[].created_at`, `updated_at` | string | Datas no formato `AAAA-MM-DD HH:MM:SS` |
| `data.current_page`, `last_page`, `per_page`, `total`, `from`, `to` | integer | Dados da paginação |
| `data.first_page_url`, `last_page_url`, `next_page_url`, `prev_page_url`, `path`, `links` | string/array | Links de navegação entre páginas |

> O objeto `data` da listagem **não inclui** `measurement`, `residential_delivery`, `non_stackable`,
> `cost_center_code`, `declared_freight`, `biller`, `manufacturer` nem `address_sender.address_code`.
> Quando o remetente foi informado por `address_code`, depois do primeiro despacho ele aparece expandido
> com os dados do endereço cadastrado.
>
> Se não houver nenhum rascunho para os filtros (ou a página pedida estiver além da última), a resposta é
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`, e não uma lista vazia.

## Endpoint: atualizar rascunho

- **Método e rota:** `PUT /v1/draft/{code}`
- **Quando usar:** para corrigir um rascunho `incomplete`.
- **Restrição:** apenas rascunhos da sua conta com status `incomplete`.
- **Corpo:** mesmos campos e regras do `POST`. **O conteúdo anterior é totalmente substituído**
  (ver [O PUT substitui o rascunho inteiro](#o-put-substitui-o-rascunho-inteiro)).
- **Efeito:** o status passa para `edited` e o rascunho volta para o próximo ciclo de despacho.
  O `status_descricao` anterior permanece até a nova validação.

### Parâmetros

| Parâmetro | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `code` | path | integer | **sim** | Código do rascunho (`codigo` na listagem) |
| corpo | body | object | não | Mesma estrutura do [`POST`](#endpoint-criar-rascunho) |

### Payload mínimo

`PUT /v1/draft/30` com corpo `{}`. É aceito, mas **apaga** o conteúdo do rascunho (ficam apenas os presets).
Use apenas se quiser recomeçar o rascunho.

```json
{}
```

### Payload máximo

O mesmo [payload máximo do `POST`](#payload-máximo). Na prática, reenvie o rascunho completo com as correções.

### Exemplo de request

Rascunho que estava `incomplete` por falta do remetente: o corpo traz o conteúdo original **mais** o remetente.

```bash
curl --request PUT \
  --url https://api.shipsmart.global/v1/draft/30 \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "channel_identifier_number": "PED-2026-0001",
    "object": "package",
    "type": "Commercial",
    "tax": "sender",
    "insurance": true,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "number": "100", "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11989677465", "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": "123456789012"},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "name": "John Doe", "email": "johndoe@example.com", "phone": "11992494597", "type": "pf"},
    "boxes": [{"name": "Caixa P", "height": 20, "width": 20, "depth": 20}],
    "items": [{"box_id": 1, "name": "Camiseta algodao", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9}],
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) atualizado(s) com sucesso.",
  "data": {
    "codigo": 30,
    "codigo_usuario": 1,
    "data": {
      "channel_identifier_number": "PED-2026-0001",
      "object": "package",
      "type": "Commercial",
      "tax": "sender",
      "insurance": true,
      "measurement": "metric",
      "currency_quote": "USD",
      "currency_payment": "BRL",
      "address_sender": {
        "country_code": "BR",
        "postal_code": "09941070",
        "city": "Diadema",
        "street": "Rua Principal",
        "number": "100",
        "name": "Loja Exemplo LTDA",
        "email": "contato@example.com",
        "phone": "11989677465",
        "type": "pj",
        "federal_tax_id": "12345678000199",
        "state_tax_id": "123456789012"
      },
      "address_receiver": {
        "country_code": "US",
        "postal_code": "33101",
        "city": "Miami",
        "street": "Main Avenue",
        "name": "John Doe",
        "email": "johndoe@example.com",
        "phone": "11992494597",
        "type": "pf"
      },
      "boxes": [
        { "name": "Caixa P", "height": 20, "width": 20, "depth": 20 }
      ],
      "items": [
        {
          "box_id": 1,
          "name": "Camiseta algodao",
          "country_code": "BR",
          "hscode": "610910",
          "weight": 0.25,
          "quantity": 3,
          "unit_value": 29.9
        }
      ],
      "payment": { "type": "billed" }
    },
    "status": "edited",
    "status_descricao": "O campo address sender é obrigatório.O campo address sender.country code é obrigatório.O campo address sender.state é obrigatório quando nenhum dos address sender.state code / address sender.postal code está presente.O campo address sender.state code é o",
    "created_at": "2026-02-20 19:04:00",
    "updated_at": "2026-02-20 19:20:31"
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `data.codigo` | integer | Código do rascunho |
| `data.codigo_usuario` | integer | Código da conta dona do rascunho |
| `data.data` | object | Conteúdo gravado: o payload enviado já mesclado com os presets. Pode conter campos adicionais vindos das predefinições da conta; ignore os que você não conhece |
| `data.status` | string | Sempre `edited` após a atualização |
| `data.status_descricao` | string/null | Motivo da validação anterior (será atualizado no próximo despacho) |
| `data.created_at`, `data.updated_at` | string | Datas no formato `AAAA-MM-DD HH:MM:SS` |

## Endpoint: remover rascunhos

- **Método e rota:** `DELETE /v1/draft`
- **Quando usar:** para descartar rascunhos que não serão usados, inclusive os que ficaram em `error`.
- **Escopo:** só são removidos os códigos que pertencem à sua conta; os demais são ignorados.
  A remoção é definitiva.

### Parâmetros

| Campo | Local | Tipo | Obrigatório | Descrição |
|---|---|---|---|---|
| `drafts` | body | object | **sim** | Envelope |
| `drafts.drafts_ids` | body | array | **sim** (não vazio) | Códigos de rascunho a remover (inteiros ou strings) |

### Payload mínimo

```json
{ "drafts": { "drafts_ids": [30] } }
```

### Payload máximo

```json
{ "drafts": { "drafts_ids": [30, 31, 32] } }
```

### Exemplo de request

```bash
curl --request DELETE \
  --url https://api.shipsmart.global/v1/draft \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{"drafts": {"drafts_ids": [30, 31]}}'
```

### Exemplo de response

```json
{
  "status": "success",
  "message": "Registro(s) deletado(s) com sucesso."
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | `success` quando ao menos um rascunho da lista foi removido |
| `message` | string | Mensagem de confirmação |

> Basta **um** código da lista pertencer à sua conta para a resposta ser `success`; os outros são
> ignorados sem aviso. Se nenhum código corresponder, a resposta é
> `400 O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados`.

## Regras de negócio

### Validação no despacho
No despacho, a plataforma aplica um conjunto **rígido** de regras. O que falta aparece em
`status_descricao` e o rascunho fica `incomplete`.

| Grupo | Exigências |
|---|---|
| Geral | `tax` e `insurance` presentes (no payload ou via preset); `channel_identifier_number` texto de até 100 caracteres |
| Remetente e destinatário | `country_code`, `city`, `street`, `name`, `email`, `phone`, `type` + (`postal_code` ou `state`); `number` com até 6 caracteres |
| Endereços no Brasil (`BR`) | `federal_tax_id` e `state_tax_id`; `federal_tax_id` deve ser CPF ou CNPJ válido, salvo `foreign: true` |
| Sem `boxes` | `items` obrigatório (qualquer `object`), com dimensões para calcular as caixas |
| Com `boxes` | Cada caixa: `kit`, `box_code`, `box_identifier` ou (`name` + `height` + `width` + `depth`). Para `object` ≠ `doc`, cada caixa (exceto kit) precisa de ao menos um item com o `box_id` dela |
| `object` = `doc` | `boxes[].weight` e `boxes[].price` (conferidos já na entrada) |
| Itens | `quantity` (1 a 1000) e `unit_value`; sem `sku`: `name`, `country_code`, `hscode`, `weight` (até 99). Com `sku`, o SKU precisa estar cadastrado |
| Pagamento | `payment.type` (`billed` ou `afterward`) |
| Fabricante | Com `manufacturer` sem `address_code`: `name`, `country_code`, `state_code`, `postal_code`, `city`, `street`, `email`, `phone` e `type` |
| Canal | `channel_code`, quando informado (ou vindo do preset), precisa pertencer à sua conta |

### Geração da cotação e da remessa
Depois de validado, o rascunho vira uma cotação e uma remessa automaticamente:

| Etapa | Comportamento |
|---|---|
| Kits | Cada `boxes[].kit` é substituído pela caixa e pelos itens do kit cadastrado |
| Caixas | Com `boxes`, usa as caixas enviadas. Sem `boxes`, calcula as caixas pelas dimensões dos itens (agrupáveis juntos; não agrupáveis, uma caixa por unidade) |
| Cotação | Tipo simples para `doc` e avançado para os demais. Usa `declared_freight` do payload ou do preset |
| Escolha do contrato | Se `carrier_code` estiver entre os resultados, ele é usado. Senão, a plataforma escolhe pela preferência da conta: o **mais barato** ou o **mais rápido** |
| Remessa | `type` vazio vira `Commercial`; `nfkey`, `cost_center_code` e `biller` são repassados quando informados |
| Recusa por regra de negócio | O rascunho volta para `incomplete`, com a mensagem da cotação/remessa em `status_descricao` |
| Falha inesperada | O rascunho vai para `error`. Remova-o e crie um novo |

### Mensagens possíveis em `status_descricao`
Além das mensagens de validação de campos (ex.: `O campo address sender.country code é obrigatório.`),
podem aparecer:

| Mensagem | Causa | Como corrigir |
|---|---|---|
| `Nenhum item foi enviado para este rascunho` | Sem `items` (e sem kits) | Enviar `items` |
| `O item com o sku :sku não foi encontrado!` | SKU não cadastrado e item sem dados completos | Cadastrar o item ou enviar os campos manuais |
| `Nenhuma caixa foi enviado para este rascunho` | `boxes` vazio, ou `box_code`/`box_identifier` não encontrado | Conferir as caixas |
| `Parâmetros da caixa não foram prenchidos corretamente por favor certifique-se dos seguintes campos (name,height,width,depth)` | Caixa manual sem `name`, `height`, `width` ou `depth` | Completar os quatro campos |
| `Nenhum item foi associado para a caixa :name !` | Caixa sem item com o `box_id` dela | Ajustar `items[].box_id` |
| `Não há itens válidos, verifique os dados e/ou o codigo SKU` | Sem `boxes`, nenhum item utilizável | Conferir os itens e SKUs |
| `O canal especificado não foi encontrado!` | `channel_code` não pertence à sua conta | Corrigir o `channel_code` |
| `O kit :sku é inválido!` | Kit inexistente ou inativo | Conferir o SKU do kit |
| `A caixa do kit :sku é inválida!` | A caixa do kit não foi encontrada | Revisar o cadastro do kit |
| `Um ou mais itens do kit :sku são inválidos!` | Item do kit não encontrado | Revisar o cadastro do kit |
| Mensagens da cotação ou da remessa | Ex.: sem contratos disponíveis, restrições de carga | Ver [Cotação](api-cotacao-v1.md) e [Remessa](api-remessa-v1.md) |

## Erros comuns

| Status | Significado / Mensagem | Como corrigir |
|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | Só edite rascunhos `incomplete`; consulte o status no `GET` |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Conferir o `code`/`drafts_ids` (pode ser de outra conta ou já ter sido convertido em remessa e removido) |
| `403` | `Has no permission!` | Solicitar ao suporte a permissão da ação (criar, editar, listar ou remover rascunhos) |
| `422` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` + `data` com os campos | Corrigir os campos listados em `data` |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` (em `data.cost_center_code`) | Usar um centro de custo da sua conta ou de contas autorizadas |
| `422` | `O campo drafts é obrigatório.` / `O campo drafts.drafts ids é obrigatório.` (no `DELETE`) | Enviar `drafts.drafts_ids` com ao menos um código |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Repetir com backoff; se persistir, acionar o suporte |

Erros de autenticação (token inválido, conta bloqueada) estão em
[Primeiros Passos](api-primeiros-passos.md#erros-de-autenticação).

## Observações finais
- O campo **`status_descricao` é o seu guia**: ele diz o que impediu o rascunho de avançar.
- Informe sempre `channel_identifier_number`: é o que amarra o rascunho ao seu pedido, permite achar o
  `codigo` na listagem e consultar a remessa depois que o rascunho for removido.
- No `PUT`, **reenvie o rascunho completo**: o conteúdo anterior é substituído.
- Se a conta tem preset de imposto, o `tax` enviado é ignorado; confira o valor gravado no `GET`.
- O rascunho **não aceita** pagamento via `gateway`. Para esse cenário use
  [cotação](api-cotacao-v1.md) + [remessa](api-remessa-v1.md).
- Para o fluxo oposto (tudo validado de uma vez, documentos imediatos), veja [Manifesto](api-manifesto.md).
- Cadastros usados pelo rascunho: [Caixa](api-caixa-v1.md), [Itens](api-itens-v1.md), [Contratos](api-contrato-v1.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
