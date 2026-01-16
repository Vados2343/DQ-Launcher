# Panoramica Architettura

## Design del Sistema

DQ Launcher è un'applicazione full-stack a tre livelli progettata per distribuzione enterprise di giochi con fingerprinting dispositivi e riconoscimento hardware.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Desktop                           │
│                    (WPF .NET 8 / Windows)                       │
│  • Autenticazione via Telegram OAuth                           │
│  • Fingerprinting hardware (SHA256 scoring pesato)             │
│  • Gestione modpack (installa/switch/aggiorna)                 │
│  • Orchestrazione download multi-server                        │
│  • Auto-updater con supporto rollback                          │
│  • Rilevamento e configurazione VPN                            │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS + JWT
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Server Backend API                         │
│                (Node.js/Express/TypeScript)                     │
│  • Integrazione OAuth Telegram (webhook)                       │
│  • Algoritmo riconoscimento dispositivi (scoring pesato)       │
│  • Matching fingerprint hardware & tracking migrazioni         │
│  • Gestione distribuzione file multi-server                    │
│  • API admin panel (user/device/modpack)                       │
│  • Progresso upload real-time (WebSocket)                      │
│  • Audit logging & rilevamento attività sospette               │
│  • Rate limiting & middleware sicurezza                        │
└──────────────────────────────┬──────────────────────────────────┘
                               │ PostgreSQL
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Database (PostgreSQL)                         │
│  • Users, Sessions, Audit Logs                                 │
│  • DeviceFingerprints (componenti hardware hashati)            │
│  • UserDevices (many-to-many + metadata)                       │
│  • DeviceMigrations (cronologia cambio + confidence)           │
│  • SharedHardwareComponents (rilevamento frodi)                │
│  • Modpacks, News, Error Reports                               │
└─────────────────────────────────────────────────────────────────┘
                               ▲
                               │ HTTPS
┌──────────────────────────────┴──────────────────────────────────┐
│                    Pannello Admin Frontend                      │
│                  (React 18 / Vite / TypeScript)                 │
│  • Gestione utenti & tracking dispositivi                      │
│  • Monitoraggio real-time (dashboard)                          │
│  • Upload & gestione modpack                                   │
│  • Allerte sicurezza & blocco dispositivi                      │
│  • Viewer audit log con filtri                                 │
│  • Analisi report errori                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Client Desktop (.NET 8 / WPF)

### Dependency Injection

Servizi registrati in `App.xaml.cs`:

```
IServiceCollection
├── Singleton
│   ├── ISettingsService
│   ├── IHwidService
│   └── IHardwareFingerprintService
│
└── Transient
    ├── ILauncherApiService
    ├── IDownloadService
    ├── IModManagerService
    ├── IGameService
    └── MainViewModel
```

### Pattern Design

**MVVM con Classi Partial**
- MainViewModel diviso per responsabilità (Auth, Downloads, Games, Mods, Settings, Connection, Update)
- Ogni partial 150-380 righe, focalizzato
- Istanza singola ViewModel guida tutta l'UI
- ObservableProperty (CommunityToolkit.Mvvm) per binding proprietà

**Dependency Injection**

MainViewModel usa classi partial per separare le responsabilità:

| File | Responsabilità | Righe |
|------|----------------|-------|
| MainViewModel.cs | Stato core, inizializzazione | ~460 |
| MainViewModel.Auth.cs | Flusso login, gestione token | ~380 |
| MainViewModel.Downloads.cs | Coda download, progresso | ~300 |
| MainViewModel.Games.cs | Stato gioco, logica avvio | ~250 |
| MainViewModel.Mods.cs | Gestione modpack | ~200 |
| MainViewModel.UI.cs | Tema, lingua, stato UI | ~150 |

### Layer Servizi

**HardwareFingerprintService**
Raccoglie identificatori hardware via WMI:
- Presenza e versione TPM
- System UUID (SMBIOS)
- CPU ID
- Indirizzi MAC (tutte le NIC)
- Numeri seriali dischi
- ID dispositivi GPU

