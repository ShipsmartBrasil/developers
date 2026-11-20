# Manifesto

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Executa **cotação + remessa + envio ao transportador + geração da etiqueta em uma única chamada síncrona**.
- A resposta já traz o identificador da remessa, o número de rastreio e a etiqueta (AWB) em PDF e/ou ZPL.
- É o fluxo indicado para operações que precisam da etiqueta **imediatamente**, sem esperar
  processamento assíncrono.
- Só considera contratos de transportadores com despacho síncrono suportado (atualmente **FedEx, UPS e DHL**).
- Só aceita **pagamento faturado** (`payment.type: billed`), e a conta precisa ter essa forma de pagamento habilitada.

> ⚠️ **O manifesto é síncrono e irreversível.** Como ele cota e despacha na mesma requisição, não há
> etapa intermediária de revisão. Valide bem o payload antes de enviar.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
Content-Type: application/json
Accept: application/json
lang: pt
```

> O header `lang` aceita `pt`, `en` e `es` (padrão `pt`). O perfil do token precisa ter a permissão de
> **criar manifesto**; sem ela a resposta é `403` com `Has no permission!`.

## Fluxo de uso
1. Monte o payload completo (endereços, caixas, itens, pagamento). Confira a seção
   [Endereço manual: envie todas as chaves](#endereço-manual-envie-todas-as-chaves).
2. `POST /v1/manifest?format=pdf` (ou `zpl`, ou sem `format` para receber os dois).
3. Imprima a etiqueta recebida em `data.docs` (use o `base64` ou baixe pela `url`).
4. Guarde o `data.shipping` e use-o em [rastreio](api-rastreio.md), [documentos](api-documentos.md) e
   [agendamento de coleta](api-agendamento.md).
5. Alguns instantes depois, consulte [Documentos de Remessa](api-documentos.md) para obter os documentos
   complementares (invoice, packing list etc.), gerados em segundo plano.

## Conceitos essenciais

### Manifesto vs. cotação + remessa

| Tema | Cotação + Remessa | Manifesto |
|---|---|---|
| Chamadas | 2 (`/quotation` + `/shipping`) | **1** (`/manifest`) |
| Escolha do transportador | Você escolhe na lista retornada | A plataforma escolhe (ou você indica com `carrier_code`/`slug`) |
| Transportadores | Todos os contratos da conta | **Somente FedEx, UPS e DHL** |
| Etiqueta (AWB) | Consulta posterior em `/docs` | **Vem na resposta** |
| Pagamento | `billed`, `gateway`, `afterward` | **Apenas `billed`** |
| Tipo de resposta | Assíncrono | **Síncrono** |
| Revisão antes do despacho | Sim | Não |
| Campos obrigatórios | Menos (divididos em 2 etapas) | **Mais** (tudo de uma vez) |

Use o manifesto quando: a operação é de alto volume, o transportador já está definido por regra de
negócio e você precisa imprimir a etiqueta na hora.

### Como o transportador é escolhido

| O que você envia | Comportamento |
|---|---|
| Nem `slug` nem `carrier_code` | A plataforma escolhe pela preferência configurada na conta: **mais rápido** (menor prazo, padrão) ou **mais barato** (menor valor na moeda de pagamento) |
| `carrier_code` | É uma **preferência**. Se o contrato aparecer no resultado da cotação, ele é usado. Se não aparecer: contrato inexistente ou de transportador compatível com o manifesto → a plataforma cai na preferência da conta; contrato existente de transportador **não** compatível com o manifesto → `404` |
| `slug` | **Fixa** o contrato: a cotação avalia apenas ele e o `slug` **sobrescreve** qualquer `carrier_code` enviado. Se o contrato não estiver entre os contratos elegíveis da sua conta para aquele envio, a cotação volta a avaliar todos e a escolha segue a regra do `carrier_code` |

> Não misture `slug` e `carrier_code`: use um ou outro. Se enviar os dois, o `slug` prevalece.
> Quando a conta tem um contrato preferido predefinido e você não envia `carrier_code`, a predefinição é usada como `carrier_code`.

### Predefinições da conta no manifesto

O manifesto aplica as [predefinições da conta](api-primeiros-passos.md#predefinições-da-conta-presets)
**depois** da validação de entrada. Por isso, todos os campos obrigatórios precisam estar no payload,
mesmo que a conta tenha predefinição para eles.

| Predefinição | Quando é aplicada no manifesto |
|---|---|
| Responsável pelo imposto (`tax`) | **Sempre que estiver configurada: substitui o `tax` enviado** (veja a tabela abaixo) |
| Centro de custo (`cost_center_code`) | Quando o payload não envia o campo ou o envia vazio (`null`, `""` ou `0`) |
| Canal de venda (`channel_code`) | Quando a chave `channel_code` não é enviada. Enviar `"channel_code": null` desativa a predefinição |
| Contrato preferido (`carrier_code`) | Quando a chave `carrier_code` não é enviada |
| Frete declarado (`declared_freight`) | Quando a chave `declared_freight` não é enviada. Enviar `declared_freight` (mesmo `null` ou `{}`) descarta a predefinição |
| Faturante (`biller`) | Quando a exibição do faturante está habilitada na conta e o payload não envia `biller`, ou o envia vazio (`null`, `{}` ou só com valores vazios). Um `biller` preenchido no payload sempre prevalece |
| Demais predefinições (`insurance`, `measurement`, `type`, moedas, entrega residencial, não empilhável, endereço do remetente) | **Não têm efeito**: esses campos são obrigatórios no manifesto e o valor do payload sempre prevalece |

**Resolução do responsável pelo imposto (`tax`):**

| Configuração da conta | `tax` efetivamente usado |
|---|---|
| Sem predefinição de imposto | O valor enviado no payload |
| Padrão `sender` ou `receiver` | O padrão da conta, **ignorando** o valor enviado |
| Padrão + país de destino na lista de exceções da conta | O **oposto** do padrão (`sender` ↔ `receiver`) |
| Modo travado | Sempre `sender`, independentemente do payload e das exceções |

> O país de destino usado nas exceções é o `address_receiver.country_code` do payload. Se o destinatário
> for informado só por `address_code`, as exceções por país não são avaliadas e vale o padrão da conta.
> Mesmo quando a predefinição substitui o valor, `tax` continua obrigatório no payload e precisa ser `sender` ou `receiver`.

### Etapas do processamento e o que acontece em caso de falha

| Etapa | Se falhar |
|---|---|
| 1. Validação de entrada | `422`. Nada é criado |
| 2. Regras da conta e complementação (pagamento, canal, `slug`, endereços, caixas, itens e kits cadastrados, estado) | `400`/`404`. Nada é criado |
| 3. Cotação, registro da remessa e envio ao transportador | Tudo é desfeito: nenhuma cotação ou remessa fica registrada na conta |
| 4. Geração da etiqueta e dos documentos (depois de o transportador aceitar o envio) | A remessa **permanece criada**, com número de rastreio, e recebe um evento de erro de processamento no rastreio. Recupere os documentos em [Documentos de Remessa](api-documentos.md) e não repita o manifesto |

## Endpoint: criar manifesto

- **Método e rota:** `POST /v1/manifest`
- **Quando usar:** para criar, em uma única chamada, uma remessa já despachada ao transportador e
  receber a etiqueta na resposta.

### Parâmetros — query

| Nome | Local | Tipo | Obrigatório | Padrão | Valores | Descrição |
|---|---|---|---|---|---|---|
| `format` | query | string | não | vazio (todos) | `pdf`, `zpl` | Formato da etiqueta na resposta. Vazio (ou valor não reconhecido) retorna PDF **e** ZPL |

> `format` também pode ser enviado **no corpo** do payload. Quando enviado nos dois lugares,
> **o valor do corpo prevalece**. No corpo, valores diferentes de `pdf`/`zpl` são rejeitados com `422`.

### Parâmetros — raiz do payload

| Campo | Tipo | Obrigatório | Mín. | Máx. | Valores / Padrão | Descrição |
|---|---|---|---|---|---|---|
| `object` | string | **sim** | - | - | `doc`, `package`, `pallet`, `not_doc` | O que está sendo enviado. `not_doc` é aceito por compatibilidade e tratado como `package` |
| `type` | string | **sim** | - | - | `Commercial`, `Gift`, `Personal, Not for Resale`, `Return for repair`, `Return after repair`, `Samples`, `Consigned` | Finalidade do envio (valor exato, com maiúsculas e espaços) |
| `tax` | string | **sim** | - | - | `sender`, `receiver` | Quem paga os impostos. Pode ser substituído pela predefinição da conta (veja [Predefinições](#predefinições-da-conta-no-manifesto)) |
| `insurance` | boolean | **sim** | - | - | - | Contratar seguro. Pode ser ativado automaticamente se a conta tiver contratação automática de seguro e o valor declarado atingir o mínimo configurado |
| `residential_delivery` | boolean | **sim** | - | - | - | Entrega em endereço residencial |
| `non_stackable` | boolean | **sim** | - | - | - | Volumes não empilháveis |
| `measurement` | string | **sim** | - | - | `metric`, `imperial` | Sistema de medida de pesos e dimensões |
| `ior` | boolean | **sim** | - | - | - | Importer of Record |
| `currency_quote` | string | **sim** | 3 | 3 | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda da cotação (e dos valores dos itens) |
| `currency_payment` | string | **sim** | 3 | 3 | ver [moedas](api-primeiros-passos.md#moedas-aceitas) | Moeda do pagamento |
| `address_sender` | object | **sim** | - | - | - | Remetente (ver tabela) |
| `address_receiver` | object | **sim** | - | - | - | Destinatário (ver tabela) |
| `boxes` | array | **sim** | 1 item | - | - | Volumes (ver tabela) |
| `payment` | object | **sim** | - | - | - | Pagamento (ver tabela) |
| `items` | array | condicional | - | - | - | Obrigatório quando `object` **não** é `doc` e nenhuma caixa usa `kit`. **Proibido (a chave não pode existir) quando `object` é `doc`** |
| `carrier_code` | integer | não | - | - | - | Contrato preferido (veja [Como o transportador é escolhido](#como-o-transportador-é-escolhido)) |
| `slug` | string | não | - | 20 | - | Identificador textual que fixa o contrato. Precisa existir; contrato removido ou fora da vigência retorna `400` |
| `channel_code` | integer | não | - | - | - | Código do canal de venda cadastrado **na sua conta** |
| `channel_identifier_number` | string | não | - | - | - | Seu identificador comercial (ex.: número do pedido). **Recomendado** |
| `cost_center_code` | integer | não | - | - | Predefinição da conta | Centro de custo cadastrado para a sua conta ou para contas associadas autorizadas. Vazio/ausente → usa a predefinição da conta |
| `nfkey` | string | não | 44 | 44 | - | Chave da NF-e (exatamente 44 caracteres) |
| `description` | string | não | 5 | 155 | - | Descrição livre do manifesto |
| `declared_freight` | object | não | - | - | Predefinição da conta | Frete declarado (ver tabela) |
| `biller` | object | não | - | - | Predefinição da conta | Faturante exibido na invoice (ver seção) |
| `manufacturer` | object | não | - | - | - | Fabricante exibido na invoice (ver seção) |
| `extra` | object | não | - | - | - | Dados auxiliares em pares chave/valor (ver tabela) |
| `format` | string | não | - | - | `pdf`, `zpl` | Alternativa à query string (prevalece sobre ela) |
| `size` | string | não | - | - | `label`, `letter` | Aceito na validação, mas **atualmente não altera** a etiqueta gerada |

### Parâmetros — `address_sender` e `address_receiver`

Diferente da remessa v1/v2, aqui o endereço é informado **completo**, porque a cotação e o despacho
acontecem na mesma chamada. Há duas formas de informar cada endereço — **não misture as duas no mesmo objeto**.

**Forma A — endereço já cadastrado:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `address_code` | integer | **sim** nesta forma | Código de um endereço cadastrado e visível para a sua conta |

> Com `address_code`, **todos** os dados do endereço vêm do cadastro; qualquer outro campo enviado no mesmo
> objeto é descartado. Se você enviar `address_code` junto com `country_code`, `street`, `name`, `email`,
> `phone`, `type` e `foreign`, a requisição é rejeitada com `422`. Remetente e destinatário não podem usar o
> mesmo `address_code`.

**Forma B — endereço manual:**

| Campo | Tipo | Obrigatório | Mín. | Máx. | Padrão | Descrição |
|---|---|---|---|---|---|---|
| `country_code` | string | **sim** | 2 | 2 | - | País em ISO 3166-1 alpha-2 |
| `postal_code` | string | condicional | - | 12 | - | CEP/ZIP. Obrigatório se `state` e `state_code` não forem informados. No Brasil, é usado para descobrir o estado quando `state_code` não é enviado |
| `state` | string | condicional | - | 100 | - | Nome do estado. Obrigatório se `state_code` e `postal_code` não forem informados |
| `state_code` | string | condicional | 1 | 10 | - | Sigla do estado. Obrigatória se `state` e `postal_code` não forem informados. **Recomendado sempre** (veja as regras de negócio) |
| `city` | string | condicional | 1 | 100 | - | Cidade. Obrigatória se `postal_code` não for informado. **Recomendada sempre**: os transportadores usam a cidade na etiqueta |
| `street` | string | **sim** | 5 | 70 | - | Logradouro |
| `number` | string | não | 1 | 6 | - | Número do imóvel |
| `district` | string | não | 1 | 30 | - | Bairro |
| `complement` | string | não | 1 | 20 | - | Complemento |
| `name` | string | **sim** | 1 | 70 | - | Nome do contato/razão social |
| `email` | string | **sim** | - | 60 | - | E-mail válido |
| `phone` | string | não | 1 | 25 | - | Telefone. **Recomendado sempre**: os transportadores exigem telefone de contato |
| `phone_extension` | string | não | - | 5 | - | Ramal |
| `type` | string | **sim** | - | - | - | `pf` (pessoa física) ou `pj` (pessoa jurídica) |
| `foreign` | boolean | **sim** | - | - | - | Indica se a pessoa é estrangeira (usado para validar o `federal_tax_id`) |
| `federal_tax_id` | string | condicional | - | 20 | - | Documento federal (CPF/CNPJ ou equivalente). **Obrigatório quando `country_code` é `BR` ou `AR`** |
| `state_tax_id` | string | não | - | 20 | - | Inscrição estadual |
| `description` | string | não | 1 | 155 | - | Rótulo interno do endereço |
| `company_name` | string | não | 5 | 50 | - | **Somente em `address_sender`.** Razão social usada em envios FedEx (o transportador considera apenas os 35 primeiros caracteres) |

#### Endereço manual: envie todas as chaves

No manifesto, as chaves abaixo precisam **existir** no objeto de endereço manual, mesmo quando você não tem
valor para elas — nesse caso, envie `null`. A ausência de alguma delas interrompe o processamento com erro
(`500`, ou `400` na etapa do transportador).

| Chaves que devem estar presentes (valor pode ser `null`) |
|---|
| `postal_code`, `city`, `number`, `district`, `complement`, `phone`, `phone_extension`, `federal_tax_id`, `state_tax_id` |

> Também não envie `"address_code": null` na forma B: a chave `address_code` deve ser omitida.

### Parâmetros — `boxes[]`

Cada caixa pode ser informada de três formas — **não misture as formas na mesma caixa**:

| Forma | Campos | Comportamento |
|---|---|---|
| A — caixa cadastrada | `box_code` **ou** `box_identifier` | Nome, dimensões e peso vêm do [cadastro de caixas](api-caixa-v1.md) |
| B — caixa manual | `name`, `height`, `width`, `depth` (+ `weight`/`price` em `doc`) | Dados informados no payload |
| C — kit | `kit` | A caixa **e** os itens daquela posição vêm do kit cadastrado |

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `kit` | string | não | - | 50 | SKU do kit cadastrado. Preenche a caixa e os itens daquela posição; `weight` e `price` da caixa são ignorados |
| `box_code` | integer | condicional | - | - | Código da caixa cadastrada. Obrigatório quando a caixa não informa nenhum outro campo |
| `box_identifier` | string | condicional | - | 50 | Identificador da caixa cadastrada. Alternativa ao `box_code` |
| `name` | string | **sim** sem `kit`/`box_code`/`box_identifier` | 5 | 155 | Nome da caixa |
| `height` | number | **sim** sem `kit`/`box_code`/`box_identifier` | 1 | - | Altura (cm em `metric`, in em `imperial`) |
| `width` | number | **sim** sem `kit`/`box_code`/`box_identifier` | 1 | - | Largura |
| `depth` | number | **sim** sem `kit`/`box_code`/`box_identifier` | 1 | - | Profundidade |
| `weight` | number | **sim** quando `object` é `doc` (inclusive com caixa cadastrada) | 0.001 | - | Peso do volume (kg em `metric`, lb em `imperial`) |
| `price` | number | **sim** quando `object` é `doc` (inclusive com caixa cadastrada) | 0 | - | Valor declarado do volume. Em `doc`, o valor declarado da remessa é a soma dos `price` |

> Com caixa cadastrada, os campos enviados no payload complementam (e sobrescrevem) os do cadastro.
> A caixa precisa pertencer à sua conta.

### Parâmetros — `items[]`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `box_id` | integer | **sim** | - | - | **Posição** da caixa no array `boxes` (1 = primeira) |
| `quantity` | integer | **sim** | 1 | 99 | Quantidade do item |
| `sku` | string | condicional | - | 50 | SKU do item cadastrado ([Itens](api-itens-v1.md)). Obrigatório quando nenhum de `name`, `country_code`, `hscode`, `weight` e `unit_value` é informado |
| `name` | string | **sim** sem `sku` | 6 | 50 | Nome do item |
| `country_code` | string | **sim** sem `sku` | 2 | 2 | País de origem do item (ISO 3166-1 alpha-2) |
| `hscode` | string | **sim** sem `sku` | 6 | 10 | NCM/HS Code |
| `weight` | number | **sim** sem `sku` | 0.001 | 99 | Peso unitário (kg em `metric`, lb em `imperial`) |
| `unit_value` | number | **sim** sem `sku` | maior que 0 | - | Valor unitário, na moeda `currency_quote`. A validação aceita `0`, mas a cotação recusa item com valor zero |
| `selling_unit_value` | number | não | 0 | - | Valor de venda unitário, usado quando `ior` é `true`. Se omitido, vem do cadastro do item (com `sku`) ou, na falta dele, assume o `unit_value` |
| `description` | string | não | 5 | 450 | Descrição detalhada do item |
| `ean` | string | não | - | 50 | Código EAN (código de barras) do item. Usado como identificador padronizado do fabricante nas informações regulatórias para destinos na União Europeia (e como identificador do produto quando não há `sku`) |
| `nsmp` | string | não | - | 50 | *Non Standard Manufacturer Product Id*: identificador de produto não padronizado do fabricante. É impresso na invoice e usado nas informações regulatórias exigidas por alguns transportadores para destinos na União Europeia |

> **Itens com `sku`:** quando o item informa `sku` e falta algum dos campos `name`, `country_code`, `hscode`,
> `weight`, `unit_value`, `nsmp` (ou `selling_unit_value` com `ior: true`), os dados ausentes são completados
> a partir do [cadastro do item](api-itens-v1.md) (incluindo `ean` e `nsmp`). Os valores enviados no payload
> sempre prevalecem sobre o cadastro. Se o SKU não estiver cadastrado, o item só é aceito se trouxer
> `name`, `country_code`, `hscode` e `weight` no próprio payload.
>
> As informações regulatórias para destinos na União Europeia só são montadas quando o item tem `sku` ou `ean`.

> **Atenção às diferenças em relação à cotação:** aqui `quantity` tem máximo **99** e
> `description` aceita até **450** caracteres.

### Parâmetros — `payment`

| Campo | Tipo | Obrigatório | Valores | Descrição |
|---|---|---|---|---|
| `payment` | object | **sim** | - | Envelope |
| `payment.type` | string | **sim** | **`billed`** | O manifesto aceita **somente** pagamento faturado. A conta também precisa ter o pagamento faturado habilitado; caso contrário, a resposta é `400` |

### Parâmetros — `declared_freight`

| Campo | Tipo | Obrigatório | Mín. | Máx. | Descrição |
|---|---|---|---|---|---|
| `value` | number | não | 0 | - | Valor do frete declarado. `0` é aceito |
| `currency` | string | **sim** quando `value` é informado | 3 | 3 | Moeda do frete declarado |
| `declared` | boolean | não | - | - | Aceito, mas **sem efeito** no manifesto: o frete declarado só é considerado quando `value` **e** `currency` são informados |

> Para declarar frete zero, envie `"value": 0` com a `currency`. Sem `value` e `currency`, o frete declarado
> não é aplicado (nem o da predefinição, se você enviou a chave `declared_freight`).

### Parâmetros — `extra`

Pares chave/valor auxiliares, gravados na remessa.

| Regra | Detalhe |
|---|---|
| Estrutura | Objeto **plano** (um nível, sem objetos ou listas aninhados) |
| Chaves | Apenas letras minúsculas, dígitos e `_` |
| Valores | Texto, inteiro ou decimal (booleano e `null` são rejeitados) |

| Chave conhecida | Tipo | Regra | Descrição |
|---|---|---|---|
| `federal_tax_provision` | number | - | Provisão de imposto federal |
| `state_tax_provision` | number | - | Provisão de imposto estadual |
| `usd_rate_provision` | number | maior que 0 | Provisão de câmbio do dólar. **Só é mantida** para contas do programa Remessa Conforme com destino `BR`; nos demais casos é descartada silenciosamente |

```json
"extra": {
  "federal_tax_provision": 10,
  "state_tax_provision": 7,
  "usd_rate_provision": 5.42
}
```

### Parâmetros — `biller` e `manufacturer`

- `biller`: mesmo formato descrito em [Remessa (v1)](api-remessa-v1.md#parâmetros--biller-faturante)
  (`address_code` **ou** campos manuais). No manifesto, `biller.name` tem de 5 a **355** caracteres.
  Campos não enviados assumem `type: pj` e `foreign: false`. Sem `biller` no payload, vale a
  predefinição de faturante da conta (quando habilitada).
- `manufacturer`: mesmo formato descrito em [Cotação (v1)](api-cotacao-v1.md#parâmetros--manufacturer)
  (`address_code` **ou** campos manuais). Com `address_code` de um endereço existente, os dados vêm do cadastro.

### Payload mínimo

Manifesto de documento (`object: doc`) com endereços manuais — `items` é proibido neste caso:

```json
{
  "object": "doc",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": {
    "country_code": "BR",
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "number": null,
    "district": null,
    "complement": null,
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11999990000",
    "phone_extension": null,
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": null,
    "foreign": false
  },
  "address_receiver": {
    "country_code": "US",
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "number": null,
    "district": null,
    "complement": null,
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550000",
    "phone_extension": null,
    "type": "pf",
    "federal_tax_id": null,
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Envelope documentos",
      "height": 2,
      "width": 23,
      "depth": 32,
      "weight": 0.3,
      "price": 50
    }
  ],
  "payment": {
    "type": "billed"
  }
}
```

**Variação mínima com endereços cadastrados e kit** (`object: package`; `items` pode ser omitido porque a caixa usa `kit`):

```json
{
  "object": "package",
  "type": "Commercial",
  "tax": "sender",
  "insurance": false,
  "residential_delivery": false,
  "non_stackable": false,
  "measurement": "metric",
  "ior": false,
  "currency_quote": "USD",
  "currency_payment": "BRL",
  "address_sender": { "address_code": 1234 },
  "address_receiver": { "address_code": 5678 },
  "boxes": [
    { "kit": "KIT-001" }
  ],
  "payment": { "type": "billed" }
}
```

### Payload máximo

Todos os campos aceitos. `slug` e `carrier_code` aparecem juntos apenas para ilustrar: na prática use um
ou outro (o `slug` prevalece). As caixas mostram a forma manual e a forma cadastrada.

```json
{
  "format": "pdf",
  "size": "label",
  "channel_code": 1,
  "channel_identifier_number": "PED-2026-0001",
  "cost_center_code": 12,
  "carrier_code": 45,
  "slug": "fedex-br",
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
  "description": "Manifesto do lote de marco",
  "declared_freight": {
    "declared": true,
    "value": 0,
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
    "company_name": "Loja Exemplo Comercio LTDA",
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
    "phone": "13055550000",
    "phone_extension": "45",
    "type": "pf",
    "federal_tax_id": "123456789",
    "state_tax_id": null,
    "foreign": true
  },
  "boxes": [
    {
      "name": "Caixa master 20x20x20",
      "height": 20,
      "width": 20,
      "depth": 20,
      "weight": 1.5,
      "price": 90
    },
    {
      "box_identifier": "CX0236"
    }
  ],
  "items": [
    {
      "box_id": 1,
      "sku": "TSH-000-S",
      "ean": "7891234567895",
      "nsmp": "MFR-TSH-000-S",
      "name": "Camiseta algodao P",
      "description": "Camiseta 100% algodao, manga curta, cor branca, tamanho P",
      "country_code": "BR",
      "hscode": "610910",
      "weight": 0.25,
      "quantity": 3,
      "unit_value": 29.9,
      "selling_unit_value": 49.9
    },
    {
      "box_id": 2,
      "name": "Tenis esportivo",
      "description": "Tenis de corrida masculino numero 44",
      "country_code": "BR",
      "hscode": "640411",
      "weight": 0.8,
      "quantity": 1,
      "unit_value": 199.9,
      "selling_unit_value": 349.9
    }
  ],
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
    "federal_tax_id": "11222333000181",
    "state_tax_id": "111222333444"
  },
  "payment": {
    "type": "billed"
  },
  "extra": {
    "federal_tax_provision": 10,
    "state_tax_provision": 7,
    "usd_rate_provision": 5.42
  }
}
```

### Exemplo de request

```bash
curl --request POST \
  --url 'https://api.shipsmart.global/v1/manifest?format=pdf' \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --header 'lang: pt' \
  --data '{
    "object": "doc",
    "type": "Commercial",
    "tax": "sender",
    "insurance": false,
    "residential_delivery": false,
    "non_stackable": false,
    "measurement": "metric",
    "ior": false,
    "currency_quote": "USD",
    "currency_payment": "BRL",
    "address_sender": {"country_code": "BR", "postal_code": "09941070", "city": "Diadema", "street": "Rua Principal", "number": null, "district": null, "complement": null, "name": "Loja Exemplo LTDA", "email": "contato@example.com", "phone": "11999990000", "phone_extension": null, "type": "pj", "federal_tax_id": "12345678000199", "state_tax_id": null, "foreign": false},
    "address_receiver": {"country_code": "US", "postal_code": "33101", "city": "Miami", "street": "Main Avenue", "number": null, "district": null, "complement": null, "name": "John Doe", "email": "johndoe@example.com", "phone": "13055550000", "phone_extension": null, "type": "pf", "federal_tax_id": null, "state_tax_id": null, "foreign": true},
    "boxes": [{"name": "Envelope documentos", "height": 2, "width": 23, "depth": 32, "weight": 0.3, "price": 50}],
    "payment": {"type": "billed"}
  }'
