# Contratto

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Elenca i contratti di trasporto disponibili per l'utente autenticato.
- Un "contratto" è la combinazione **corriere + servizio + condizioni commerciali** del tuo account.
  È ciò che la quotazione restituisce in `carriers[]`.
- Usa questo endpoint per la diagnostica e per sapere in anticipo quali corrieri possono comparire
  nelle tue quotazioni.

> **Non è necessario** chiamare questo endpoint nel flusso normale di integrazione. La quotazione restituisce già i
> contratti eleggibili per quella spedizione specifica, con prezzo e tempi.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
lang: pt
```

## Quando usarlo

| Situazione | Come aiuta |
|---|---|
| La quotazione restituisce `Nenhum transportador encontrado para este usuário!` | Conferma se l'account ha qualche contratto attivo |
| La quotazione restituisce `Nenhum contrato disponível para este usuário!` | Conferma il collegamento dei contratti |
| Vuoi mostrare all'utente i corrieri abilitati | Elenca nomi e periodi di validità |
| Hai ricevuto `Contrato especificado não foi listado para esta cotação!` | Aiuta a verificare il `code`/`slug` previsto |

## Endpoint: elencare i contratti

- **Metodo e rotta:** `GET /v1/contract`

| Parametro | Posizione | Tipo | Obbligatorio | Predefinito | Valori | Descrizione |
|---|---|---|---|---|---|---|
| `status` | query | string | no | `active` | `active`, `inactive`, `all` | Filtra per validità del contratto. I valori sconosciuti ricadono sul predefinito `active` |
| `page` | query | integer | no | `1` | - | Pagina dell'elenco |

- **Elementi per pagina:** 15

### Esempio di request

```bash
# Contratti attivi (predefinito)
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'

# Tutti i contratti, attivi e inattivi
curl --request GET \
  --url 'https://api.shipsmart.global/v1/contract?status=all' \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'lang: pt'
```

### Esempio di response

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
        "active": "active",
        "direct_quotation": false
      }
    ],
    "first_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "from": 1,
    "last_page": 1,
    "last_page_url": "https://api.shipsmart.global/v1/contract?page=1",
    "next_page_url": null,
    "path": "https://api.shipsmart.global/v1/contract",
    "per_page": 15,
    "prev_page_url": null,
    "to": 5,
    "total": 5
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `code` | integer | **Codice del contratto.** È lo stesso valore che compare in `carriers[].code` nella quotazione e che invii in `carrier_code` nella spedizione |
| `contract_description` | string | Descrizione commerciale del contratto |
| `contract_name` | string | Nome del contratto |
| `country` | string | Paese di origine del contratto (ISO 3166-1 alpha-2) |
| `carrier` | string | Nome del corriere |
| `initial_date` | string | Inizio della validità (`Y-m-d H:i:s`) |
| `final_date` | string | Fine della validità (`Y-m-d H:i:s`) |
| `active` | string | `active` o `inactive` |
| `direct_quotation` | boolean | `true` quando il contratto quota direttamente presso il corriere (in tempo reale), invece di usare una tabella di nolo |

## Relazione con la quotazione e la spedizione

```
GET /v1/contract            → contratti abilitati sull'account (panoramica)
        │
POST /v1/quotation          → carriers[] : contratti eleggibili PER QUELLA SPEDIZIONE
        │                     (con code, slug, prezzo, tempi e valid_until)
        ▼
POST /v1/shipping           → carrier_code (o slug) scelto
```

| Punto di attenzione | Spiegazione |
|---|---|
| La quotazione può restituire **meno** contratti di questo endpoint | Il contratto esiste, ma non copre quel peso/dimensione/rotta |
| La spedizione accetta **solo** i contratti elencati nella quotazione | Inviare un `carrier_code` attivo ma assente nella quotazione restituisce `404 Contrato especificado não foi listado para esta cotação!` |
| Lo `slug` non compare in questo elenco | Usa lo `slug` restituito in `carriers[]` nella quotazione |

## Regole di business
- Il risultato dipende dal **collegamento del tuo utente** ai contratti.
- Se nessun contratto viene trovato con il filtro indicato, la piattaforma effettua una seconda ricerca
  considerando i contratti predefiniti dell'account.
- I contratti `inactive` (fuori validità) **non** compaiono nelle quotazioni.
- `direct_quotation: true` indica una quotazione in tempo reale presso il corriere. Questi contratti possono richiedere
  un po' più di tempo per rispondere e dipendono da credenziali valide
  (`Credenciais inválidas para cotação direta`).

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `403` | Permesso mancante per elencare i contratti | Richiedere il permesso al supporto |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Questo endpoint è di **sola lettura**: i contratti si configurano commercialmente, non via API.
- Usalo come strumento di diagnostica quando la quotazione non restituisce corrieri.
- Per capire perché un contratto specifico non è comparso nella quotazione, vedi
  **Restrizioni di carico** in [Quotazione (v1)](api-cotacao-v1.md#restrizioni-di-carico).
