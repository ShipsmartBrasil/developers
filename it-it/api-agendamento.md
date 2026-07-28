# Programmazione del Ritiro

- [Torna al README della lingua](README.md)

## Panoramica
- Crea, consulta e annulla programmazioni di ritiro collegate alla spedizione.
- Il ritiro può essere effettuato tramite quotazione/spedizione o tramite AWB.
- Associato al flusso operativo principale.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Convenzioni
- Date nel formato `Y-m-d`.
- Orari nel formato `H:i:s`.
- La programmazione deve rispettare il paese del mittente.

## Flusso di utilizzo
1. Assicurati che la spedizione sia pronta per la programmazione.
2. Crea il ritiro con finestra oraria e istruzione.
3. Consulta lo stato della programmazione.
4. Se necessario, annulla usando uno dei motivi consentiti.

## Endpoints
### Creare programmazione
- Metodo e rotta: `POST /v1/schedule`
- Regole importanti: informa `shipping` o `awb_number`.

### Consultare programmazione
- Metodo e rotta: `GET /v1/schedule/{shipping}`

### Annullare programmazione
- Metodo e rotta: `DELETE /v1/schedule/{shipping}/{reason}`

### Parametri
| Nome | Posizione | Tipo | Obbligatorio | Descrizione | Esempio | Osservazioni |
|---|---|---:|---:|---|---|---|
| `shipping` | body/path | string | condizionale | UUID della quotazione/spedizione | `9c06fd75-d5f4-40eb-a1a8-f12086f49944` | Obbligatorio se `awb_number` non viene inviato |
| `awb_number` | body | string | condizionale | AWB | `271249639538` | Obbligatorio se `shipping` non viene inviato |
| `date` | body | date | sì | Data del ritiro | `2023-11-21` | Formato `Y-m-d` |
| `initial_time` | body | time | sì | Inizio della finestra | `08:00:00` | Formato `H:i:s` |
| `final_time` | body | time | sì | Fine della finestra | `18:00:00` | Formato `H:i:s` |
| `instruction` | body | string | sì | Istruzione per il ritiro | `Deixar na portaria` | Da 5 a 80 caratteri |
| `address` | body | object | no | Indirizzo alternativo | - | Deve essere dello stesso paese del mittente |
| `reason` | path | string | sì | Motivo dell'annullamento | `001` | Codice fisso |

### Motivi di annullamento
| Codice | Significato |
|---|---|
| `001` | Package not ready |
| `002` | Rates too high |
| `003` | Transit time too slow |
| `004` | Take to service center or drop box |
| `005` | Commitment time not met |
| `006` | Reason not given |
| `007` | Other |
| `008` | Pickup modified |

### Esempio di request
```bash
curl --request POST \
  --url https://api.shipsmart.global/v1/schedule \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Content-Type: application/json' \
  --header 'lang: pt' \
  --data '{
    "shipping": "9c06fd75-d5f4-40eb-a1a8-f12086f49944",
    "date": "2023-11-21",
    "initial_time": "08:00:00",
    "final_time": "18:00:00",
    "instruction": "Deixar na portaria"
  }'
```

### Esempio di response
```json
{
  "status": "success",
  "message": "Agendamento de coleta registrado com sucesso"
}
```

## Regole di business
- È possibile programmare solo se la spedizione è pronta per questo.
- Potrebbe già esserci una programmazione in corso, e in questo caso l'API restituisce warning.
- L'indirizzo alternativo deve rispettare lo stesso paese del mittente.

## Errori comuni
| Status | Significato | Come correggere |
|---|---|---|
| `400` | Programmazione già esistente o spedizione non pronta | Adeguare il flusso | 
| `422` | Validazione | Correggere data, ora o campi obbligatori |
| `500` | Errore imprevisto | Contattare il team responsabile |

## Osservazioni finali
- Consulta lo stato prima di annullare per evitare richieste ridondanti.
