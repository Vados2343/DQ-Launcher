# Backend API (Node.js/Express/TypeScript)

Riferimento completo servizi backend DQ Launcher.

**Stack:** Node.js v18+ · Express.js · TypeScript · Prisma ORM · PostgreSQL

## Struttura Directory

```
site/backend/
├── src/
│   ├── index.ts                     # Express app, setup route
│   ├── config/                      # Env, porte, URL
│   ├── lib/                         # Prisma, JWT, retry DB
│   ├── middleware/                  # Auth, error handler, logger
│   ├── routes/                      # Tutte le rotte API
│   ├── services/                    # Device recognition, Telegram, upload
│   └── types/                       # Interfacce TypeScript
├── prisma/
│   ├── schema.prisma                # Modelli dati
│   └── migrations/                  # Migrazioni database
├── public/launcher/                 # Download launcher
├── Dockerfile
└── docker-compose.yml
```

## Rotte Principali

### POST /launcher/initiate
**Avvia flusso autenticazione**

**Request:**
```json
{
  "hwid": "md5hash...",
  "fingerprint": {
    "tpmHash": "sha256...",
    "systemUuidHash": "sha256...",
    "macAddressHashes": ["sha256..."]
  }
}
```

**Response:**
```json
{
  "success": true,
  "status": "pending",
  "authCode": "auth_abc123",
  "visibleCode": "123456",
  "deepLink": "https://t.me/dq_bot?start=auth_abc123",
  "expiresIn": 600
}
```

---

### GET /launcher/status/:authCode
**Poll stato autenticazione**

**Response (pendente):**
```json
{
  "success": true,
  "status": "pending"
}
```

**Response (autenticato):**
```json
{
  "success": true,
  "status": "authenticated",
  "token": "eyJhbGc...",
  "downloadToken": "dl_token...",
  "user": {
    "id": "user_123",
    "username": "vados2343",
    "role": "USER"
  }
}
```

---

### GET /launcher/manifest
**Lista file da scaricare**

**Headers:** `Authorization: Bearer {downloadToken}`

**Response:**
```json
{
  "files": [
    {
      "path": "game/textures/car.dds",
      "size": 1048576,
      "checksum": "sha256hash...",
      "url": "/download/game/textures/car.dds"
    }
  ]
}
```

---

### GET /download/:filePath
**Scarica file con autenticazione**

**Headers:** `Authorization: Bearer {downloadToken}`

**Behavior:**
- Autenticazione downloadToken
- Log download
- Stream file da storage
- Track bandwidth

---

### GET /admin/users
**Lista utenti (solo admin)**

**Headers:** `Authorization: Bearer {adminToken}`

**Query:** `?page=1&limit=50&search=username`

**Response:**
```json
{
  "users": [
    {
      "id": "user_123",
      "username": "vados2343",
      "telegramId": "123456789",
      "createdAt": "2025-12-01T10:00:00Z",
      "deviceCount": 3,
      "lastLoginAt": "2026-01-16T09:30:00Z",
      "role": "USER"
    }
  ],
  "total": 150,
  "page": 1
}
```

---

### GET /admin/users/:userId/devices
**Lista dispositivi utente specifico**

**Response:**
```json
{
  "devices": [
    {
      "id": "device_123",
      "status": "ACTIVE",
      "trustLevel": "VERIFIED",
      "firstSeen": "2025-12-15T00:00:00Z",
      "lastSeen": "2025-01-16T09:30:00Z",
      "migrationCount": 1,
      "loginCount": 45
    }
  ]
}
```

---

### POST /admin/users/:userId/devices/:deviceId/block
**Sospendi o blocca dispositivo**

**Request:**
```json
{
  "action": "SUSPEND|BLOCK",
  "reason": "Suspected account sharing",
  "durationDays": 7
}
```

---

### GET /launcher/version
**Controlla aggiornamenti launcher**

**Response:**
```json
{
  "version": "3.1.0",
  "downloadUrl": "/launcher/DQ Launcher.exe",
  "changelog": "Enterprise HWID system...",
  "mandatory": false,
  "checksum": "sha256..."
}
```

---

## Servizi

### deviceRecognitionService.ts

**Scopo:** Matching fingerprint hardware con algoritmo pesato

