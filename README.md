# PriceCreator — Cartellini prezzo Doctor Phone

App desktop (Tauri 2 + React + TypeScript) per comporre cartellini prezzo e
generare un PDF A4 pronto da stampare: griglia 4×2 (8 cartellini per foglio,
oltre l'ottavo si passa a una nuova pagina), cornice blu, prezzi in evidenza e
ribbon rosso "OFFERTA" per i prodotti scontati.

## Immagini del brand

Copia le due immagini in `public/tag-assets/` (PNG o JPEG, l'estensione può
restare `.png`; per il robot è consigliato un PNG con sfondo trasparente):

- `public/tag-assets/robot.png` — la mascotte robot (centrata in basso nel cartellino)
- `public/tag-assets/logo.png` — il logo Doctor Phone (in basso a sinistra)

Se mancano, il PDF viene generato comunque senza immagini (nessun errore).
Non serve riavviare l'app: i PNG vengono riletti a ogni generazione.

## Piattaforme

- **macOS** — bundle universal (Intel x86_64 + Apple Silicon aarch64)
- **Windows** — installer NSIS (`.exe`)

I build di CI **sono firmati in modo gratuito**:

- **macOS** — firma *ad-hoc* (`signingIdentity: "-"`). Su Apple Silicon il file
  scaricato da GitHub non risulta «danneggiato». Gatekeeper continua a chiedere
  «Apri comunque» (tasto destro → Apri): togliere del tutto l'avviso richiede
  l'Apple Developer Program (**99 €/anno**, non esiste alternativa gratis).
- **Windows** — certificato Authenticode **self-signed**. L'installer risulta
  firmato; SmartScreen può ancora avvisare finché il publisher non ha
  reputazione (un certificato CA a pagamento lo evita subito).

Per un publisher Windows **stabile** (stesso certificato a ogni release):

```bash
bash scripts/generate-windows-cert.sh
```

Poi in GitHub → Settings → Secrets:

| Secret | Contenuto |
| --- | --- |
| `WINDOWS_CERTIFICATE` | file `.tauri/windows-codesign.pfx.b64` |
| `WINDOWS_CERTIFICATE_PASSWORD` | file `.tauri/windows-codesign.password` |

Se i secret non ci sono, il job Windows genera un certificato efimero (sempre
firmato, ma identità diversa a ogni build).

## Comandi

```bash
npm install          # dipendenze
npm run tauri dev    # avvia l'app in sviluppo
npm run tauri build  # crea il bundle distribuibile (macOS/Windows)
npm run render:sample  # genera out/sample.pdf senza GUI (test del layout)
```

## Release (GitHub)

La versione unica è il **tag git**. Il workflow `.github/workflows/release.yml`
sincronizza `package.json`, `src-tauri/tauri.conf.json` e `src-tauri/Cargo.toml`
dal tag, poi `tauri-action` pubblica una GitHub Release con installer e
`latest.json` / `.sig` per l'updater.

```bash
git tag v0.2.0
git push origin v0.2.0
```

Si può anche lanciare il workflow a mano (`workflow_dispatch`): in quel caso
resta la versione già scritta in `tauri.conf.json` (oggi `0.1.0`).

### Endpoint updater

`https://github.com/sebastianoboem/PriceCreator/releases/latest/download/latest.json`

### Secret GitHub da impostare

Repository → Settings → Secrets and variables → Actions:

| Secret | Obbligatorio | Contenuto |
| --- | --- | --- |
| `TAURI_SIGNING_PRIVATE_KEY` | sì | contenuto intero di `.tauri/updater.key` (generato in locale, **mai** committare) |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | solo se la chiave ha password | questa chiave è stata creata **senza** password: lascia il secret non impostato |
| `WINDOWS_CERTIFICATE` | consigliato | Base64 del `.pfx` (output di `scripts/generate-windows-cert.sh`) |
| `WINDOWS_CERTIFICATE_PASSWORD` | con il certificato | password del `.pfx` |

Firma/notarizzazione Apple (opzionale, a pagamento): `APPLE_CERTIFICATE`,
`APPLE_CERTIFICATE_PASSWORD`, `APPLE_SIGNING_IDENTITY`, `APPLE_ID`,
`APPLE_PASSWORD`, `APPLE_TEAM_ID`. Senza di essi si usa la firma ad-hoc.

### Updater in-app

All'avvio l'app chiama il plugin updater. Se GitHub ha una release più nuova
mostra: «È disponibile la versione X. Vuoi aggiornare?» con **Aggiorna** /
**Più tardi**. Aggiorna scarica, installa e riavvia. Se il check fallisce
(offline, repo assente) l'errore è silenzioso e
l'UI resta usabile.

## Struttura

- `src/pdf/generatePdf.ts` — motore PDF puro (pdf-lib + fontkit), eseguibile anche sotto Node
- `src/pdf/resources.ts` — caricamento font/immagini lato app (con validazione PNG)
- `src/App.tsx`, `src/components/TagCard.tsx` — interfaccia (in italiano)
- `src/assets/fonts/` — Barlow Regular/Bold (testi e prezzi) e Glacial
  Indifference Regular (riga NUOVO/RICONDIZIONATO), entrambi con licenza OFL
- `scripts/render-sample.ts` — render di prova con i 5 cartellini di esempio

## Note

- Prezzi in stile italiano: interi senza decimali ("189€"), altrimenti con la
  virgola ("189,50€").
- Il campo "Prezzo originale" compare solo con "In offerta" attivo; se lasciato
  vuoto il prezzo barrato non viene disegnato (restano ribbon e prezzo rosso).
