# Contratto

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Elenca i contratti di trasporto disponibili per l'utente autenticato, in modo paginato.
- Un "contratto" è la combinazione **corriere + servizio + condizioni commerciali** del tuo account.
  È ciò che la quotazione restituisce in `carriers[]`.
- Usa questo endpoint per la diagnostica e per sapere in anticipo quali corrieri possono comparire
  nelle tue quotazioni.
- Sola lettura: i contratti vengono configurati commercialmente, non tramite API.

> **Non è necessario** chiamare questo endpoint nel flusso normale di integrazione. La quotazione restituisce già i
> contratti idonei per quella spedizione specifica, con prezzo e tempi di consegna.

## URL di base
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
lang: pt
```

> Il tuo profilo deve avere il permesso di elencare i contratti. Senza di esso, la risposta è `403` con `Has no permission!`.

## Flusso di utilizzo

```
1. GET /v1/contract                 → contratti attivi e in vigore (predefinito)
2. GET /v1/contract?status=all      → includere anche quelli disattivati
3. GET /v1/contract?status=all&page=2 → pagine successive (ripeti lo status in ogni pagina)
```

### Quando usarlo

| Situazione | Come aiuta |
|---|---|
| La quotazione restituisce `Nenhum transportador encontrado para este usuário!` | Conferma se l'account ha qualche contratto attivo e in vigore |
| La quotazione restituisce `Nenhum contrato disponível para este usuário!` | Conferma il collegamento dei contratti |
| Vuoi mostrare all'utente i corrieri abilitati | Elenca nomi e periodi di validità |
| Hai ricevuto `Contrato especificado não foi listado para esta cotação!` | Aiuta a verificare il `code` previsto |

## Concetti essenziali

| Concetto | Significato |
|---|---|
| Contratto collegato | Contratto associato direttamente al tuo account. Se il tuo utente è collegato a un account principale, valgono i contratti dell'account principale |
| Contratto predefinito | Contratto predefinito della piattaforma, usato quando l'account non ha contratti collegati |
| Validità | Periodo tra `initial_date` e `final_date`. **Vengono elencati solo i contratti in vigore oggi**, indipendentemente dal filtro `status` |
| Attivo / inattivo | `active` = contratto abilitato; `inactive` = contratto disattivato (ancora entro il periodo di validità). **Non** si riferisce alla validità |

## Endpoint: elencare i contratti

- **Metodo e rotta:** `GET /v1/contract`
- **Quando usarlo:** diagnostica di una quotazione senza corrieri o visualizzazione dei corrieri abilitati.

### Parametri

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Valori | Descrizione |
|---|---|---|---|---|---|---|
| `status` | query | string | no | `active` | `active`, `inactive`, `all` | Filtra per stato del contratto (abilitato/disattivato). I valori vuoti o sconosciuti ricadono sul predefinito `active`, senza errore |
| `page` | query | integer | no | `1` | intero ≥ 1 | Pagina dell'elenco. I valori non validi vengono trattati come `1` |

- **Elementi per pagina:** 15 (fisso; non esiste un parametro per modificarlo).
- **Ordinamento:** non garantito. Non fare affidamento sull'ordine degli elementi.

### Payload minimo

Senza corpo e senza query string:

```
GET /v1/contract
```

### Payload massimo

Tutti i parametri accettati:

```
GET /v1/contract?status=all&page=2
```

### Esempio di request

```bash
# Contratti attivi (predefinito)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'

