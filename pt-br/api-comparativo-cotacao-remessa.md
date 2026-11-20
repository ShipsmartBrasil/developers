# Cotação & Remessa: v1 vs v2

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
Existem duas versões publicadas para o par **cotação + remessa**. Elas usam o **mesmo cálculo**
e os **mesmos transportadores**. As diferenças estão em **quando** você informa o endereço completo e em
**como** descreve a carga: só a v2 aceita a cotação por itens, sem informar as caixas.

| | v1 | v2 |
|---|---|---|
| Endereço completo | Informado **na cotação** | Informado **na remessa** |
| Payload da cotação | País + cidade + CEP/estado | Apenas o país (+ estado opcional) |
| Cotação por itens (`type: items`) | Não disponível: caixas sempre informadas | **Disponível**: só o `sku` dos itens cadastrados |
| Payload da remessa | Logradouro, nome, contato | Logradouro, nome, contato **+ cidade, estado, CEP** |
| Validação do endereço | Na cotação | Na remessa |
| Ciclo operacional completo | Sim, em `/v1` | Criação em `/v2`, restante em `/v1` |

## Em uma frase

> **Use a v1** se, no momento de cotar, você já sabe para qual cidade/CEP vai enviar.
> **Use a v2** se, no momento de cotar, você só sabe o país de destino **ou** quer cotar apenas pelos SKUs
> cadastrados, sem informar as caixas.

## Regra de ouro: não misture versões

Uma cotação criada na v1 **só** gera remessa em `POST /v1/shipping`.
Uma cotação criada na v2 **só** gera remessa em `POST /v2/shipping`.

Misturar retorna `422`, com a mensagem abaixo em `data.quotation`:
```
Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.
```

## Comparativo da cotação