Tutti i valori hashati con SHA256 prima della trasmissione.

**ManifestDownloadService**
1. Fetch manifest dal server (lista file con checksum)
2. Confronto con file locali
3. Download file mancanti/modificati (4 concorrenti)
4. Verifica ogni file: controllo dimensione → hash SHA256
5. Su mismatch: elimina e riscarica (no resume)

**LauncherApiService**
- `InitiateAuthAsync()` — Avvia flusso OAuth Telegram
- `CheckAuthStatusAsync()` — Poll per completamento
- `AutoSelectBestServerAsync()` — Ping server, seleziona latenza minore
- `GetManifestAsync()` — Fetch manifest file
- `ReportErrorAsync()` — Invia errori client al server

### Flusso Dati

```
Utente clicca Login
    │
    ▼
MainViewModel.LoginCommand
    │
    ▼
LauncherApiService.InitiateAuthAsync()
    ├── Raccolta HWID (legacy MD5)
    ├── Raccolta DeviceFingerprint (SHA256)
    └── POST /launcher/initiate
            │
            ▼
        Server riconosce dispositivo
        Ritorna: authToken, sessionId
            │
            ▼
    Poll /launcher/status fino a conferma
            │
            ▼
    Salva token in SettingsService
    Aggiorna IsAuthenticated = true
```

## Backend (Node.js/Express/TypeScript)

### Flusso Autenticazione

```
1. LAUNCHER avvia auth
   POST /launcher/initiate
   ├── Invia HWID (legacy MD5)
   ├── Invia DeviceFingerprint (SHA256 hash tutti componenti)
   └── Riceve authCode + deep link Telegram

2. UTENTE approva in Telegram
   Tocca t.me/dq_bot?start={authCode}
   └── Bot memorizza approvazione in Redis (600s TTL)

3. LAUNCHER controlla status
   GET /launcher/status/{authCode}
   ├── Poll ogni 2 secondi
   ├── Server controlla Redis per approvazione
   └── Ritorna JWT token + downloadToken su successo

4. LAUNCHER memorizza token
   localStorage/settings.json
   └── Usa token per richieste API successive
```

### Algoritmo Riconoscimento Dispositivi

```
Scoring Pesato (max 100):
┌──────────────────────────────────────────┐
│ Componente      │ Peso │ Affidabilità  │
├─────────────────┼──────┼───────────────┤
│ TPM             │ 40%  │ Massima       │
│ System UUID     │ 25%  │ Molto Alta    │
│ MAC Address     │ 15%  │ Media*        │
│ Disk Serial     │ 10%  │ Media         │
│ CPU ID          │ 5%   │ Bassa**       │
│ GPU ID          │ 5%   │ Bassa         │
└──────────────────────────────────────────┘

*MAC: Può essere virtualizzato/spoofato facilmente
**CPU: Cambia con reset BIOS

Logica Decisione:
• Score >= 70% → Dispositivo riconosciuto
  └─ Aggiorna record dispositivo
• 50% ≤ Score < 70% → Possibile migrazione
  ├─ Controlla limite migrazione (2/anno)
  ├─ Registra migrazione con confidence
  └─ Segnala per review manuale se necessario
• Score < 50% → Nuovo dispositivo
  └─ Crea nuovo record DeviceFingerprint
```

### Modello Sicurezza