**Funzioni Principali:**

#### recognizeDevice(fingerprint, userId?)
Trova dispositivi corrispondenti nel database

**Algoritmo Scoring:**
```
TPM match:    +40 punti (più affidabile)
UUID match:   +25 punti
MAC match:    +15 punti (qualsiasi MAC)
Disk match:   +10 punti (qualsiasi disco)
CPU match:    +5 punti
GPU match:    +5 punti
─────────────────────
Massimo:      100 punti

Score >= 70:  Dispositivo riconosciuto
50 <= score < 70: Possibile migrazione
score < 50:   Nuovo dispositivo
```

#### checkDeviceForUser(userId, fingerprint, ipAddress)
Controllo sicurezza per login

**Logica:**
1. Riconosce dispositivo
2. Se riconosciuto (score >= 70): aggiorna last seen
3. Se possibile migrazione (50-70): verifica limite migrazione/anno
4. Se nuovo: crea nuovo DeviceFingerprint

#### trackSharedComponents(fingerprint, userId)
Rileva account sharing via riuso hardware

---

### telegramBot.ts

**Scopo:** OAuth Telegram e comandi bot

**Flusso OAuth:**
```
1. User clicca login in launcher
2. Launcher reindirizza a Telegram
3. Bot /start handler riceve authCode
4. Memorizza approvazione in Redis
5. Launcher poll fino a conferma
6. Server crea JWT token
7. Launcher salva token
```

**Comandi Bot:**
- `/start authCode` — OAuth flow
- `/account` — Mostra info account
- `/devices` — Lista dispositivi collegati
- `/bind` — Aggiungi nuovo dispositivo
- `/unbind` — Rimuovi dispositivo
- `/support` — Link risorse help

---

### uploadProgressService.ts

**Scopo:** Tracciamento progresso real-time upload

**Implementazione:**
- WebSocket per notifiche progresso
- Tracking chunk upload
- Stima tempo rimanente
- Validazione integrità

---

## Database (Prisma Schema)

### User
```prisma
model User {
  id              String    @id
  username        String    @unique
  email           String?   @unique
  telegramId      String?   @unique
  role            UserRole  @default(USER)
  createdAt       DateTime  @default(now())
  devices         UserDevice[]
  sessions        Session[]
  migrations      DeviceMigration[]
}
```

### DeviceFingerprint
```prisma
model DeviceFingerprint {
  id                  String    @id
  tpmHash             String?
  systemUuidHash      String?   @unique
  cpuIdHash           String?
  macAddressHashes    String[]
  diskSerialHashes    String[]
  gpuIdHashes         String[]
  componentCount      Int
  firstSeenAt         DateTime  @default(now())
  lastSeenAt          DateTime  @updatedAt
  userDevices         UserDevice[]
}
```

### UserDevice
```prisma
model UserDevice {
  id                  String          @id
  userId              String
  fingerprintId       String
  status              DeviceStatus    @default(ACTIVE)
  trustLevel          TrustLevel      @default(UNVERIFIED)
  migrationCount      Int             @default(0)
  loginCount          Int             @default(0)
  createdAt           DateTime        @default(now())
  lastLoginAt         DateTime?
  user                User
  fingerprint         DeviceFingerprint
}
```

### DeviceMigration
```prisma
model DeviceMigration {
  id                  String    @id
  userId              String
  confidenceScore     Float
  migrationType       MigrationType
  changedComponents   String[]
  matchedComponents   String[]
  createdAt           DateTime  @default(now())
  user                User
}
```

---

## Middleware

### auth.ts
Verifica JWT

```typescript
export async function authenticateToken(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

export function requireRole(...roles: string[]) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
```

### errorHandler.ts
Gestione centralizzata errori

---

## Variabili Ambiente

```env
NODE_ENV=production
PORT=5000
DATABASE_URL=postgresql://user:pass@localhost/dq_launcher
JWT_SECRET=your-secret-key
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234...
STORAGE_BASE_URL=https://storage-ger.dq.local
REDIS_URL=redis://localhost:6379
```

---

## Sviluppo

```bash
npm install
npx prisma generate
npx prisma migrate dev
npm run dev
```

### Studio Database
```bash
npx prisma studio
```

### Test
```bash
npm test
npm run test:coverage
```