```

### Exemplo de response

`200 OK` (sem `format`, a resposta traz `AWB` e `AWB_ZPL`; com `format=pdf`, só `AWB`; com `format=zpl`, só `AWB_ZPL`):

```json
{
  "status": "success",
  "message": "Remessa criada com sucesso! Os documentos relacionados a mesma, ficarão disponíveis em nossa plataforma dentro de alguns minutos!",
  "data": {
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "carrier": "Fedex - Teste",
    "tracking_id": "271249639538",
    "tracking_packages_ids": [
      "271249639538"
    ],
    "docs": {
      "AWB": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.pdf",
        "base64": "JVBERi0xLjcK..."
      },
      "AWB_ZPL": {
        "url": "https://files.example.com/documentos/9c06fd75-d5f4-40eb-a1a8-f12086f49944_awb.zpl",
        "base64": "XlhBDQpeTUNZ..."
      }
    }
  }
}
```

### Campos da resposta

| Campo | Tipo | Presença | Descrição |
|---|---|---|---|
| `data.shipping` | string (UUID) | sempre | **Identificador da remessa.** Use em rastreio, documentos e agendamento |
| `data.carrier` | string | sempre | Nome do contrato/transportador que processou o envio |
| `data.tracking_id` | string | sempre | Número de rastreio principal (AWB) |
| `data.tracking_packages_ids` | array de string | quando o transportador informa | Número de rastreio de cada volume, na ordem das caixas |
| `data.docs` | object | sempre | Etiquetas geradas, no formato `nome → { url, base64 }` |
| `data.docs.AWB` | object | `format` vazio ou `pdf` | Etiqueta em PDF |
| `data.docs.AWB_ZPL` | object | `format` vazio ou `zpl` | Etiqueta em ZPL (impressoras térmicas) |
| `data.docs.AWBPRC` / `data.docs.AWBPRCZPL` | object | contas do programa Remessa Conforme, em importações (origem fora do Brasil e destino `BR`) | Etiqueta específica do programa, em PDF / ZPL |
| `data.docs.*.url` | string | sempre | Link do arquivo. Se a conta protege os links de documentos, o link é **temporário** (válido por cerca de 5 minutos) |
| `data.docs.*.base64` | string | sempre | Conteúdo do arquivo em Base64 |
| `data.valorProvCBS`, `data.valorProvIBSEstadual`, `data.valorProvIBSMunicipal` | number | quando o cálculo de impostos traz essas provisões | Provisões de CBS e IBS (estadual e municipal) |
| `data.serviceType` | string | envios FedEx | Serviço FedEx utilizado |
| `data.operationalDetail` | object | envios FedEx | Dados operacionais do transportador (`ursaPrefixCode`, `ursaSuffixCode`, `originLocationId`, `originLocationNumber`, `originServiceArea`, `destinationLocationId`, `destinationLocationNumber`, `destinationServiceArea`, `destinationLocationStateOrProvinceCode`, `deliveryDate`, `deliveryDay`, `commitDate`, `commitDay`, `ineligibleForMoneyBackGuarantee`, `astraPlannedServiceLevel`, `astraDescription`, `postalCode`, `stateOrProvinceCode`, `countryCode`, `airportId`, `serviceCode`, `packagingCode`, `publishedDeliveryTime`, `scac`). Campos sem valor vêm `null` |

> Documentos complementares — invoice (envios que não são `doc`), declaração de conteúdo (quando o
> remetente é `pf`), packing list e declaração de produtos não perigosos — são gerados em segundo plano.
> Consulte [Documentos de Remessa](api-documentos.md) alguns instantes depois para obter a lista completa.

### Resposta de erro do transportador

Quando o transportador recusa o envio, a resposta é `400` com `status: "error"` e traz o retorno bruto dele:

```json
{
  "status": "error",
  "message": "[1234] Fedex - Teste - Mensagem de erro do transportador",
  "data": {
    "output": [],
    "response": []
  }
}
```

| Campo | Descrição |
|---|---|
| `message` | `[código do contrato] nome do transportador - mensagem` retornada pelo transportador. Se o transportador responder sem mensagem reconhecível, vem `CARRIER SYNC - SHIPPING RETURN UNEXPECTED RESPONSE!` |
| `data` | Payload bruto do transportador. **A estrutura varia por transportador** e pode vir `null` |

> Registre `message` e `data` nos seus logs: é a única fonte do motivo técnico da recusa. Uma recusa do
> transportador desfaz a cotação e a remessa; corrija o payload e envie um novo manifesto.

## Regras de negócio
- **Pagamento:** somente `billed`, e a conta precisa ter o pagamento faturado habilitado (sozinho ou junto
  com o pagamento imediato). Outro valor em `payment.type` → `422`; conta sem faturamento → `400`.
- **Transportadores:** o manifesto cota apenas contratos FedEx, UPS e DHL da sua conta. A escolha segue
  [Como o transportador é escolhido](#como-o-transportador-é-escolhido).
- **Predefinições:** aplicadas depois da validação, conforme
  [Predefinições da conta no manifesto](#predefinições-da-conta-no-manifesto). Destaques: o `tax` da
  predefinição **substitui** o enviado; o `cost_center_code` da predefinição preenche o campo quando ele vem vazio.
- **`items` e `object: doc`:** com `object: doc`, a chave `items` **não pode** ser enviada (nem vazia).
  Com qualquer outro `object`, `items` é obrigatório — exceto quando alguma caixa usa `kit`.
- **Kits:** quando uma caixa informa `kit`, a caixa e os itens daquela posição vêm do cadastro,
  e `boxes[].weight`/`boxes[].price` são ignorados.
- **Valor declarado:** em `doc`, é a soma de `boxes[].price`; nos demais, a soma de `unit_value × quantity`
  dos itens. Item com valor unitário vazio ou zero é recusado na cotação (`400`).
- **`imperial`:** pesos e dimensões informados em libras/polegadas são convertidos para kg/cm antes da cotação.
- **Estado (`state_code`):** precisa ser uma sigla de estado reconhecida pela plataforma; para países sem
  divisão em estados, envie o próprio `country_code`. Sem `state_code`, a plataforma tenta descobri-lo
  (pelo CEP no Brasil, pelo nome em `state` ou pelo código postal no exterior). Se não conseguir para o
  remetente **ou** para o destinatário, o manifesto é recusado com `400` antes de qualquer registro.
- **`federal_tax_id`:** obrigatório quando o país do endereço é `BR` ou `AR`.
- **`address_code`:** o endereço precisa estar visível para a sua conta; com ele, os demais campos do
  objeto são ignorados. Remetente e destinatário não podem ter o mesmo `address_code`.
- **Endereço manual:** envie todas as chaves listadas em
  [Endereço manual: envie todas as chaves](#endereço-manual-envie-todas-as-chaves) (use `null` quando não houver valor).
- **`channel_code`:** precisa existir e pertencer à sua conta (inclusive quando vem da predefinição).
- **`cost_center_code`:** precisa existir e pertencer à sua conta, à conta principal ou a contas
  associadas autorizadas.
- **`slug`:** precisa existir (`422`) e o contrato precisa estar ativo e dentro da vigência (`400`).
- **Seguro automático:** se a conta contrata seguro automaticamente a partir de um valor mínimo e o valor
  declarado (convertido para USD) atingir esse mínimo, o seguro é contratado mesmo com `insurance: false`.
- **`extra.usd_rate_provision`:** mantido apenas para contas do programa Remessa Conforme com destino `BR`.
- **`not_doc`** é aceito por compatibilidade e tratado como `package`.

## Erros comuns

Erros de validação (`422`) têm `status: "warning"`, `message` igual a
`Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!`
e `data` com o mapa `campo → mensagens`. Erros de regra de negócio têm `status: "warning"` e não trazem `data`.

| Status | Significado / Mensagem | Como corrigir |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Conferir o token enviado em `Authorization` |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Acionar o suporte |
| `403` | `Has no permission!` | Solicitar ao suporte a permissão de criar manifesto |
| `422` | Validação de campos (mensagens em `data`, ex.: `O campo :attribute é obrigatório.`) | Corrigir os campos listados em `data` |
| `422` | `The :attribute field must be missing when :other is :value.` (em `items`) | Remover a chave `items` quando `object` é `doc` |
| `422` | `The :attribute field must be missing when :values are present.` (em `address_code`) | Usar só `address_code` **ou** só os campos manuais |
| `422` | `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | Usar um `cost_center_code` da sua conta |
| `400` | `Tipo de pagamento não autorizado!` | Solicitar ao suporte a habilitação do pagamento faturado |
| `400` | `O canal especificado não foi encontrado!` | Usar um `channel_code` cadastrado na sua conta (confira também a predefinição) |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | Usar um `slug` de contrato ativo |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Usar `address_code` diferentes |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | Conferir o `address_code` |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | Corrigir o `postal_code` ou enviar `state_code` |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Corrigir `state` ou enviar `state_code`/`postal_code` |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Enviar `state_code` nos dois endereços |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | Conferir `box_code`/`box_identifier` |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | Cadastrar o item ou enviar os campos manuais |
| `400` | `O kit :sku é inválido!` / `A caixa do kit :sku é inválida!` / `Um ou mais itens do kit :sku são inválidos!` | Revisar o cadastro do kit |
| `400` | Mensagem da cotação (ex.: `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. ...`, `Nenhum transportador encontrado para este usuário!`) | No manifesto, todo erro da etapa de cotação volta como `400`. Ler `message` e ajustar o payload |
| `400` | Mensagem do registro da remessa | Ler `message` e ajustar o payload |
| `404` | `A transportadora informada não está disponível para esta remessa!` | Usar `carrier_code` de um contrato FedEx, UPS ou DHL, ou omitir o campo |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | O contrato escolhido não tem despacho síncrono; acionar o suporte |
| `400` | `[código] transportador - mensagem` (`status: "error"`, `data` com o retorno do transportador) | Ver [Resposta de erro do transportador](#resposta-de-erro-do-transportador) |
| `500` | Erro inesperado (`status: "error"`) | **Antes de repetir**, confira em [Rastreio](api-rastreio.md) se a remessa foi criada |

> ⚠️ **Nunca repita um manifesto automaticamente após `500` ou após falha de rede/timeout.** Se a falha
> acontecer depois de o transportador aceitar o envio, a remessa existe de verdade e uma repetição gera um
> envio duplicado. Verifique primeiro pelo `channel_identifier_number` + `channel_code` em [Rastreio](api-rastreio.md).

## Observações finais
- Informe sempre `channel_identifier_number` (com `channel_code`): é a sua proteção contra duplicidade e a
  forma de localizar a remessa se a resposta se perder.
- O manifesto concentra muita responsabilidade em uma chamada. Para integrações que podem esperar,
  o fluxo de [cotação](api-cotacao-v1.md) + [remessa](api-remessa-v1.md) é mais seguro.
- Para o fluxo oposto (payload parcial, validado depois), veja [Rascunho](api-rascunho.md).
- Cadastros usados pelo manifesto: [Caixa](api-caixa-v1.md) e [Itens](api-itens-v1.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