```
Threat Model:
┌─────────────────────────────────────┐
│ Rilevamento Account Sharing         │
├─────────────────────────────────────┤
│ Utenti multipli con stesso:         │
│ • Indirizzo MAC (macchine virtuali) │
│ • Serial disco (unità condivise)    │
│ • TPM (stesso hardware)             │
│ → Allerta admin, segnala sospetto   │
└─────────────────────────────────────┘

Policy Dispositivi:
• Max 5 dispositivi per utente
• Max 2 migrazioni per 365 giorni
• Auth fallite loggete con IP
• Rapidi cambiamenti dispositivo → allerta
• Cambio IP da 100+ paesi = sospetto

Protezione Dati:
• Tutti gli ID hardware hashati lato client prima trasmissione
• Solo hash memorizzati in database
• JWT token: durata 24h
• Token download: durata 7d
• Rate limiting: 5 tentativi auth per 15 min per IP
• Azioni admin loggete con timestamp + IP
```

### Struttura Route

```
/launcher
├── POST /initiate      # Avvia auth, riceve HWID + fingerprint
├── GET  /status/:id    # Poll stato auth
├── GET  /manifest      # Lista file con checksum
└── GET  /download/:file # Download file autenticato

/admin
├── GET  /users         # Lista utenti (paginata)
├── GET  /users/:id     # Dettagli utente + dispositivi
├── GET  /users/:id/devices  # Lista dispositivi
├── POST /users/:id/devices/:did/block
└── GET  /checksum-errors    # Errori riportati dai client

/security
├── GET  /logs          # Audit trail
└── GET  /suspicious    # Attività segnalate
```

### Algoritmo Riconoscimento Dispositivi

```
Input: DeviceFingerprint { tpmHash, uuidHash, cpuIdHash, macHashes[], diskHashes[], gpuHashes[] }

1. Query fingerprint esistenti con componenti corrispondenti:
   - Match esatto su tpmHash
   - Match esatto su uuidHash
   - Intersezione su macHashes
   - Intersezione su diskHashes

2. Per ogni candidato, calcola score pesato:
   Match TPM:     +40 punti
   Match UUID:    +25 punti
   Match MAC:     +15 punti (qualsiasi)
   Match Disk:    +10 punti (qualsiasi)
   Match CPU:     +5 punti
   Match GPU:     +5 punti
   ─────────────────────────
   Max:           100 punti

3. Se score >= 70: dispositivo riconosciuto
   Se 50 <= score < 70: possibile migrazione, richiede conferma
   Se score < 50: nuovo dispositivo

4. Traccia migrazioni per utente (limite: 2/anno)
```

### Schema Database

Tabelle principali:

**DeviceFingerprint**
- Memorizza hash SHA256 dei componenti hardware
- Campi array per componenti multi-valore (MAC, dischi)
- Indicizzati per lookup veloce

**UserDevice**
- Many-to-many: User ↔ DeviceFingerprint
- Stato: ACTIVE | SUSPENDED | BLOCKED
- Livello fiducia: UNVERIFIED | VERIFIED | TRUSTED
- Contatore migrazioni

**DeviceMigration**
- Audit trail dei cambiamenti dispositivo
- Confidence score al momento della migrazione
- Componenti cambiati vs corrispondenti

**SharedHardwareComponent**
- Segnala quando stesso componente appare per più utenti
- Usato per rilevamento frodi

## Considerazioni Sicurezza

1. **Nessun seriale raw trasmesso** — Tutti gli ID hardware hashati lato client
2. **Rate limiting** — Limiti per-IP e per-utente sugli endpoint auth
3. **Token JWT** — Token accesso breve durata, refresh via Telegram
4. **Audit logging** — Tutte le azioni admin loggate con IP e timestamp
5. **Limiti dispositivi** — Utenti possono avere max 3 dispositivi attivi
6. **Limiti migrazione** — Max 2 migrazioni dispositivo per anno

## Architettura Deployment

