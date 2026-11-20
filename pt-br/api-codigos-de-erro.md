# Códigos de Erro

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Reúne, em um só lugar, as mensagens de erro e aviso que a API Pública devolve, agrupadas por módulo.
- Para cada mensagem: o código HTTP, o texto literal, a causa e como corrigir.
- As mensagens estão no idioma padrão da API (`pt`). Com `lang: en` ou `lang: es`, o texto vem traduzido,
  mas o código HTTP e o `status` são os mesmos.
- Placeholders como `:name`, `:sku` e `:address` são substituídos pelo valor real na resposta.
- O detalhamento de cada regra fica na página do módulo, linkada em cada seção.

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

## Fluxo de uso

Como tratar uma resposta que não é `200`:

```
HTTP 422 ──► leia data (campo → mensagens) ──► corrija o payload ──► reenvie
HTTP 4xx ──► leia message                   ──► corrija o cenário ──► reenvie
HTTP 500 ──► aguarde (backoff exponencial)  ──► repita            ──► persistiu? acione o suporte
```

1. Verifique o **código HTTP** primeiro: ele define a família do problema.
2. Em `422`, a `message` de topo é genérica — os erros reais estão em `data`.
3. Em `400`, `401`, `403`, `404` e `406`, a `message` descreve a regra de negócio que bloqueou a operação.
4. Só repita automaticamente erros `500`. Erros `4xx` se repetem enquanto o payload não for corrigido.

## Conceitos essenciais

### Envelope de erro

```json
{
  "status": "warning",
  "message": "Cotação já foi utilizada em outra remessa!"
}
```

| Campo | Descrição |
|---|---|
| `status` | `warning` para regra de negócio/validação; `error` para falha inesperada ou recusa do transportador |
| `message` | Texto do erro, traduzido conforme o header `lang` |
| `data` | Presente em `422` (mapa de campos) e em alguns erros do transportador (retorno bruto) |

