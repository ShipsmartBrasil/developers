# Scatola

- [Torna al README della lingua](README.md)

## Panoramica
- Gestisce la registrazione e la consultazione delle scatole usate in quotazioni, bozze e spedizioni.

## Convenzioni
- `Content-Type: application/json`
- La consulta usa `boxes[box_ids][]`.

## Flusso d'uso
1. Elenca le scatole disponibili.
2. Registra o aggiorna le scatole base.
3. Rimuovi le scatole quando necessario.
4. Riutilizza i codici nelle quotazioni e bozze.

## Endpoint
- `GET /v1/box`
- `POST /v1/box`
- `DELETE /v1/box`

## Endpoint
- `GET /v1/box`
- `POST /v1/box`
- `DELETE /v1/box`
