# Bozza

- [Torna al README della lingua](README.md)

## Panoramica
- Crea, elenca, aggiorna ed elimina bozze di spedizione.

## Convenzioni
- La bozza accetta payload parziali.
- Lo stato può essere `received`, `edited`, `validating`, `incomplete`, `complete`, `processing` o `error`.

## Flusso d'uso
1. Invia la bozza con i dati che hai già.
2. Attendi l'elaborazione asincrona.
3. Elenca o consulta le bozze.
4. Aggiorna solo le bozze `incomplete`.
5. Rimuovi le bozze quando necessario.

## Endpoint
- `POST /v1/draft`
- `PUT /v1/draft/{code}`
- `GET /v1/draft`
- `DELETE /v1/draft`
