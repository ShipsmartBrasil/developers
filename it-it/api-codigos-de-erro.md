# Codici di Errore

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Raccoglie, in un unico posto, i messaggi di errore e di avviso che l'API Pubblica restituisce, raggruppati per modulo.
- Per ogni messaggio: il codice HTTP, il testo letterale, la causa e come correggere.
- I messaggi sono nella lingua predefinita dell'API (`pt`). Con `lang: en` o `lang: es`, il testo arriva tradotto,
  ma il codice HTTP e lo `status` sono gli stessi.
- I placeholder come `:name`, `:sku` e `:address` vengono sostituiti dal valore reale nella risposta.
- Il dettaglio di ogni regola si trova nella pagina del modulo, collegata in ogni sezione.

## URL di base
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
Content-Type: application/json
lang: pt
```

## Flusso di utilizzo

Come gestire una risposta diversa da `200`:

```
HTTP 422 ──► leggi data (campo → messaggi) ──► correggi il payload  ──► reinvia
HTTP 4xx ──► leggi message                 ──► correggi lo scenario ──► reinvia
HTTP 500 ──► attendi (backoff esponenziale) ──► ripeti              ──► persiste? contatta il supporto
```

1. Verifica prima il **codice HTTP**: definisce la famiglia del problema.
2. Con `422`, la `message` di primo livello è generica — gli errori reali si trovano in `data`.
3. Con `400`, `401`, `403`, `404` e `406`, la `message` descrive la regola di business che ha bloccato l'operazione.
4. Ripeti automaticamente solo gli errori `500`. Gli errori `4xx` si ripetono finché il payload non viene corretto.

## Concetti essenziali

### Envelope di errore

```json
{
  "status": "warning",
  "message": "Cotação já foi utilizada em outra remessa!"
}
```

| Campo | Descrizione |
|---|---|
| `status` | `warning` per regola di business/validazione; `error` per errore imprevisto o rifiuto del corriere |
| `message` | Testo dell'errore, tradotto in base all'header `lang` |
| `data` | Presente in `422` (mappa dei campi) e in alcuni errori del corriere (risposta grezza) |

### Errore di validazione (`422`)

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

> La `message` del `422` è sempre lo stesso messaggio generico. **Usa `data`** per mostrare l'errore all'utente.

### Messaggi di validazione più frequenti in `data`

| Messaggio (predefinito) | Significato |
|---|---|
| `O campo :attribute é obrigatório.` | Campo obbligatorio assente |
| `O campo :attribute é obrigatório quando :other não está presente.` | Obbligatorietà condizionale (es.: indirizzo manuale senza `address_code`) |
| `O campo :attribute não contém um valor válido.` | Valore fuori dall'elenco accettato (enum) |
| `O valor selecionado para o campo :attribute é inválido.` | Codice di anagrafica inesistente (`channel_code`, `box_code`, `address_code`, `quotation`...) |
| `O campo :attribute deve conter no mínimo :min caracteres.` / `não pode conter mais de :max caracteres.` | Lunghezza del testo fuori limite |
| `O campo :attribute deve conter um número inteiro.` / `deve conter um valor numérico.` | Tipo numerico errato |
| `O campo :attribute não contém um endereço de email válido.` | E-mail non valida |
| `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` non autorizzato |

## Errori per status HTTP

| Status | `status` tipico | Famiglia | Cosa fare |
|---|---|---|---|
| `308` | - | Chiamata senza prefisso di versione | Usare `/v1/...` o `/v2/...` |
| `400` | `warning` / `error` | Regola di business | Leggere `message` e correggere lo scenario |
| `401` | `warning` | Tipo di pagamento non abilitato | Abilitare l'opzione di pagamento nell'account |
| `403` | `warning` | Senza permesso, account bloccato, quotazione non idonea | Richiedere il permesso / rifare la quotazione |
| `404` | `warning` / `error` | Risorsa non trovata | Verificare identificatori e anagrafiche |
| `406` | `warning` | Conflitto di business | Leggere `message` e adeguare il flusso |
| `422` | `warning` | Validazione dei campi (o regola sui dati) | Correggere i campi elencati in `data` |
| `500` | `error` | Errore imprevisto | Ripetere con backoff; se persiste, contattare il supporto |

## Errori di autenticazione e permessi

Valgono per **tutti** gli endpoint.

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Header `Authorization` assente o token non valido | Inviare `Authorization: Bearer IL_TUO_TOKEN` con un token valido |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Account bloccato | Contattare il supporto |
| `403` | `Has no permission!` | Il profilo non ha il permesso per l'azione (quotare, spedire, consultare documenti ecc.) | Richiedere il permesso al supporto |
| `400` | Messaggio variabile (`status: "error"`) | Errore imprevisto durante la validazione del token | Ripetere; se persiste, contattare il supporto |

Dettagli in [Primi Passi](api-primeiros-passos.md#autenticazione).

## Errori di quotazione

`POST /v1/quotation`, `POST /v2/quotation` e `GET /v1/quotation/{quotation}/{level}`.
Dettagli in [Quotazione (v1)](api-cotacao-v1.md#regole-di-business).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Scatola senza articoli in `advanced` o `items` (v2) | Associare almeno un articolo (`box_id`) a ogni scatola |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` fuori dall'intervallo delle scatole | Usare la posizione (1, 2, 3...) della scatola in `boxes[]` |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Articolo manuale senza `box_id` | Indicare `box_id` o `sku` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | Peso superiore al massimo della scatola registrata | Ridurre il peso o cambiare scatola |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Somma degli articoli superiore al massimo della scatola registrata | Ridistribuire gli articoli |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inesistente o di un altro account | Verificare l'[anagrafica delle scatole](api-caixa-v1.md) |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` non registrato e articolo incompleto | Registrare l'articolo o inviare i dati manuali |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | `unit_value` uguale o inferiore a zero | Indicare un valore positivo |
| `400` | `O kit :sku é inválido!` | Kit inesistente/inattivo | Rivedere il kit nel pannello |
| `400` | `A caixa do kit :sku é inválida!` | Scatola del kit inesistente | Rivedere il kit nel pannello |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Articolo del kit rimosso | Rivedere il kit nel pannello |
| `400` | `Para usar SmartBox, informe ao menos uma caixa em boxes[].` | `type: items` (v2) con raggruppamento `smartbox` senza `boxes[]` | Inviare le scatole candidate |
| `400` | `Nenhuma caixa comporta o item :name (Smart Box).` | Nessuna scatola inviata può contenere l'articolo | Inviare una scatola più grande |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Origine e destinazione identiche | Rivedere gli indirizzi |
| `400` | `Nenhum endereco encontrado para o usuário especificado!` | `address_code` inesistente o di un altro account | Verificare il codice dell'indirizzo |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP brasiliano non valido (v1) | Correggere il CEP o inviare `state_code` |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Nessun corriere ha restituito risultati | Rivedere lo scenario o parlare con il commerciale |
| `400` | `Não foi encontrada a cotação especificada!` | `GET` di una quotazione inesistente o di un altro account | Verificare il `quotation` |
| `400` | `Nivel de detalhe da cotação não existe` | `level` non valido nel `GET` | Usare `simple` o `detailed` |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` di un altro account | Usare un canale del tuo account |
| `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` | Indirizzo brasiliano senza CEP (v1) | Indicare `postal_code` |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Indirizzo internazionale non individuato (v1) | Rivedere città/CAP/stato |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Stato/provincia non riconosciuto | Inviare uno `state_code` valido |
| `404` | `O item com o sku :name não existe!` | `type: items` (v2) con SKU non registrato | Registrare l'articolo |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Raggruppamento `default`: scatola collegata all'articolo inesistente | Rivedere il collegamento articolo → scatola |
| `404` | `Nenhum transportador encontrado para este usuário!` | Account senza contratto per la rotta/oggetto | Parlare con il commerciale |
| `404` | `Não há contratos que atendem as especificações do pacote enviado. Se você possui duvidas sobre restrições de envios (como dimensões e pesos), por favor contate o suporte para atendimento!` | Nessun contratto copre peso/dimensione/rotta | Rivedere il carico o parlare con il supporto |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado. Fale com nossa equipe pelo chat para recebermos seu pedido especial.` | Carico fuori dalla fascia automatica | Parlare con il team commerciale |
| `406` | `Codigo de endereço do remetente não encontrado!` | Il tipo di account richiede un mittente registrato | Inviare `address_sender.address_code` |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | HS Code senza tabella di imposta nella destinazione | Rivedere l'`hscode` |
| `400` | `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³` | Pallet senza misure e senza `volume` | Inviare le misure o `volume` |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Errore imprevisto | Ripetere con backoff |

> Avvisi (non bloccano): `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` e
> `Valores declarados dos itens podem ocasionar problemas aduaneiros`, in `data.warnings`.

## Errori di spedizione

`POST /v1/shipping` e `POST /v2/shipping`. Dettagli in [Spedizione (v1)](api-remessa-v1.md#quotazioni-non-idonee-alla-spedizione)
e [Spedizione (v2)](api-remessa-v2.md#regole-di-business).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inesistente o di un'altra versione (in `data.quotation`) | Usare la spedizione della stessa versione della quotazione |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | Quotazione di un altro account | Usare il token dell'account proprietario |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` | Account vincolato a DDP con quotazione DDU non pagata | Rifare la quotazione con `tax: sender` |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Documento federale assente | Indicare `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brasiliano senza iscrizione statale | Indicare `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ non valido | Correggere il CNPJ |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF non valido | Correggere il CPF |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | Persona fisica con IE esente | Indicare una IE valida |
| `406` | `Cotação já foi utilizada em outra remessa!` | Quotazione riutilizzata | Fare una nuova quotazione |
| `403` | `Cotação Simples é inelegível para remessa!` | `simple` di `package`/`pallet` | Quotare con `advanced` (o `items`, nella v2) |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` superato | Fare una nuova quotazione |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` fuori da `carriers[]` | Usare un contratto della quotazione |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` di un contratto inattivo | Usare un altro contratto |
| `401` | `Opção de pagamento não autorizado!` | Tipo di pagamento non abilitato | Abilitarlo nell'account o usare un altro tipo |
| `400` | `Numero da nota fiscal é requerido` | Il contratto richiede `nfkey` | Indicare la chiave della NF-e |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` non valido | Verificare il codice |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` incompatibile | Usare l'indirizzo quotato |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` incompatibile | Usare l'indirizzo quotato |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` incompatibile (nella v2, quotazione fatta solo con il paese) | Usare l'indirizzo quotato / indirizzo manuale |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP non valido (v2) | Correggere il CEP |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Indirizzo internazionale non individuato (v2) | Rivedere l'indirizzo |
| `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | Stato/provincia non riconosciuto (v2) | Inviare uno `state_code` valido |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Errore imprevisto | Ripetere con backoff |

## Errori di aggiornamento, allegati e cancellazione

`PUT /v1/shipping/{shipping}`, `DELETE /v1/shipping/{shipping}/docs` e `DELETE /v1/shipping/...`.
Dettagli in [Spedizione (v1)](api-remessa-v1.md#endpoint-aggiornare-la-spedizione).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | `PUT` senza `nfkey`, `attachments` né `cost_center_code` | Inviare almeno un campo |
| `400` | `Status da remessa não permite atualização na NFE.` | Spedizione oltre `shipped` | Non è possibile collegare la NF-e |
| `400` | `Erro ao salvar arquivo!` | `base64` non valido | Ricodificare il file |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Allegato superiore a 5 MB | Ridurre il file |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` non valido o di un altro account (`PUT`); AWB/NF-e non individuato | Verificare l'identificatore |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Spedizione cancellata/chiusa | Non è possibile allegare |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Formato non accettato | Convertire il file |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML malformato | Correggere l'XML |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Errore imprevisto nel `PUT` | Ripetere con backoff |
| `400` | `Remessa não encontrada!` | Rimozione di allegato con UUID senza spedizione | Verificare l'UUID |
| `400` | `Esta remessa não foi encontrada!` | Cancellazione con identificatore non valido o di un altro account | Verificare l'identificatore |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | Lo status attuale non consente la cancellazione | Vedere la tabella di cancellazione in [Spedizione (v1)](api-remessa-v1.md#endpoint-cancellare-la-spedizione) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Errore imprevisto nella cancellazione | Ripetere con backoff |