```
┌─────────────────────────────────────────────────────────────┐
│            Ambiente Produzione                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌──────────────────┐  ┌──────────────────┐                │
│ │ Bot Telegram     │  │ Frontend (React) │                │
│ │ Webhooks         │  │ Vite SPA         │                │
│ │ (tg.me/dq_bot)   │  │ TailwindCSS      │                │
│ └────────┬─────────┘  └────────┬─────────┘                │
│          │                     │                          │
│          └─────────────────────┼──────────────────────────│
│                                ▼                          │
│                  ┌──────────────────────────┐             │
│                  │  Express Backend         │             │
│                  │  • /launcher/* route     │             │
│                  │  • /admin/* route        │             │
│                  │  • /download/* route     │             │
│                  │  • WebSocket (upload)    │             │
│                  │  • JWT middleware        │             │
│                  │  • Rate limiting         │             │
│                  └────────┬──────────────────┘            │
│                           │                              │
│  ┌────────────────────────┼────────────────────────┐    │
│  │                        ▼                        │    │
│  │     ┌──────────────────────────────────┐      │    │
│  │     │  Database PostgreSQL             │      │    │
│  │     │  • Users, Devices, Sessions      │      │    │
│  │     │  • DeviceFingerprints (indexed)  │      │    │
│  │     │  • Audit Logs (retention: 90d)  │      │    │
│  │     │  • Modpacks, News                │      │    │
│  │     │  • Error Reports                 │      │    │
│  │     └──────────────────────────────────┘      │    │
│  │                                              │    │
│  │     ┌──────────────────────────────────┐      │    │
│  │     │  Redis (Sessions + Cache)        │      │    │
│  │     │  • Auth code (TTL: 10min)        │      │    │
│  │     │  • User session (TTL: 24h)       │      │    │
│  │     │  • Contatori rate limit          │      │    │
│  │     └──────────────────────────────────┘      │    │
│  │                                              │    │
│  │     ┌──────────────────────────────────┐      │    │
│  │     │  File Storage (S3-compatible)    │      │    │
│  │     │  • Binari modpack                │      │    │
│  │     │  • File gioco                    │      │    │
│  │     │  • Eseguibili launcher           │      │    │
│  │     │  • ~500GB per regione            │      │    │
│  │     └──────────────────────────────────┘      │    │
│  │                                              │    │
│  └──────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Dettagli Infrastruttura:**
- **Container:** Docker (Dockerfile incluso)
- **Orchestrazione:** docker-compose o Kubernetes
- **Web Server:** Nginx reverse proxy (SSL termination)
- **Database:** PostgreSQL 14+ (servizio gestito consigliato)
- **Cache:** Redis (per session, rate limiting)
- **Storage:** S3-compatible storage (S3, Wasabi, MinIO)
- **Monitoring:** Prometheus + Grafana (log, metriche)

---

## Decisioni Tecniche Chiave

### 1. Fingerprinting Hardware Invece di HWID

**Perché:**
- HWID singolo si rompe su cambio hardware minore
- Utenti non possono upgradare PC o reinstallare OS
- Scoring pesato gestisce match parziali

---

### 2. SHA256 Hash Tutti Componenti (No Seriali Raw)

**Perché:**
- Previene credential stuffing attack
- Blocca component tracking tra servizi
- Compliant con regolazioni privacy

---

### 3. Download Multi-Server con Selezione Automatica

**Perché:**
- Resilienza a blocchi regionali
- Load balancing geografico
- Failover automatico su server down

---

### 4. MVVM con Classi Partial (Non VM Multipli)

**Perché:**
- Oggetto state singolo per tutta l'app
- Testing più facile (mock ViewModel singolo)
- Separazione chiara per feature
- Righe per file rimangono gestibili (<500)

---

### 5. OAuth Telegram Invece di Email/Password

**Perché:**
- Nessuna password da compromettere
- 2FA built-in (sicurezza Telegram)
- Device linking facile via bot
- UX migliore (apri app, tocca pulsante, fatto)

---

Vedi documentazione dettagliata:
- **[classes.it.md](classes.it.md)** — Tutte classi .NET
- **[backend.it.md](backend.it.md)** — Route Express & servizi
- **[frontend.it.md](frontend.it.md)** — Componenti React
- **[telegram.it.md](telegram.it.md)** — Dettagli flusso OAuth