| Tema | v1 (`POST /v1/quotation`) | v2 (`POST /v2/quotation`) |
|---|---|---|
| `address_*.country_code` | Obrigatório | Obrigatório |
| `address_*.city` | **Obrigatório** | Ignorado |
| `address_*.postal_code` | Obrigatório (ou `state`); **sempre obrigatório no Brasil** | Ignorado |
| `address_*.state` | Obrigatório (ou `state_code`/`postal_code`) | Ignorado |
| `address_*.state_code` | Obrigatório (ou `state`/`postal_code`) | **Opcional** (melhora a precisão) |
| `object` | `doc`, `package`, `pallet`, `not_doc` | Idêntico |
| `type` | `simple`, `advanced` | `simple`, `advanced` **e `items`** (exclusivo da v2) |
| `boxes[]` (incl. `kit`, `volume`) | **Sempre obrigatório** | Obrigatório, exceto em `type: items` (conforme o agrupamento da conta) |
| `items[]` (incl. `ean`, `nsmp`) | Suportado | Idêntico, **sem** `selling_unit_value` |
| `ior` (Importer of Record) | Suportado | **Não faz parte do contrato** |
| `declared_freight` | Suportado | Idêntico |
| `manufacturer` | Suportado | Idêntico |
| `level` (`simple`/`detailed`) | Suportado | Idêntico |
| Validação do endereço (CEP, cidade, estado) | Na cotação | Não há (feita na remessa) |
| Consultar resultado depois | `GET /v1/quotation/{quotation}/{level}` | **Não existe** |
| Warnings aduaneiros | Suportado | Idêntico |
| Predefinições da conta (imposto, seguro automático etc.) | Aplicadas | Idêntico |
| Cenários (documento, pacote, pallet, itens, kit) | [Cenários da v1](api-cotacao-v1.md#cenários-de-cotação) | [Cenários da v2](api-cotacao-v2.md#cenários-de-cotação) |

### Impacto prático na cotação
- **Precisão:** na v1, cidade e CEP permitem aplicar taxas e restrições regionais **já no cálculo**.
  Na v2, o cálculo considera país e estado; o endereço completo entra só na remessa.
- **Erros de endereço:** na v1 aparecem na cotação; na v2 aparecem na remessa.
- **Reconsulta:** a v2 não tem endpoint de reconsulta. **Guarde a resposta do `POST`** se precisar
  reexibir a cotação ao usuário.

## Comparativo da remessa

| Tema | v1 (`POST /v1/shipping`) | v2 (`POST /v2/shipping`) |
|---|---|---|
| `quotation` | Cotação v1 | Cotação v2 |
| `carrier_code` / `slug` | Um dos dois | Idêntico |
| `address_*.street`, `name`, `email`, `phone`, `type` | Obrigatórios | Idênticos |
| `address_*.city` | Ignorado (vem da cotação) | **Obrigatório** (mín. 3, máx. 100) |
| `address_*.postal_code` | Ignorado | **Obrigatório** se `state` não for enviado |
| `address_*.state` | Ignorado | **Obrigatório** se `postal_code` não for enviado |
| `address_*.state_code` | Ignorado | Opcional; o da cotação prevalece |
| `address_*.address_code` | Precisa bater com o endereço da cotação | Só funciona se a cotação v2 usou o mesmo `address_code` |
| `payment.type` | `billed`, `gateway`, `afterward` | Idêntico |
| `nfkey`, `invoice_number`, `shipment_reference` | Suportados | Idênticos |
| `cost_center_code`, `payment_notify_shipping_billing` | Suportados | Idênticos |
| `attachments` | Suportado | Idêntico |
| `biller` | Suportado | Idêntico, com localidade validada |
| Elegibilidade da cotação | [Inelegíveis v1](api-remessa-v1.md#cotações-inelegíveis-para-remessa) | [Inelegíveis v2](api-remessa-v2.md#cotações-inelegíveis-para-remessa) |

## Operações pós-criação

Todas as operações após a criação da remessa usam **os mesmos endpoints `/v1`**, independentemente da
versão em que a remessa foi criada.

| Operação | Rota | Serve para remessa v1? | Serve para remessa v2? |
|---|---|---|---|
| Consultar status | `GET /v1/shipping/{shipping}/status` | Sim | **Sim** |
| Status por canal | `GET /v1/shipping/status/{channel_identifier_number}/{channel_code}` | Sim | **Sim** |
| Consultar documentos | `GET /v1/shipping/{shipping}/docs` | Sim | **Sim** |
| Documentos por canal | `GET /v1/shipping/docs/{channel_identifier_number}/{channel_code}` | Sim | **Sim** |
| Atualizar (NF-e, anexos, centro de custo) | `PUT /v1/shipping/{shipping}` | Sim | **Sim** |
| Remover anexos | `DELETE /v1/shipping/{shipping}/docs` | Sim | **Sim** |
| Cancelar | `DELETE /v1/shipping/{shipping}` | Sim | **Sim** |
| Cancelar por canal | `DELETE /v1/shipping/{channel_identifier_number}/{channel_code}` | Sim | **Sim** |
| Agendar coleta | `POST /v1/schedule` | Sim | **Sim** |
| Consultar agendamento | `GET /v1/schedule/{shipping}` | Sim | **Sim** |
| Cancelar agendamento | `DELETE /v1/schedule/{shipping}/{reason}` | Sim | **Sim** |

> Não existem rotas `GET`, `PUT` ou `DELETE` sob `/v2/shipping`.

## Comparação dos payloads lado a lado

### Cotação

**v1 — endereço completo na cotação:**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": {
    "country_code": "BR",
    "city": "Diadema",
    "postal_code": "09941070"
  },
  "address_receiver": {
    "country_code": "US",
    "city": "Miami",
    "state_code": "FL",
    "postal_code": "33101"
  },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "Camiseta algodao", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

**v2 — apenas país/estado na cotação:**
```json
{
  "object": "package",
  "type": "advanced",
  "tax": "sender",
  "insurance": true,
  "currency_quote": "USD",
  "currency_payment": "USD",
  "address_sender": { "country_code": "BR", "state_code": "SP" },
  "address_receiver": { "country_code": "US", "state_code": "FL" },
  "boxes": [{ "height": 20, "width": 20, "depth": 20 }],
  "items": [
    { "box_id": 1, "name": "Camiseta algodao", "country_code": "BR", "hscode": "610910", "weight": 0.25, "quantity": 3, "unit_value": 29.9 }
  ]
}
```

### Remessa

**v1 — sem localidade (veio da cotação):**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

**v2 — com localidade (não veio da cotação):**
```json
{
  "quotation": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
  "carrier_code": 1,
  "address_sender": {
    "postal_code": "09941070",
    "city": "Diadema",
    "street": "Rua Principal",
    "name": "Loja Exemplo LTDA",
    "email": "contato@example.com",
    "phone": "11989677465",
    "type": "pj",
    "federal_tax_id": "12345678000199",
    "state_tax_id": "123456789012",
    "foreign": false
  },
  "address_receiver": {
    "postal_code": "33101",
    "city": "Miami",
    "street": "Main Avenue",
    "name": "John Doe",
    "email": "johndoe@example.com",
    "phone": "13055550100",
    "type": "pf"
  },
  "payment": { "type": "billed" }
}
```

## Como escolher

Responda às perguntas em ordem:

1. **No momento de cotar, você já sabe a cidade e o CEP de destino?**
   - Sim → **v1**
   - Não → **v2**
2. **Você precisa reconsultar a cotação depois, em outra requisição?**
   - Sim → **v1** (`GET /v1/quotation/{quotation}/{level}`)
   - Não → qualquer uma
3. **Você usa `ior` (Importer of Record) ou `selling_unit_value`?**
   - Sim → **v1**
   - Não → qualquer uma
4. **Você quer o maior nível de precisão de impostos já na cotação?**
   - Sim → **v1**
   - Não → qualquer uma
5. **Você quer cotar só com os SKUs cadastrados, sem informar as caixas?**
   - Sim → **v2** ([Cotação de Itens](api-cotacao-v2.md#cotação-de-itens))
   - Não → qualquer uma

### Cenários típicos

| Cenário | Versão recomendada | Motivo |
|---|---|---|
| Checkout de e-commerce (endereço já preenchido) | **v1** | Cotação precisa, exibida ao cliente final |
| Calculadora de frete na página do produto (só país) | **v2** | O cliente ainda não informou o endereço |
| Marketplace que recebe o pedido completo via webhook | **v1** | Todos os dados já existem |
| Integração B2B com despacho em lote | **v1** | Fluxo operacional completo na mesma versão |
| Simulação em landing page internacional | **v2** | Payload mínimo, menos atrito |
| ERP com catálogo de produtos já cadastrado | **v2** | Cotação por itens: só SKU e quantidade, a plataforma monta as caixas |
| Cotação + remessa + documentos em uma chamada | **[Manifesto](api-manifesto.md)** | Fluxo síncrono, independente de v1/v2 |

## Outras formas de fazer remessa

Além do par cotação + remessa, existem dois fluxos alternativos:

| Fluxo | Rota | Característica |
|---|---|---|
| [Manifesto](api-manifesto.md) | `POST /v1/manifest` | **Síncrono**: cota, cria a remessa no transportador e devolve os documentos na mesma resposta. Só aceita pagamento `billed` |
| [Rascunho](api-rascunho.md) | `POST /v1/draft` | **Assíncrono**: aceita payload parcial, valida depois e evolui para cotação + remessa. Aceita pagamento `billed` ou `afterward` |

## Observações finais
- As duas versões convivem: você pode usar a v2 na vitrine e a v1 no checkout, desde que **cada cotação
  gere a remessa na sua própria versão**.
- Os endpoints de apoio ([Caixa](api-caixa-v1.md), [Itens](api-itens-v1.md),
  [Contrato](api-contrato-v1.md)) são compartilhados e existem apenas em `/v1`.
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
