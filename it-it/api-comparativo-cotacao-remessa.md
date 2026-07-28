# Confronto Quotazione e Spedizione

- [Torna al README della lingua](README.md)

## Panoramica
- Spiega la differenza pratica tra i flussi di quotazione e spedizione nelle due versioni pubblicate.
- Usa questa guida per scegliere la versione corretta prima di integrare.

## Base URL
- Produzione: `https://api.shipsmart.global`
- Documentazione: `https://api.shipsmart.global/docs`
- Omologazione: `https://hmlapi.shipsmart.com.br`

## Autenticazione
- Bearer token nell'header `Authorization`.
- Header della lingua: `lang`.

## Regola di utilizzo
- Il flusso standard di integrazione è: quotazione e spedizione usando gli endpoint della prima versione.
- La seconda versione è indicata quando non vuoi informare dati oltre al paese nella quotazione.

## Quotazione
### Differenza pratica
- Nella prima versione, la quotazione accetta il flusso completo con origine, destinazione, scatole e articoli.
- Nella seconda versione, la quotazione è più snella per scenari in cui l'integratore vuole informare meno dati all'inizio del processo.

### Confronto
| Tema | Prima versione | Seconda versione | Impatto |
|---|---|---|---|
| Obiettivo | Quotazione completa | Quotazione semplificata | Scegli il livello di dettaglio |
| Indirizzo | Più campi nel contratto | Contratto più snello | Meno dati in ingresso |
| Articoli | Flusso completo e avanzato | Flusso esplicito per articoli | Migliore per payload snelli |
| Consultazione | Consultazione e status disponibili nell'endpoint della prima versione | Informazione non identificata nella fonte analizzata | Entrambe le consultazioni operative restano nella prima versione |

## Spedizione
### Differenza pratica
- Nella prima versione, la spedizione copre creazione, consultazione dello status, consultazione dei documenti, annullamento, aggiornamento e allegati.
- Nella seconda versione, la fonte analizzata mostra solo la creazione della spedizione.

### Confronto
| Tema | Prima versione | Seconda versione | Impatto |
|---|---|---|---|
| Creazione | Disponibile | Disponibile | Flusso base comune |
| Status | Disponibile | Informazione non identificata nella fonte analizzata | Il monitoraggio operativo dipende dalla prima versione |
| Documenti | Disponibile | Informazione non identificata nella fonte analizzata | Il download dei documenti resta nella prima versione |
| Annullamento | Disponibile | Informazione non identificata nella fonte analizzata | Il ciclo completo resta nella prima versione |
| Aggiornamento | Disponibile solo in casi eccezionali | Informazione non identificata nella fonte analizzata | Di norma, la spedizione è irreversibile dopo la creazione |

## Quando usare ciascuna
1. Usa la prima versione quando vuoi il flusso operativo completo.
2. Usa la seconda versione quando il payload iniziale di quotazione deve essere più semplice.
3. Se il tuo flusso dipende da status, documenti o annullamento, mantieni la prima versione.
4. Se crei tramite la seconda versione, il monitoraggio delle azioni viene fatto tramite le API della prima versione.

## Osservazioni finali
- La fonte analizzata non porta una documentazione equivalente completa della seconda versione per status, docs e annullamento.
- Per l'operatività completa, la prima versione continua a essere il riferimento funzionale.