# Tutti i contratti, attivi e inattivi, pagina 2
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all&page=2' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK`

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "current_page": 1,
    "data": [
      {
        "code": 3,
        "contract_description": "Contratto standard PRO",
        "contract_name": "Standard Default Pro",
        "country": "BR",
        "carrier": "DHL",
        "initial_date": "2026-01-06 00:00:00",
        "final_date": "2026-12-31 23:59:59",
        "direct_quotation": false,
        "active": "active"
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "links": [
      { "url": null, "label": "&laquo; Anterior", "active": false },
      { "url": "https://api.shipsmart.global/v1/contract?page=1", "label": "1", "active": true },
      { "url": null, "label": "Próximo &raquo;", "active": false }
    ],
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 1,
    "total": 1
  }
}
```

### Campi della risposta

Elementi di `data.data[]`:

| Campo | Tipo | Descrizione |
|---|---|---|
| `code` | integer | **Codice del contratto.** È lo stesso valore che compare in `carriers[].code` nella quotazione e che invii in `carrier_code` nella spedizione |
| `contract_description` | string \| null | Descrizione commerciale del contratto |
| `contract_name` | string | Nome del contratto |
| `country` | string | Paese di origine del contratto (ISO 3166-1 alpha-2) |
| `carrier` | string \| null | Nome del servizio del corriere |
| `initial_date` | string | Inizio della validità (`AAAA-MM-GG HH:MM:SS`) |
| `final_date` | string | Fine della validità (`AAAA-MM-GG HH:MM:SS`) |
| `direct_quotation` | boolean | `true` quando il contratto quota direttamente presso il corriere (in tempo reale), invece di usare una tabella di nolo |
| `active` | string | `active` (abilitato) o `inactive` (disattivato) |

Paginazione (in `data`):

| Campo | Tipo | Descrizione |
|---|---|---|
| `current_page` | integer | Pagina corrente |
| `from` / `to` | integer \| null | Posizione del primo e dell'ultimo elemento della pagina (`null` quando la pagina è vuota) |
| `total` | integer | Totale dei contratti trovati |
| `per_page` | integer | Elementi per pagina (`15`) |
| `last_page` | integer | Numero dell'ultima pagina |
| `first_page_url` / `last_page_url` | string | URL della prima e dell'ultima pagina |
| `prev_page_url` / `next_page_url` | string \| null | URL della pagina precedente e di quella successiva (`null` quando non esistono) |
| `path` | string | URL di base dell'elenco |
| `links` | array | Link di navigazione (`url`, `label`, `active`). Il `label` viene tradotto in base all'header `lang` |

> ⚠️ Gli URL di paginazione **non conservano il filtro `status`**. Navigando con `status=inactive` o
> `status=all`, costruisci tu stesso l'URL della pagina successiva (`?status=all&page=N`) invece di seguire
> direttamente `next_page_url`.

## Relazione con la quotazione e la spedizione

```
GET /v1/contract            → contratti abilitati nell'account (visione generale)
        │
POST /v1/quotation          → carriers[] : contratti idonei PER QUELLA SPEDIZIONE
        │                     (con code, slug, prezzo, tempi e valid_until)
        ▼
POST /v1/shipping           → carrier_code (o slug) scelto
```

| Punto di attenzione | Spiegazione |
|---|---|
| La quotazione può restituire **meno** contratti di questo endpoint | La quotazione considera solo i contratti attivi del paese di origine della spedizione, con tabella di nolo per il tipo di oggetto (documento o non documento) e che soddisfano quel peso/dimensione/rotta |
| La spedizione accetta **solo** contratti elencati nella quotazione | Inviare un `carrier_code` attivo ma assente nella quotazione restituisce `404 Contrato especificado não foi listado para esta cotação!` |
| Lo `slug` non compare in questo elenco | Usa lo `slug` restituito in `carriers[]` nella quotazione |

## Regole di business

| Regola | Dettaglio |
|---|---|
| Collegamento | Il risultato dipende dai contratti collegati al tuo account (o all'account principale, per gli utenti collegati) |
| Contratti predefiniti | Se la ricerca non trova alcun contratto collegato **nella pagina richiesta**, la piattaforma restituisce i contratti predefiniti della piattaforma con lo stesso filtro. Per questo, richiedere una pagina oltre l'ultima può restituire contratti predefiniti |
| Validità | I contratti fuori validità (prima di `initial_date` o dopo `final_date`) non vengono mai elencati |
| Filtro `status` | `active` (predefinito), `inactive` o `all`. Qualsiasi altro valore viene trattato come `active` |
| Quotazione | I contratti `inactive` (disattivati) **non** compaiono nelle quotazioni |
| Quotazione diretta | `direct_quotation: true` indica una quotazione in tempo reale presso il corriere. Questi contratti possono impiegare un po' più di tempo a rispondere e dipendono da credenziali valide (`Credenciais inválidas para cotação direta`) |

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `403` | `Has no permission!` | Richiedere al supporto il permesso di elencare i contratti |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Contattare il supporto |
| `404` | `Nenhum usuário encontrado para o token especificado!` | Verificare l'header `Authorization: Bearer ...` |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Ripetere con backoff; se persiste, contattare il supporto |

> I filtri non validi **non** generano errori: uno `status` sconosciuto diventa `active` e una `page` non valida diventa `1`.
> Un elenco vuoto (`data.data: []`, `total: 0`) significa che non ci sono contratti in vigore né contratti predefiniti
> per quel filtro.

## Note finali
- Questo endpoint è **di sola lettura**: i contratti vengono configurati commercialmente, non tramite API.
- Usalo come strumento di diagnostica quando la quotazione non restituisce corrieri.
- Per capire perché un contratto specifico non è comparso nella quotazione, consulta
  **Restrizioni di carico** in [Quotazione (v1)](api-cotacao-v1.md#restrizioni-di-carico).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
