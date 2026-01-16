# DQ Launcher

Game launcher desktop per la community DQ con riconoscimento dispositivi enterprise-grade, distribuzione file multi-server e gestione modpack.

## Stack Tecnologico

**Client Desktop**
- .NET 8 / WPF
- CommunityToolkit.Mvvm (architettura MVVM)
- Microsoft.Extensions.DependencyInjection
- Material Design UI
- Fingerprinting hardware SHA256 via WMI

**Backend**
- Node.js / Express / TypeScript
- Prisma ORM + PostgreSQL
- Integrazione Telegram OAuth
- Autenticazione JWT

**Pannello Admin**
- React 18 / Vite
- TailwindCSS
- TanStack Query / Zustand

## Funzionalità

- **Riconoscimento Dispositivi** — Fingerprinting hardware pesato (TPM, UUID, MAC, Disk, CPU, GPU) con matching basato su confidence invece di confronto hash rigido
- **Download Intelligenti** — Download paralleli multi-server con selezione automatica per latenza, verifica checksum, logica di retry
- **Sistema Modpack** — Installazione, aggiornamento, switch tra modifiche di gioco con risoluzione conflitti
- **Auto-Updater** — Aggiornamenti delta con supporto rollback
- **Pannello Admin** — Gestione utenti, tracking migrazione dispositivi, log audit sicurezza

## Architettura

```
┌─────────────────────────────────────────────────────────────┐
│                    DQ Launcher (WPF)                        │
├─────────────────────────────────────────────────────────────┤
│  ViewModels          │  Services              │  Core       │
│  ├─ MainViewModel    │  ├─ LauncherApiService │  ├─ DI      │
│  ├─ Auth partial     │  ├─ HardwareFingerprint│  ├─ Config  │
│  ├─ Downloads        │  ├─ ManifestDownload   │  └─ Paths   │
│  └─ UI/Theme         │  ├─ ModManager         │             │
│                      │  └─ VpnService         │             │
└──────────────────────┴────────────────────────┴─────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend (Express)                        │
├─────────────────────────────────────────────────────────────┤
│  Routes              │  Services              │  Middleware │
│  ├─ /launcher/*      │  ├─ DeviceRecognition  │ ├─ Auth     │
│  ├─ /admin/*         │  ├─ TelegramBot        │ ├─ RateLimit│
│  └─ /security/*      │  └─ UploadTracker      │ └─ ErrorH   │
└──────────────────────┴────────────────────────┴─────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    PostgreSQL (Prisma)                      │
│  User, Session, DeviceFingerprint, UserDevice,              │
│  DeviceMigration, SharedHardwareComponent                   │
└─────────────────────────────────────────────────────────────┘
```

## Screenshots

| Finestra Principale | Progresso Download | Pannello Admin |
|---------------------|-------------------|----------------|
| ![Main](docs/screenshots/main-window.png) | ![Download](docs/screenshots/download.png) | ![Admin](docs/screenshots/admin.png) |

## Decisioni Tecniche Chiave

**Fingerprinting HWID**
Sistema di scoring pesato: TPM (40%), System UUID (25%), indirizzi MAC (15%), seriali Disk (10%), CPU/GPU (10%). Tutti i componenti hashati con SHA256. Permette riconoscimento dispositivo dopo reinstallazione OS o modifiche hardware parziali.

**Pipeline Download**
4 download concorrenti per sessione. Opzioni file WriteThrough per prevenire corruzione. Verifica dimensione prima del controllo hash. Retry automatico con backoff esponenziale.

**MVVM con Classi Partial**
MainViewModel diviso in partial per responsabilità: Auth, Downloads, Games, Mods, UI. Mantiene ogni file sotto 500 righe preservando singola istanza ViewModel.

## Struttura Progetto

```
DQ Launcher/
├── Core/                    # Costanti, percorsi, versione
├── Services/
│   ├── Interfaces/          # Contratti servizi
│   └── Implementations/     # Logica servizi
├── ViewModels/              # MVVM view models
├── Models/                  # Modelli dati
├── Converters/              # WPF value converters
├── Styles/                  # XAML resource dictionaries
├── UI/                      # Controlli custom
└── Updater/                 # App auto-updater separata
```

## Documentazione

- **[Architettura](docs/architecture.it.md)** — Design del sistema, flusso dati, pattern
- **[Classi & Servizi](docs/classes.it.md)** — Client desktop .NET (WPF, ViewModels, Services)
- **[Backend API](docs/backend.it.md)** — Servizi Node.js/Express/TypeScript
- **[Frontend](docs/frontend.it.md)** — Pannello admin React
- **[Integrazione Telegram](docs/telegram.it.md)** — OAuth, comandi bot

## Build

```bash
dotnet publish -c Release -r win-x64 --self-contained
```

Output: eseguibile single-file con runtime embedded.

## Versione

Attuale: **3.1.0**

---