> Nella rimozione degli allegati, gli errori individuali arrivano in `data.failed[]` con `reason` (es.: `Nenhum documento encontrado!`),
> anche con HTTP `200`.

## Errori di tracciamento e documenti

`GET /v1/shipping/.../status` e `GET /v1/shipping/.../docs`.
Dettagli in [Tracciamento della Spedizione](api-rastreio.md#errori-comuni) e [Documenti della Spedizione](api-documentos.md#errori-comuni).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `400` | `remessa.status.status_not_found: ` | Status: nessuna spedizione per l'UUID/AWB indicato (testo senza traduzione) | Usare l'UUID della quotazione o l'AWB corretto |
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Status: spedizione di un altro account o canale senza spedizioni | Verificare l'identificatore/canale |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Spedizione ancora senza registrazione di pagamento | Attendere; se persiste, contattare il supporto |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Errore imprevisto nello status | Ripetere con backoff |
| `400` | `Remessa não encontrada!` | Documenti: UUID senza spedizione | Verificare l'UUID |
| `404` | `Esta remessa não foi encontrada!` | Documenti: AWB/NF-e non individuato | Verificare o usare l'UUID |
| `400` | `Nenhum documento encontrado com o código especificado!` | Documenti non ancora generati, filtro senza risultati o canale senza spedizioni | Attendere `shipped` / rivedere `documents` |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Errore imprevisto nei documenti | Ripetere con backoff |

## Errori di pianificazione del ritiro

`POST /v1/schedule`, `GET /v1/schedule/{shipping}` e `DELETE /v1/schedule/{shipping}/{reason}`.
Dettagli in [Pianificazione del Ritiro](api-agendamento.md#errori-comuni).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Data di oggi o passata | Usare una data a partire da domani |
| `400` | `Coleta não pode ser realizada em um final de semana` | Sabato o domenica | Scegliere un giorno lavorativo |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Festività nazionale | Scegliere un altro giorno lavorativo |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Data troppo lontana | Ridurre l'anticipo |
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Esiste già una pianificazione per la spedizione | Consultare con `GET` |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Spedizione fuori da `shipped` | Attendere `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Corriere senza ritiro integrato | Portare il collo a un punto di spedizione |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Indirizzo di ritiro incompleto | Completare l'indirizzo |
| `400` | `O atributo federal_tax_id não é válido` | CPF/CNPJ non valido | Correggere il documento |
| `400` | `Endereço enviado não pertence ao pais da remessa` | Stato di un altro paese | Usare un indirizzo del paese del mittente |
| `400` | `Agendamento não foi encontrado para esta remessa` | AWB non individuato (`POST`/`DELETE`) | Verificare l'AWB o usare l'UUID |
| `404` | `Agendamento não foi encontrado para esta remessa` | `GET` senza pianificazione | Verificare l'identificatore |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Cancellazione fuori dallo status consentito | Cancellare solo pianificazioni confermate |
| `422` | `O campo reason não contém um valor válido.` (in `data`) | `reason` fuori da `001` a `008` | Usare un motivo valido |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Errore imprevisto (include chiave assente in `address`) | Inviare tutte le chiavi di `address`; ripetere |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Errore imprevisto nella cancellazione | Ripetere; contattare il supporto |

## Errori del manifesto

`POST /v1/manifest`. Dettagli in [Manifesto](api-manifesto.md#errori-comuni).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `400` | `Tipo de pagamento não autorizado!` | Account senza pagamento fatturato | Abilitare la fatturazione |
| `400` | `O canal especificado não foi encontrado!` | `channel_code` di un altro account | Usare un canale dell'account |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` inattivo | Usare un altro contratto |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | Stato non risolto | Inviare `state_code` |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | `address_code` non visibile per l'account | Verificare il codice |
| `404` | `A transportadora informada não está disponível para esta remessa!` | `carrier_code` senza manifesto | Omettere o cambiare il `carrier_code` |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Contratto senza spedizione sincrona | Contattare il supporto |
| `400` | Risposta del corriere (`status: "error"`, con `data`) | Il corriere ha rifiutato la spedizione | Correggere in base alla risposta |
| `400` | `MANIFEST WARNING - AWB cannot be generated, carrier not mapped` | Etichetta non generata dopo l'accettazione; la spedizione esiste | Cercarla in Documenti; **non** ripetere |

> Gli errori della fase di quotazione all'interno del manifesto (scatole, articoli, kit, contratti) tornano come `400 warning`,
> con gli stessi messaggi della [quotazione](#errori-di-quotazione).

## Errori della bozza

`POST /v1/draft`, `PUT /v1/draft/{code}`, `GET /v1/draft` e `DELETE /v1/draft`.
Dettagli in [Bozza](api-rascunho.md#errori-comuni).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | `PUT` su una bozza fuori da `incomplete` | Consultare lo status |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Codice inesistente, di un altro account o già convertito in spedizione | Verificare il codice |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Errore imprevisto | Ripetere con backoff |

> Gli errori dell'elaborazione asincrona della bozza non tornano nella risposta HTTP: compaiono in
> `status_descricao` della bozza (es.: `Nenhum item foi enviado para este rascunho`).

## Errori di scatola e articoli

`/v1/box` e `/v1/item`. Dettagli in [Scatola](api-caixa-v1.md#errori-comuni) e [Articoli](api-itens-v1.md#errori-comuni).

| Status | Messaggio | Causa | Come correggere |
|---|---|---|---|
| `422` | Validazione (in `data`) | Campi della scatola non validi; filtri di elenco non validi | Correggere i campi elencati |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Errore imprevisto | Ripetere con backoff |
| `200` + report | `Box Error: box :id not found!` / `Box Error: box :id cannot be deleted, has related item` | Rimozione della scatola rifiutata durante l'elaborazione | Usare un `box_code` intero / scollegare gli articoli |
| `200` + report | `sku => :sku, status => invalid, reason => :campo` | Articolo scartato nell'elaborazione asincrona | Correggere il campo indicato |

> Le anagrafiche di scatole e articoli sono **asincrone**: la risposta `200` conferma la ricezione, e i record
> non validi vengono scartati in seguito, con un report inviato all'e-mail dell'account.

## Regole di business

### Politica di nuovo tentativo
| Situazione | Ripetere automaticamente? |
|---|---|
| `500` | Sì, con backoff esponenziale (es.: 2 s, 4 s, 8 s) e limite di tentativi |
| `4xx` | No. Correggi prima il payload/scenario |
| Manifesto con `400` dopo l'accettazione del corriere | **No.** La spedizione esiste già: consulta [Tracciamento](api-rastreio.md) e [Documenti](api-documentos.md) |
| Spedizione (`POST /v*/shipping`) con timeout di rete | Prima di ripetere, consulta lo status tramite il `quotation`: una quotazione genera una sola spedizione |

### Rifiuti con `status: "error"`
Alcuni rifiuti di business relativi al **peso** o al **corriere** arrivano con `status: "error"` e codice
`400`/`404`. Gestiscili sempre tramite il codice HTTP e la `message`, non solo tramite lo `status`.

## Note finali
- Convenzioni generali di risposta in [Primi Passi](api-primeiros-passos.md#formato-standard-delle-risposte).
- Quotazioni che non possono diventare spedizioni: [Quotazione (v1)](api-cotacao-v1.md#quotazioni-non-idonee-alla-spedizione).
- In caso di dubbi, comunica al supporto il `quotation`, l'orario della chiamata e la `message` ricevuta.
