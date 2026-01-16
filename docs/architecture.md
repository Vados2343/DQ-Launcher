# Architecture Overview

## System Design

DQ Launcher is a full-stack three-tier application designed for enterprise game distribution with device fingerprinting and hardware recognition.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Desktop Client                           │
│                    (WPF .NET 8 / Windows)                       │
│  • User authentication via Telegram OAuth                       │
│  • Hardware fingerprinting (SHA256 weighted scoring)            │
│  • Modpack management (install/switch/update)                  │
│  • Multi-server download orchestration                         │
│  • Auto-updater with rollback support                          │
│  • VPN detection & configuration                               │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS + JWT
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Backend API Server                         │
│                (Node.js/Express/TypeScript)                     │
│  • Telegram OAuth integration (webhooks)                       │
│  • Device recognition algorithm (weighted scoring)             │
│  • Hardware fingerprint matching & migration tracking          │
│  • Multi-server file distribution management                   │
│  • Admin panel API (user/device/modpack management)            │
│  • Real-time upload progress (WebSocket)                       │
│  • Audit logging & suspicious activity detection               │
│  • Rate limiting & security middleware                         │
└──────────────────────────────┬──────────────────────────────────┘
                               │ PostgreSQL
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Database (PostgreSQL)                      │
│  • Users, Sessions, Audit Logs                                 │
│  • DeviceFingerprints (hashed hardware components)             │
│  • UserDevices (many-to-many + metadata)                       │
│  • DeviceMigrations (change history with confidence)           │
│  • SharedHardwareComponents (fraud detection)                  │
│  • Modpacks, News, Error Reports                               │
└─────────────────────────────────────────────────────────────────┘
                               ▲
                               │ HTTPS
┌──────────────────────────────┴──────────────────────────────────┐
│                      Admin Panel Frontend                       │
│                   (React 18 / Vite / TypeScript)                │
│  • User management & device tracking                           │
│  • Real-time monitoring (dashboard)                            │
│  • Modpack upload & management                                 │
│  • Security alerts & device blocking                           │
│  • Audit log viewer with filtering                             │
│  • Error report analysis                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Desktop Client (.NET 8 / WPF)

### Design Patterns

**MVVM with Partial Classes**
- MainViewModel split by responsibility (Auth, Downloads, Games, Mods, Settings, Connection, Update)
- Each partial 150-380 lines, kept focused
- Single ViewModel instance drives entire UI
- ObservableProperty (CommunityToolkit.Mvvm) for property binding

**Dependency Injection**
Registered in `App.xaml.cs`:

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

### ViewModel Structure

MainViewModel uses partial classes to separate concerns:

| File | Responsibility | Lines |
|------|---------------|-------|
| MainViewModel.cs | Core state, initialization | ~460 |
| MainViewModel.Auth.cs | Login flow, token management | ~380 |
| MainViewModel.Downloads.cs | Download queue, progress | ~300 |
| MainViewModel.Games.cs | Game state, launch logic | ~250 |
| MainViewModel.Mods.cs | Modpack management | ~200 |
| MainViewModel.UI.cs | Theme, language, UI state | ~150 |

### Service Layer

**HardwareFingerprintService**
Collects hardware identifiers via WMI:
- TPM presence and version
- System UUID (SMBIOS)
- CPU ID
- MAC addresses (all NICs)
- Disk serial numbers
- GPU device IDs

All values hashed with SHA256 before transmission.

**ManifestDownloadService**
1. Fetch manifest from server (file list with checksums)
2. Compare with local files
3. Download missing/changed files (4 concurrent)
4. Verify each file: size check → SHA256 hash
5. On mismatch: delete and redownload (no resume)

**LauncherApiService**
- `InitiateAuthAsync()` — Start Telegram OAuth flow
- `CheckAuthStatusAsync()` — Poll for completion
- `AutoSelectBestServerAsync()` — Ping servers, select lowest latency
- `GetManifestAsync()` — Fetch file manifest
- `ReportErrorAsync()` — Send client errors to server

