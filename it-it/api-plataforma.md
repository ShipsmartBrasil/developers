# Piattaforma

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

> ⚠️ **Esempi puramente illustrativi:** tutti i payload, le risposte, gli identificatori, i codici, i documenti e i token
> di questa pagina sono fittizi e **non sono dati validi**. Sostituiscili con i dati reali del tuo account e della tua operatività.

## Panoramica
- Endpoint di utilità che restituisce il nome, l'ambiente e la versione dell'API Pubblica.
- È la chiamata più semplice ed economica dell'API — ideale per **validare token e connettività** prima di
  qualsiasi integrazione, e per l'**health check** nel monitoraggio.
- Richiede solo un token valido: **non dipende da alcun permesso** del profilo.
- È di sola lettura e non ha effetti collaterali.
- Questo endpoint non compare nella documentazione interattiva (`/doc`); questa pagina ne è il riferimento.

## URL di base
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
lang: pt
```

> Questo endpoint **richiede un token valido**, ma non richiede alcun permesso specifico.
> Per questo è il test migliore per sapere se il tuo token funziona.
> L'header `lang` (`pt`, `en`, `es`) modifica solo il testo di `message`.

## Flusso di utilizzo

```
1. GET /v1/platform/version   → 200: token valido e API raggiungibile
2. Verificare data.env        → ambiente atteso (produzione o staging)
3. Proseguire con l'integrazione → quotazione, spedizione ecc.
```

## Endpoint: versione della piattaforma

- **Metodo e rotta:** `GET /v1/platform/version`
- **Quando usarlo:** quando configuri un token nuovo, quando cambi ambiente e nelle verifiche periodiche di salute.

### Parametri

Nessun parametro di path, query o corpo. Solo gli header di [Autenticazione](#autenticazione).

### Payload minimo

Senza corpo:

```
GET /v1/platform/version
```

### Payload massimo

Non ci sono parametri opzionali. L'unica variazione possibile è la lingua del messaggio, tramite header:

```
GET /v1/platform/version
lang: en
```

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Accept: application/json' \
  --header 'lang: pt'
```

### Esempio di response

`200 OK`

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "production",
    "version": "1.0.0"
  }
}
```

### Campi della risposta

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | string | Sempre `success` quando il token viene accettato |
| `message` | string | `Registro(s) listado(s) com sucesso.` (tradotto in base all'header `lang`) |
| `data.name` | string \| null | Nome dell'applicazione |
| `data.env` | string \| null | Ambiente a cui sei connesso (es.: `production`, `staging`) |
| `data.version` | string \| null | Versione pubblicata dell'API |

> Controlla l'`env` per assicurarti di non puntare all'ambiente sbagliato — è un errore comune
> nelle integrazioni che condividono la configurazione tra gli ambienti.
>
> I valori di `data` sono informativi e definiti dalla configurazione di ciascun ambiente. Non usarli come
> chiave di logica (es.: confronto esatto della versione) e considera `null` un valore possibile.

## Casi d'uso

| Caso | Come usarlo |
|---|---|
| Validare un token nuovo | Se risponde `200`, il token è valido |
| Diagnosticare un `404` di autenticazione | Se anche questo endpoint risponde `404`, il problema è il token, non la rotta |
| Diagnosticare un `403 Has no permission!` in un'altra rotta | Se questo endpoint risponde `200`, il token è valido e manca solo il permesso per quell'operazione |
| Confermare l'ambiente | Leggere `data.env` |
| Health check di monitoraggio | Chiamata leggera, senza effetti collaterali |
| Registrare la versione nei tuoi log | Utile per correlare il comportamento con le release dell'API |

## Regole di business

| Regola | Dettaglio |
|---|---|
| Autenticazione | Il token viene validato prima di qualsiasi risposta. Senza l'header `Authorization`, la risposta è `404` |
| Permesso | Nessuno. Qualsiasi token valido di un utente attivo riceve `200` |
| Account bloccato | Il token di un utente bloccato riceve `403` |
| Versione nel path | Le chiamate senza il prefisso (`/platform/version`) ricevono `308` verso `/v1/platform/version`, preservando la query string |
| Sola lettura | Non modifica quotazioni, spedizioni né registrazioni |

## Errori comuni

| Status | Significato/Messaggio | Come correggere |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Token assente, errato o rigenerato. Verificare l'header `Authorization: Bearer ...` e, se necessario, generare un nuovo token nel pannello |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Account bloccato. Contattare il supporto |
| `400` | `status: "error"` con la descrizione dell'errore di autenticazione | Errore inatteso durante la validazione del token. Riprovare con backoff; se persiste, contattare il supporto |
| `308` | Reindirizzamento permanente | Chiamata senza `/v1`. Usare la rotta completa `/v1/platform/version` |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Questo endpoint è di **sola lettura** e non ha alcun effetto su quotazioni, spedizioni o registrazioni.
- Le chiamate senza il prefisso di versione (`/platform/version`) vengono reindirizzate con `308` a
  `/v1/platform/version`. Preferisci sempre indicare la versione.
- Dettagli su token, lingua e formato delle risposte: [Primi Passi](api-primeiros-passos.md).
- Elenco consolidato dei messaggi di errore: [Codici di Errore](api-codigos-de-erro.md).
