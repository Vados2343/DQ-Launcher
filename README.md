# DQ Launcher

Desktop game launcher for DQ Community with enterprise-grade device recognition, multi-server file distribution, and modpack management.

![Main Window](docs/screenshots/main-window.png)

## Tech Stack

**Desktop Client**
- .NET 8 / WPF
- CommunityToolkit.Mvvm (MVVM architecture)
- Microsoft.Extensions.DependencyInjection
- Material Design UI
- SHA256 hardware fingerprinting via WMI

**Backend**
- Node.js / Express / TypeScript
- Prisma ORM + PostgreSQL
- Telegram OAuth integration
- JWT authentication

**Admin Panel**
- React 18 / Vite
- TailwindCSS
- TanStack Query / Zustand

## Features

- **Device Recognition** — Weighted hardware fingerprinting (TPM, UUID, MAC, Disk, CPU, GPU) with confidence-based matching instead of rigid hash comparison
- **Smart Downloads** — Multi-server parallel downloads with automatic server selection by latency, checksum verification, retry logic
- **Modpack System** — Install, update, switch between game modifications with conflict resolution
- **Auto-Updater** — Delta updates with rollback support
- **Admin Panel** — User management, device migration tracking, security audit logs

## Architecture

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
│  ├─ /launcher/*      │  ├─ DeviceRecognition  │  ├─ Auth    │
│  ├─ /admin/*         │  ├─ TelegramBot        │  ├─ RateLimit│
│  └─ /security/*      │  └─ UploadTracker      │  └─ ErrorH  │
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

| Main Window | Download Progress | Admin Panel |
|-------------|-------------------|-------------|
| ![Main](docs/screenshots/main-window.png) | ![Download](docs/screenshots/download.png) | ![Admin](docs/screenshots/admin.png) |

## Key Technical Decisions

**HWID Fingerprinting**
Weighted scoring system: TPM (40%), System UUID (25%), MAC addresses (15%), Disk serials (10%), CPU/GPU (10%). All components hashed with SHA256. Allows device recognition after OS reinstall or partial hardware changes.

**Download Pipeline**
4 concurrent downloads per session. WriteThrough file options to prevent corruption. Size verification before hash check. Automatic retry with exponential backoff.

**MVVM with Partial Classes**
MainViewModel split into partials by responsibility: Auth, Downloads, Games, Mods, UI. Keeps each file under 500 lines while maintaining single ViewModel instance.

## Project Structure

```
DQ Launcher/
├── Core/                    # Constants, paths, version
├── Services/
│   ├── Interfaces/          # Service contracts
│   └── Implementations/     # Service logic
├── ViewModels/              # MVVM view models
├── Models/                  # Data models
├── Converters/              # WPF value converters
├── Styles/                  # XAML resource dictionaries
├── UI/                      # Custom controls
└── Updater/                 # Separate auto-updater app
```

## Documentation

- **[Architecture](docs/architecture.md)** — System design, data flow, patterns
- **[Classes & Services](docs/classes.md)** — .NET desktop client (WPF, ViewModels, Services)
- **[Backend API](docs/backend.md)** — Node.js/Express/TypeScript services
- **[Frontend](docs/frontend.md)** — React admin panel
- **[Telegram Integration](docs/telegram.md)** — OAuth, bot commands

## Building

```bash
dotnet publish -c Release -r win-x64 --self-contained
```

Output: single-file executable with embedded runtime.

## Version

Current: **3.1.0**

---

**Developed by:** Vados2343
**Repository:** DQ Launcher