### Data Flow

```
User clicks Login
    │
    ▼
MainViewModel.LoginCommand
    │
    ▼
LauncherApiService.InitiateAuthAsync()
    ├── Collect HWID (legacy MD5)
    ├── Collect DeviceFingerprint (SHA256)
    └── POST /launcher/initiate
            │
            ▼
        Server recognizes device
        Returns: authToken, sessionId
            │
            ▼
    Poll /launcher/status until confirmed
            │
            ▼
    Store token in SettingsService
    Update IsAuthenticated = true
```

## Backend (Node.js/Express/TypeScript)

### Authentication Flow

```
1. LAUNCHER initiates auth
   POST /launcher/initiate
   ├── Send HWID (legacy MD5)
   ├── Send DeviceFingerprint (SHA256 hash all components)
   └── Receive authCode + Telegram deep link

2. USER approves in Telegram
   Tap t.me/dq_bot?start={authCode}
   └── Bot stores approval in Redis (600s TTL)

3. LAUNCHER polls status
   GET /launcher/status/{authCode}
   ├── Poll every 2 seconds
   ├── Server checks Redis for approval
   └── Return JWT token + downloadToken on success

4. LAUNCHER stores token
   localStorage/settings.json
   └── Use token for subsequent API requests
```

### Device Recognition Algorithm

```
Weighted Scoring (max 100):
┌──────────────────────────────────────────┐
│ Component       │ Weight │ Reliability   │
├─────────────────┼────────┼───────────────┤
│ TPM             │ 40%    │ Highest      │
│ System UUID     │ 25%    │ Very High    │
│ MAC Address     │ 15%    │ Medium*      │
│ Disk Serial     │ 10%    │ Medium       │
│ CPU ID          │ 5%     │ Low**        │
│ GPU ID          │ 5%     │ Low          │
└──────────────────────────────────────────┘

*MAC: Can be virtualized/spoofed easily
**CPU: Changes with BIOS reset

Decision Logic:
• Score >= 70% → Device recognized
  └─ Update device record
• 50% ≤ Score < 70% → Possible migration
  ├─ Check migration limit (2/year)
  ├─ Record migration with confidence
  └─ Flag for manual review if needed
• Score < 50% → New device
  └─ Create new DeviceFingerprint record
```

### Security Model

```
Threat Model:
┌─────────────────────────────────────┐
│ Account Sharing Detection           │
├─────────────────────────────────────┤
│ Multiple users with same:           │
│ • MAC address (virtual machines)    │
│ • Disk serial (shared drives)       │
│ • TPM (same hardware)               │
│ → Alert admin, flag suspicious      │
└─────────────────────────────────────┘

Device Limit Policy:
• Max 5 devices per user
• Max 2 migrations per 365 days
• Failed auth logged with IP
• Rapid device changes trigger alert
• IP change from 100+ countries = suspicious

Data Protection:
• All hardware IDs hashed client-side before transmission
• No raw serials stored in database
• JWT tokens: 24h lifetime
• Download tokens: 7d lifetime
• Rate limiting: 5 auth attempts per 15 min per IP
• Admin actions logged with timestamp + IP
```

### Route Structure

```
/launcher
├── POST /initiate      # Start auth, receive HWID + fingerprint
├── GET  /status/:id    # Poll auth status
├── GET  /manifest      # File list with checksums
└── GET  /download/:file # Authenticated file download

/admin
├── GET  /users         # List users (paginated)
├── GET  /users/:id     # User details + devices
├── GET  /users/:id/devices  # Device list
├── POST /users/:id/devices/:did/block
└── GET  /checksum-errors    # Client-reported errors

/security
├── GET  /logs          # Audit trail
└── GET  /suspicious    # Flagged activities
```

### Device Recognition Algorithm