### Erro de validação (`422`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "items.0.hscode": ["O campo items.0.hscode deve conter no mínimo 6 caracteres."],
    "address_sender.city": ["O campo address sender.city é obrigatório quando address sender.address code não está presente."]
  }
}
```

> A `message` do `422` é sempre a mesma mensagem genérica. **Use `data`** para mostrar o erro ao usuário.

### Mensagens de validação mais frequentes em `data`

| Mensagem (padrão) | Significado |
|---|---|
| `O campo :attribute é obrigatório.` | Campo obrigatório ausente |
| `O campo :attribute é obrigatório quando :other não está presente.` | Obrigatoriedade condicional (ex.: endereço manual sem `address_code`) |
| `O campo :attribute não contém um valor válido.` | Valor fora da lista aceita (enum) |
| `O valor selecionado para o campo :attribute é inválido.` | Código de cadastro inexistente (`channel_code`, `box_code`, `address_code`, `quotation`...) |
| `O campo :attribute deve conter no mínimo :min caracteres.` / `não pode conter mais de :max caracteres.` | Tamanho de texto fora do limite |
| `O campo :attribute deve conter um número inteiro.` / `deve conter um valor numérico.` | Tipo numérico incorreto |
| `O campo :attribute não contém um endereço de email válido.` | E-mail inválido |
| `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` não liberado |

## Erros por status HTTP

| Status | `status` típico | Família | O que fazer |
|---|---|---|---|
| `308` | - | Chamada sem prefixo de versão | Usar `/v1/...` ou `/v2/...` |
| `400` | `warning` / `error` | Regra de negócio | Ler `message` e corrigir o cenário |
| `401` | `warning` | Tipo de pagamento não habilitado | Habilitar a opção de pagamento na conta |
| `403` | `warning` | Sem permissão, conta bloqueada, cotação inelegível | Solicitar permissão / refazer a cotação |
| `404` | `warning` / `error` | Recurso não encontrado | Conferir identificadores e cadastros |
| `406` | `warning` | Conflito de negócio | Ler `message` e ajustar o fluxo |
| `422` | `warning` | Validação de campos (ou regra sobre dados) | Corrigir os campos listados em `data` |
| `500` | `error` | Falha inesperada | Repetir com backoff; se persistir, acionar o suporte |

## Erros de autenticação e permissão

Valem para **todos** os endpoints.

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Header `Authorization` ausente ou token inválido | Enviar `Authorization: Bearer SEU_TOKEN` com um token válido |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Conta bloqueada | Acionar o suporte |
| `403` | `Has no permission!` | O perfil não tem a permissão da ação (cotar, remeter, consultar documentos etc.) | Solicitar a permissão ao suporte |
| `400` | Mensagem variável (`status: "error"`) | Falha inesperada ao validar o token | Repetir; se persistir, acionar o suporte |

Detalhes em [Primeiros Passos](api-primeiros-passos.md#autenticação).

## Erros de cotação

`POST /v1/quotation`, `POST /v2/quotation` e `GET /v1/quotation/{quotation}/{level}`.
Detalhes em [Cotação (v1)](api-cotacao-v1.md#regras-de-negócio).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Caixa sem item em `advanced` ou `items` (v2) | Associar ao menos um item (`box_id`) a cada caixa |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` fora do intervalo de caixas | Usar a posição (1, 2, 3...) da caixa em `boxes[]` |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Item manual sem `box_id` | Informar `box_id` ou `sku` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | Peso acima do máximo da caixa cadastrada | Reduzir o peso ou trocar de caixa |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Soma dos itens acima do máximo da caixa cadastrada | Redistribuir os itens |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inexistente ou de outra conta | Conferir o [cadastro de caixas](api-caixa-v1.md) |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` não cadastrado e item incompleto | Cadastrar o item ou enviar os dados manuais |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | `unit_value` igual ou menor que zero | Informar valor positivo |
| `400` | `O kit :sku é inválido!` | Kit inexistente/inativo | Revisar o kit no painel |
| `400` | `A caixa do kit :sku é inválida!` | Caixa do kit inexistente | Revisar o kit no painel |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Item do kit removido | Revisar o kit no painel |
| `400` | `Para usar SmartBox, informe ao menos uma caixa em boxes[].` | `type: items` (v2) com agrupamento `smartbox` sem `boxes[]` | Enviar as caixas candidatas |
| `400` | `Nenhuma caixa comporta o item :name (Smart Box).` | Nenhuma caixa enviada comporta o item | Enviar uma caixa maior |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Origem e destino idênticos | Revisar os endereços |
| `400` | `Nenhum endereco encontrado para o usuário especificado!` | `address_code` inexistente ou de outra conta | Conferir o código do endereço |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP brasileiro inválido (v1) | Corrigir o CEP ou enviar `state_code` |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Nenhum transportador retornou resultado | Revisar o cenário ou falar com o comercial |
| `400` | `Não foi encontrada a cotação especificada!` | `GET` de cotação inexistente ou de outra conta | Conferir o `quotation` |
| `400` | `Nivel de detalhe da cotação não existe` | `level` inválido no `GET` | Usar `simple` ou `detailed` |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` de outra conta | Usar um canal da sua conta |
| `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` | Endereço brasileiro sem CEP (v1) | Informar `postal_code` |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Endereço internacional não localizado (v1) | Revisar cidade/CEP/estado |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Estado/província não reconhecido | Enviar `state_code` válido |
| `404` | `O item com o sku :name não existe!` | `type: items` (v2) com SKU não cadastrado | Cadastrar o item |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Agrupamento `default`: caixa vinculada ao item inexistente | Revisar o vínculo item → caixa |
| `404` | `Nenhum transportador encontrado para este usuário!` | Conta sem contrato para a rota/objeto | Falar com o comercial |
| `404` | `Não há contratos que atendem as especificações do pacote enviado. Se você possui duvidas sobre restrições de envios (como dimensões e pesos), por favor contate o suporte para atendimento!` | Nenhum contrato cobre peso/dimensão/rota | Revisar a carga ou falar com o suporte |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado. Fale com nossa equipe pelo chat para recebermos seu pedido especial.` | Carga fora da faixa automática | Falar com o time comercial |
| `406` | `Codigo de endereço do remetente não encontrado!` | Tipo de conta exige remetente cadastrado | Enviar `address_sender.address_code` |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | HS Code sem tabela de imposto no destino | Revisar o `hscode` |
| `400` | `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³` | Pallet sem medidas e sem `volume` | Enviar as medidas ou `volume` |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Falha inesperada | Repetir com backoff |

> Alertas (não bloqueiam): `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` e
> `Valores declarados dos itens podem ocasionar problemas aduaneiros`, em `data.warnings`.

## Erros de remessa

`POST /v1/shipping` e `POST /v2/shipping`. Detalhes em [Remessa (v1)](api-remessa-v1.md#cotações-inelegíveis-para-remessa)
e [Remessa (v2)](api-remessa-v2.md#regras-de-negócio).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inexistente ou de outra versão (em `data.quotation`) | Usar a remessa da mesma versão da cotação |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | Cotação de outra conta | Usar o token da conta dona |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` | Conta travada em DDP com cotação DDU não paga | Refazer a cotação com `tax: sender` |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Documento federal ausente | Informar `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brasileiro sem inscrição estadual | Informar `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ inválido | Corrigir o CNPJ |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF inválido | Corrigir o CPF |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | Pessoa física com IE isenta | Informar IE válida |
| `406` | `Cotação já foi utilizada em outra remessa!` | Cotação reutilizada | Fazer nova cotação |
| `403` | `Cotação Simples é inelegível para remessa!` | `simple` de `package`/`pallet` | Cotar com `advanced` (ou `items`, na v2) |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` ultrapassado | Fazer nova cotação |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fora de `carriers[]` | Usar um contrato da cotação |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` de contrato inativo | Usar outro contrato |
| `401` | `Opção de pagamento não autorizado!` | Tipo de pagamento não habilitado | Habilitar na conta ou usar outro tipo |
| `400` | `Numero da nota fiscal é requerido` | Contrato exige `nfkey` | Informar a chave da NF-e |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` inválido | Conferir o código |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` incompatível | Usar o endereço cotado |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` incompatível | Usar o endereço cotado |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` incompatível (na v2, cotação feita só com país) | Usar o endereço cotado / endereço manual |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP inválido (v2) | Corrigir o CEP |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Endereço internacional não localizado (v2) | Revisar o endereço |
| `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Estado/província não reconhecido (v2) | Enviar `state_code` válido |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Falha inesperada | Repetir com backoff |

## Erros de atualização, anexos e cancelamento

`PUT /v1/shipping/{shipping}`, `DELETE /v1/shipping/{shipping}/docs` e `DELETE /v1/shipping/...`.
Detalhes em [Remessa (v1)](api-remessa-v1.md#endpoint-atualizar-remessa).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | `PUT` sem `nfkey`, `attachments` nem `cost_center_code` | Enviar ao menos um campo |
| `400` | `Status da remessa não permite atualização na NFE.` | Remessa além de `shipped` | Não é possível vincular a NF-e |
| `400` | `Erro ao salvar arquivo!` | `base64` inválido | Recodificar o arquivo |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Anexo acima de 5 MB | Reduzir o arquivo |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` inválido ou de outra conta (`PUT`); AWB/NF-e não localizado | Conferir o identificador |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Remessa cancelada/encerrada | Não é possível anexar |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Formato não aceito | Converter o arquivo |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML malformado | Corrigir o XML |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Falha inesperada no `PUT` | Repetir com backoff |
| `400` | `Remessa não encontrada!` | Remoção de anexo com UUID sem remessa | Conferir o UUID |
| `400` | `Esta remessa não foi encontrada!` | Cancelamento com identificador inválido ou de outra conta | Conferir o identificador |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | Status atual não permite cancelar | Ver a tabela de cancelamento em [Remessa (v1)](api-remessa-v1.md#endpoint-cancelar-remessa) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Falha inesperada no cancelamento | Repetir com backoff |

> Na remoção de anexos, falhas individuais vêm em `data.failed[]` com `reason` (ex.: `Nenhum documento encontrado!`),
> mesmo com HTTP `200`.

## Erros de rastreio e documentos

`GET /v1/shipping/.../status` e `GET /v1/shipping/.../docs`.
Detalhes em [Rastreio de Remessa](api-rastreio.md#erros-comuns) e [Documentos de Remessa](api-documentos.md#erros-comuns).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `400` | `remessa.status.status_not_found: ` | Status: nenhuma remessa para o UUID/AWB informado (texto sem tradução) | Usar o UUID da cotação ou o AWB correto |
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Status: remessa de outra conta ou canal sem remessa | Conferir o identificador/canal |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Remessa ainda sem registro de pagamento | Aguardar; se persistir, acionar o suporte |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Falha inesperada no status | Repetir com backoff |
| `400` | `Remessa não encontrada!` | Documentos: UUID sem remessa | Conferir o UUID |
| `404` | `Esta remessa não foi encontrada!` | Documentos: AWB/NF-e não localizado | Conferir ou usar o UUID |
| `400` | `Nenhum documento encontrado com o código especificado!` | Documentos ainda não gerados, filtro sem resultado ou canal sem remessa | Aguardar `shipped` / revisar `documents` |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Falha inesperada nos documentos | Repetir com backoff |

## Erros de agendamento de coleta

`POST /v1/schedule`, `GET /v1/schedule/{shipping}` e `DELETE /v1/schedule/{shipping}/{reason}`.
Detalhes em [Agendamento de Coleta](api-agendamento.md#erros-comuns).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Data de hoje ou passada | Usar uma data a partir de amanhã |
| `400` | `Coleta não pode ser realizada em um final de semana` | Sábado ou domingo | Escolher um dia útil |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Feriado nacional | Escolher outro dia útil |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Data muito distante | Reduzir a antecedência |
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Já existe agendamento para a remessa | Consultar com `GET` |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Remessa fora de `shipped` | Aguardar `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Transportador sem coleta integrada | Levar o volume a um ponto de postagem |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Endereço de coleta incompleto | Completar o endereço |
| `400` | `O atributo federal_tax_id não é válido` | CPF/CNPJ inválido | Corrigir o documento |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Estado de outro país | Usar endereço do país do remetente |
| `400` | `Agendamento não foi encontrado para esta remessa` | AWB não localizado (`POST`/`DELETE`) | Conferir o AWB ou usar o UUID |
| `404` | `Agendamento não foi encontrado para esta remessa` | `GET` sem agendamento | Conferir o identificador |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Cancelamento fora do status permitido | Cancelar só agendamentos confirmados |
| `422` | `O campo reason não contém um valor válido.` (em `data`) | `reason` fora de `001` a `008` | Usar um motivo válido |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Falha inesperada (inclui chave ausente em `address`) | Enviar todas as chaves de `address`; repetir |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Falha inesperada no cancelamento | Repetir; acionar o suporte |

## Erros de manifesto

`POST /v1/manifest`. Detalhes em [Manifesto](api-manifesto.md#erros-comuns).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `400` | `Tipo de pagamento não autorizado!` | Conta sem pagamento faturado | Habilitar o faturamento |
| `400` | `O canal especificado não foi encontrado!` | `channel_code` de outra conta | Usar um canal da conta |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` inativo | Usar outro contrato |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Estado não resolvido | Enviar `state_code` |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | `address_code` não visível para a conta | Conferir o código |
| `404` | `A transportadora informada não está disponível para esta remessa!` | `carrier_code` sem manifesto | Omitir ou trocar o `carrier_code` |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Contrato sem despacho síncrono | Acionar o suporte |
| `400` | Retorno do transportador (`status: "error"`, com `data`) | O transportador recusou o envio | Corrigir conforme o retorno |
| `400` | `MANIFEST WARNING - AWB cannot be generated, carrier not mapped` | Etiqueta não gerada após o aceite; a remessa existe | Buscar em Documentos; **não** repetir |

> Erros da etapa de cotação dentro do manifesto (caixas, itens, kits, contratos) voltam como `400 warning`,
> com as mesmas mensagens da [cotação](#erros-de-cotação).

## Erros de rascunho

`POST /v1/draft`, `PUT /v1/draft/{code}`, `GET /v1/draft` e `DELETE /v1/draft`.
Detalhes em [Rascunho](api-rascunho.md#erros-comuns).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | `PUT` em rascunho fora de `incomplete` | Consultar o status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Código inexistente, de outra conta ou já convertido em remessa | Conferir o código |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Falha inesperada | Repetir com backoff |

> Os erros do processamento assíncrono do rascunho não voltam na resposta HTTP: eles aparecem em
> `status_descricao` do rascunho (ex.: `Nenhum item foi enviado para este rascunho`).

## Erros de caixa e itens

`/v1/box` e `/v1/item`. Detalhes em [Caixa](api-caixa-v1.md#erros-comuns) e [Itens](api-itens-v1.md#erros-comuns).

| Status | Mensagem | Causa | Como corrigir |
|---|---|---|---|
| `422` | Validação (em `data`) | Campos de caixa inválidos; filtros de listagem inválidos | Corrigir os campos listados |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Falha inesperada | Repetir com backoff |
| `200` + relatório | `Box Error: box :id not found!` / `Box Error: box :id cannot be deleted, has related item` | Remoção de caixa recusada no processamento | Usar `box_code` inteiro / desvincular itens |
| `200` + relatório | `sku => :sku, status => invalid, reason => :campo` | Item descartado no processamento assíncrono | Corrigir o campo indicado |

> Cadastros de caixas e itens são **assíncronos**: a resposta `200` confirma o recebimento, e os registros
> inválidos são descartados depois, com relatório enviado ao e-mail da conta.

## Regras de negócio

### Política de nova tentativa
| Situação | Repetir automaticamente? |
|---|---|
| `500` | Sim, com backoff exponencial (ex.: 2 s, 4 s, 8 s) e limite de tentativas |
| `4xx` | Não. Corrija o payload/cenário antes |
| Manifesto com `400` após o aceite do transportador | **Não.** A remessa já existe: consulte [Rastreio](api-rastreio.md) e [Documentos](api-documentos.md) |
| Remessa (`POST /v*/shipping`) com timeout de rede | Antes de repetir, consulte o status pelo `quotation`: uma cotação só gera uma remessa |

### Recusas com `status: "error"`
Algumas recusas de negócio relacionadas a **peso** ou ao **transportador** vêm com `status: "error"` e código
`400`/`404`. Trate sempre pelo código HTTP e pela `message`, não apenas pelo `status`.

## Observações finais
- Convenções gerais de resposta em [Primeiros Passos](api-primeiros-passos.md#formato-padrão-das-respostas).
- Cotações que não podem virar remessa: [Cotação (v1)](api-cotacao-v1.md#cotações-inelegíveis-para-remessa).
- Em caso de dúvida, informe ao suporte o `quotation`, o horário da chamada e a `message` recebida.
