# Piattaforma

- [Torna al README della lingua](README.md)
- Prerequisito: [Primi Passi](api-primeiros-passos.md)

## Panoramica
- Endpoint di utilità che restituisce il nome, l'ambiente e la versione dell'API Pubblica.
- È la chiamata più semplice ed economica dell'API — ideale per **validare token e connettività** prima di
  qualsiasi integrazione, e per l'**health check** nel monitoraggio.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Staging: `https://hmlapi.shipsmart.com.br`
- Documentazione interattiva: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticazione
```
Authorization: Bearer IL_TUO_TOKEN
```

> Questo endpoint **richiede un token valido**, ma non richiede alcun permesso specifico.
> Per questo è il test migliore per sapere se il tuo token funziona.

## Endpoint: versione della piattaforma

- **Metodo e rotta:** `GET /v1/platform/version`
- **Parametri:** nessuno.

### Esempio di request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer IL_TUO_TOKEN' \
  --header 'Accept: application/json'
```

### Esempio di response

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
| `data.name` | string | Nome dell'applicazione |
| `data.env` | string | Ambiente a cui sei connesso (es.: `production`, `staging`) |
| `data.version` | string | Versione pubblicata dell'API |

> Controlla l'`env` per assicurarti di non puntare all'ambiente sbagliato — è un errore comune
> nelle integrazioni che condividono la configurazione tra gli ambienti.

## Casi d'uso

| Caso | Come usarlo |
|---|---|
| Validare un token nuovo | Se risponde `200`, il token è valido |
| Diagnosticare un `404` di autenticazione | Se anche questo endpoint fallisce, il problema è il token, non la rotta |
| Confermare l'ambiente | Leggere `data.env` |
| Health check di monitoraggio | Chiamata leggera, senza effetti collaterali |
| Registrare la versione nei tuoi log | Utile per correlare il comportamento con le release dell'API |

## Errori comuni

| Status | Significato | Come correggere |
|---|---|---|
| `404` | Token non trovato o non valido | Verificare l'header `Authorization: Bearer ...` e generare un nuovo token nel pannello |
| `403` | Account bloccato | Contattare il supporto |
| `500` | Errore inatteso | Riprovare con backoff; se persiste, contattare il supporto |

## Note finali
- Questo endpoint è di **sola lettura** e non ha alcun effetto su quotazioni, spedizioni o registrazioni.
- Le chiamate senza il prefisso di versione (`/platform/version`) vengono reindirizzate con `308` a
  `/v1/platform/version`. Preferisci sempre indicare la versione.