```
Input: DeviceFingerprint { tpmHash, uuidHash, cpuIdHash, macHashes[], diskHashes[], gpuHashes[] }

1. Query existing fingerprints with matching components:
   - Exact match on tpmHash
   - Exact match on uuidHash
   - Any intersection on macHashes
   - Any intersection on diskHashes

2. For each candidate, calculate weighted score:
   TPM match:     +40 points
   UUID match:    +25 points
   MAC match:     +15 points (any)
   Disk match:    +10 points (any)
   CPU match:     +5 points
   GPU match:     +5 points
   ─────────────────────────
   Max:           100 points

3. If score >= 70: device recognized
   If 50 <= score < 70: possible migration, require confirmation
   If score < 50: new device

4. Track migrations per user (limit: 2/year)
```

### Database Schema

Key tables:

**DeviceFingerprint**
- Stores SHA256 hashes of hardware components
- Array fields for multi-value components (MACs, disks)
- Indexed for fast lookup

**UserDevice**
- Many-to-many: User ↔ DeviceFingerprint
- Status: ACTIVE | SUSPENDED | BLOCKED
- Trust level: UNVERIFIED | VERIFIED | TRUSTED
- Migration counter

**DeviceMigration**
- Audit trail of device changes
- Confidence score at migration time
- Changed vs matched components

**SharedHardwareComponent**
- Flags when same component appears for multiple users
- Used for fraud detection

## Security Considerations

1. **No raw serials transmitted** — All hardware IDs hashed client-side
2. **Rate limiting** — Per-IP and per-user limits on auth endpoints
3. **JWT tokens** — Short-lived access tokens, refresh via Telegram
4. **Audit logging** — All admin actions logged with IP and timestamp
5. **Device limits** — Users can have max 3 active devices
6. **Migration limits** — Max 2 device migrations per year

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               Production Environment                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌──────────────────┐  ┌──────────────────┐                │
│ │ Telegram Bot     │  │ Frontend (React) │                │
│ │ Webhooks         │  │ Vite SPA         │                │
│ │ (tg.me/dq_bot)   │  │ TailwindCSS      │                │
│ └────────┬─────────┘  └────────┬─────────┘                │
│          │                     │                          │
│          └─────────────────────┼──────────────────────────│
│                                ▼                          │
│                  ┌──────────────────────────┐             │
│                  │  Express Backend         │             │
│                  │  • /launcher/* routes    │             │
│                  │  • /admin/* routes       │             │
│                  │  • /download/* routes    │             │
│                  │  • WebSocket (uploads)   │             │
│                  │  • JWT middleware        │             │
│                  │  • Rate limiting         │             │
│                  └────────┬──────────────────┘            │
│                           │                              │
│  ┌────────────────────────┼────────────────────────┐    │
│  │                        ▼                        │    │
│  │     ┌──────────────────────────────────┐      │    │
│  │     │  PostgreSQL Database             │      │    │
│  │     │  • Users, Devices, Sessions      │      │    │
│  │     │  • DeviceFingerprints (indexed)  │      │    │
│  │     │  • Audit Logs (retention: 90d)  │      │    │
│  │     │  • Modpacks, News                │      │    │
│  │     │  • Error Reports                 │      │    │
│  │     └──────────────────────────────────┘      │    │
│  │                                              │    │
│  │     ┌──────────────────────────────────┐      │    │
│  │     │  Redis (Sessions + Cache)        │      │    │
│  │     │  • Auth codes (TTL: 10min)       │      │    │
│  │     │  • User sessions (TTL: 24h)      │      │    │
│  │     │  • Rate limit counters           │      │    │
│  │     └──────────────────────────────────┘      │    │
│  │                                              │    │
│  │     ┌──────────────────────────────────┐      │    │
│  │     │  File Storage (S3-compatible)    │      │    │
│  │     │  • Modpack binaries              │      │    │
│  │     │  • Game files                    │      │    │
│  │     │  • Launcher executables          │      │    │
│  │     │  • ~500GB per region             │      │    │
│  │     └──────────────────────────────────┘      │    │
│  │                                              │    │
│  └──────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Infrastructure Details:**
- **Container:** Docker (Dockerfile included)
- **Orchestration:** docker-compose or Kubernetes
- **Web Server:** Nginx reverse proxy (SSL termination)
- **Database:** PostgreSQL 14+ (managed service recommended)
- **Cache:** Redis (for sessions, rate limiting)
- **Storage:** S3-compatible storage (S3, Wasabi, MinIO)
- **Monitoring:** Prometheus + Grafana (logs, metrics)

---

## Key Technical Decisions

### 1. Hardware Fingerprinting Instead of HWID

**Why:**
- Single HWID breaks on minor hardware change
- Users can't upgrade PC or reinstall OS
- Weighted scoring handles partial matches

**Trade-off:**
- More complex algorithm
- Requires ML-like confidence scoring
- Better UX for legitimate users

---

### 2. SHA256 Hash All Components (No Raw Serials)

**Why:**
- Prevents credential stuffing attacks
- Blocks component tracking across services
- Compliant with privacy regulations

**Implementation:**
- Hash on client-side before transmission
- Store only hashes in database
- No way to reverse (one-way function)

---

### 3. Multi-Server Download with Automatic Selection

**Why:**
- Resilience to regional blocking
- Geographic load balancing
- Automatic failover on server down

**Algorithm:**
1. Ping all servers in parallel
2. Select by lowest latency
3. Remember user preference
4. Auto-switch on 3 consecutive failures

---

### 4. MVVM with Partial Classes (Not Multiple ViewModels)

**Why:**
- Single state object for entire app
- Easier testing (one mock ViewModel)
- Clear separation by feature
- Lines per file stay manageable (<500)

**Alternative considered:** Separate VM per page
- ❌ Would require complex message passing
- ❌ State coordination becomes nightmare

---

### 5. Telegram OAuth Instead of Email/Password

**Why:**
- No passwords to compromise
- 2FA built-in (Telegram security)
- Easy device linking via bot
- Better UX (open app, tap button, done)

**Trade-off:**
- Requires Telegram account
- Telegram service outage = auth outage

---

## Performance Characteristics

| Operation | Latency | Notes |
|-----------|---------|-------|
| Hardware fingerprint collection | 200-500ms | WMI queries on each component |
| Auth initiation (POST) | 150-300ms | Database + Redis write |
| Auth polling (GET) | 50-100ms | Cache hit in most cases |
| File manifest fetch | 300-800ms | Network transfer + parsing |
| Single file download | Bandwidth limited | 4 concurrent streams |
| Modpack installation | 5-30s | Copy + extraction + verification |
| Game launch | <1s | Process spawn only |

---

## Data Flow Diagrams

### Authentication Sequence
```
┌─────────────┐                    ┌─────────────┐               ┌────────┐
│   Launcher  │                    │   Backend   │               │Telegram│
└──────┬──────┘                    └──────┬──────┘               └───┬────┘
       │                                  │                         │
       │──POST /launcher/initiate────────>│                         │
       │ (hwid, fingerprint)              │                         │
       │                                  │──POST setWebhook───────>│
       │                                  │                         │
       │<──────authCode + deepLink────────│                         │
       │                                  │                         │
       │ [Display to user]                │                         │
       │                                  │                         │
       │ User taps deep link              │                         │
       │────────────────────────────────────────────────────────────>│
       │                                  │                         │
       │                                  │<────/start handler──────│
       │                                  │ (approve auth)          │
       │                                  │                         │
       │──GET /launcher/status────────────>│                         │
       │         (authCode)               │ (check Redis)           │
       │                                  │                         │
       │<──token + downloadToken──────────│                         │
       │                                  │                         │
```

---

Refer to detailed documentation:
- **[classes.md](classes.md)** — All .NET classes
- **[backend.md](backend.md)** — Express routes & services
- **[frontend.md](frontend.md)** — React components
- **[telegram.md](telegram.md)** — OAuth flow details
