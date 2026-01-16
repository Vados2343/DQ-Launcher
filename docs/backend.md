# Backend API (Node.js/Express/TypeScript)

Complete reference of DQ Launcher backend services.

**Stack:** Node.js v18+ · Express.js · TypeScript · Prisma ORM · PostgreSQL

## Directory Structure

```
site/backend/
├── src/
│   ├── index.ts                     # Express app, routes setup
│   ├── config/
│   │   ├── index.ts                 # Environment, ports, URLs
│   │   └── launcherVersion.ts       # Sync with launcher version
│   │
│   ├── lib/
│   │   ├── prisma.ts                # Prisma client singleton
│   │   ├── jwt.ts                   # JWT sign/verify
│   │   └── dbRetry.ts               # Retry logic for DB operations
│   │
│   ├── middleware/
│   │   ├── auth.ts                  # JWT verification
│   │   ├── errorHandler.ts          # Global error handler
│   │   ├── requestLogger.ts         # Request/response logging
│   │   └── staticFallback.ts        # Serve React static files
│   │
│   ├── routes/
│   │   ├── launcher.ts              # Manifest, download files
│   │   ├── launcherAuth.ts          # POST /launcher/initiate, /status
│   │   ├── admin.ts                 # User management, device tracking
│   │   ├── adminLogs.ts             # Audit logs
│   │   ├── auth.ts                  # Login, logout, refresh
│   │   ├── download.ts              # File download with auth
│   │   ├── errorReports.ts          # Client error reports
│   │   ├── metrics.ts               # App metrics, analytics
│   │   ├── modpacks.ts              # List, upload modpacks
│   │   ├── news.ts                  # News/announcements CRUD
│   │   └── security.ts              # Rate limiting, suspicious activity
│   │
│   ├── services/
│   │   ├── deviceRecognitionService.ts
│   │   │   ├── recognizeDevice()
│   │   │   ├── calculateWeightedScore()
│   │   │   ├── checkDeviceForUser()
│   │   │   └── trackSharedComponents()
│   │   │
│   │   ├── telegramBot.ts           # Telegram OAuth, bot commands
│   │   ├── uploadProgressService.ts # Real-time progress tracking
│   │   ├── uploadTracker.ts         # Multipart upload management
│   │   └── (other service files)
│   │
│   └── types/
│       └── index.ts                 # TypeScript interfaces
│
├── prisma/
│   ├── schema.prisma                # Data models
│   ├── migrations/                  # Database migrations
│   └── seed.ts                      # Seed data script
│
├── public/
│   └── launcher/
│       ├── version.json             # Current launcher version
│       ├── DQ Launcher.exe           # Download endpoint
│       └── changelog.md
│
├── Dockerfile
├── docker-compose.yml
├── package.json
├── tsconfig.json
└── .env.example
```

## Routes

### POST /launcher/initiate
**Purpose:** Start authentication flow

**Request:**
```json
{
  "hwid": "md5hash...",
  "legacyHwid": "old...",
  "fingerprint": {
    "tpmHash": "sha256...",
    "systemUuidHash": "sha256...",
    "cpuIdHash": "sha256...",
    "macAddressHashes": ["sha256..."],
    "diskSerialHashes": ["sha256..."],
    "gpuIdHashes": ["sha256..."]
  }
}
```

**Response (Success):**
```json
{
  "success": true,
  "status": "pending",
  "authCode": "auth_1234567890",
  "visibleCode": "123456",
  "deepLink": "https://t.me/dq_bot?start=auth_1234567890",
  "expiresIn": 600
}
```

**Response (Already authenticated):**
```json
{
  "success": true,
  "status": "authenticated",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "downloadToken": "dl_token...",
  "user": {
    "id": "user_123",
    "username": "vados2343",
    "role": "USER"
  }
}
```

**Processing:**
1. Validate HWID + fingerprint format
2. Call `deviceRecognitionService.recognizeDevice()`
3. If confidence >= 70%: create/update device binding
4. Generate auth code + Telegram deep link
5. Store in Redis (expires 10 min)
6. Return auth code + link

---

### GET /launcher/status/:authCode
**Purpose:** Poll authentication status

**Query:** `?hwid=...`

**Response (Still waiting):**
```json
{
  "success": true,
  "status": "pending"
}
```

**Response (User approved in Telegram):**
```json
{
  "success": true,
  "status": "authenticated",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "downloadToken": "dl_token...",
  "user": {
    "id": "user_123",
    "username": "vados2343",
    "role": "USER"
  }
}
```

**Response (Expired):**
```json
{
  "success": false,
  "status": "expired"
}
```

---

### GET /launcher/manifest
**Purpose:** Get list of game files to download

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
  ],
  "checksum": "sha256hash of all files"
}
```

---

### GET /download/:filePath
**Purpose:** Download file with authentication

**Headers:** `Authorization: Bearer {downloadToken}`

**Behavior:**
- Authenticate downloadToken
- Log download to analytics
- Stream file from storage
- Track bandwidth usage

**Response:** Binary file stream

---

### GET /admin/users
**Purpose:** List all users (admin only)

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
**Purpose:** List devices for specific user

**Response:**
```json
{
  "devices": [
    {
      "id": "device_123",
      "status": "ACTIVE",
      "trustLevel": "VERIFIED",
      "firstSeen": "2025-12-01T00:00:00Z",
      "lastSeen": "2026-01-16T09:30:00Z",
      "migrationCount": 1,
      "loginCount": 45,
      "fingerprint": {
        "tpmHash": "...",
        "systemUuidHash": "...",
        "cpuIdHash": "...",
        "components": 12
      }
    }
  ]
}
```

---

### GET /admin/users/:userId/migrations
**Purpose:** Device migration history

**Response:**
```json
{
  "migrations": [
    {
      "id": "mig_123",
      "confidenceScore": 85,
      "migrationType": "HARDWARE_UPGRADE",
      "changedComponents": ["BOARD_SERIAL"],
      "matchedComponents": ["CPU_ID", "MAC_1", "MAC_2"],
      "createdAt": "2026-01-10T14:00:00Z"
    }
  ]
}
```

---

### POST /admin/users/:userId/devices/:deviceId/block
**Purpose:** Suspend or block a device

**Request:**
```json
{
  "action": "SUSPEND|BLOCK",
  "reason": "Suspected account sharing",
  "durationDays": 7
}
```

**Response:**
```json
{
  "success": true,
  "device": { ...updated device... }
}
```

---

### GET /admin/security/suspicious
**Purpose:** List suspicious activities

**Response:**
```json
{
  "activities": [
    {
      "id": "sus_123",
      "userId": "user_123",
      "type": "RAPID_DEVICE_CHANGE|SHARED_COMPONENT|FAILED_AUTH",
      "severity": "HIGH|MEDIUM|LOW",
      "description": "Multiple login attempts from different devices in 1 hour",
      "ipAddresses": ["123.45.67.89"],
      "deviceIds": ["device_123", "device_456"],
      "createdAt": "2026-01-16T09:00:00Z",
      "resolved": false
    }
  ]
}
```

---

### GET /launcher/version
**Purpose:** Check for launcher updates

**Response:**
```json
{
  "version": "3.1.0",
  "downloadUrl": "/launcher/DQ Launcher.exe",
  "changelog": "Enterprise HWID system...",
  "mandatory": false,
  "fileSize": 45678901,
  "checksum": "sha256..."
}
```

---

## Services

### deviceRecognitionService.ts

**Purpose:** Hardware fingerprint matching algorithm

**Main Functions:**

#### recognizeDevice(fingerprint, userId?)
Finds matching devices in database

```typescript
function recognizeDevice(
  fingerprint: DeviceFingerprint,
  userId?: string
): MatchResult {
  // 1. Query DeviceFingerprint table for candidates
  const candidates = await findCandidateFingerprints({
    tpmHash: fingerprint.tpmHash,
    macHashes: fingerprint.macAddressHashes,
    diskHashes: fingerprint.diskSerialHashes
  });

  // 2. Calculate weighted score for each candidate
  const scored = candidates.map(c => ({
    fingerprint: c,
    confidence: calculateWeightedScore(fingerprint, c),
    matchedComponents: findMatchedComponents(fingerprint, c)
  }));

  // 3. Sort by confidence, return top matches
  return {
    matched: scored.filter(s => s.confidence >= 50),
    totalCandidates: scored.length,
    bestMatch: scored[0]?.confidence >= 70 ? scored[0] : null
  };
}
```

**Scoring Algorithm:**
```
TPM match:    +40 points (most reliable)
UUID match:   +25 points
MAC match:    +15 points (any MAC)
Disk match:   +10 points (any disk)
CPU match:    +5 points
GPU match:    +5 points
─────────────────────
Maximum:      100 points

Score >= 70:  Recognized device
50 <= score < 70: Possible migration
score < 50:   New device
```

#### checkDeviceForUser(userId, fingerprint, ipAddress)
Security check for login

```typescript
async function checkDeviceForUser(
  userId: string,
  fingerprint: DeviceFingerprint,
  ipAddress: string
): Promise<DeviceCheckResult> {
  // 1. Recognize device
  const match = recognizeDevice(fingerprint, userId);

  // 2. If recognized (score >= 70):
  if (match.bestMatch?.confidence >= 70) {
    // Update last seen, increment login count
    await updateDeviceLoginHistory(match.bestMatch.id, ipAddress);
    return { allowed: true, result: match };
  }

  // 3. If possible migration (50 <= score < 70):
  if (match.bestMatch?.confidence >= 50) {
    const migs = await getUserMigrationCount(userId);
    if (migs.yearCount >= 2) {
      // Migration limit exceeded
      return { allowed: false, reason: "Migration limit" };
    }

    // Record migration
    await createDeviceMigration(userId, match.bestMatch);
    return { allowed: true, result: match, requiresConfirmation: true };
  }

  // 4. If new device:
  // Create new DeviceFingerprint record
  const newDevice = await createNewDevice(fingerprint);
  return { allowed: true, result: { ...match, newDeviceId: newDevice.id } };
}
```

#### trackSharedComponents(fingerprint, userId)
Detect account sharing via hardware reuse

```typescript
async function trackSharedComponents(
  fingerprint: DeviceFingerprint,
  userId: string
) {
  // For each component (MAC, disk serial, TPM):
  const components = extractComponents(fingerprint);

  for (const component of components) {
    const otherUsers = await findOtherUsersWithComponent(
      component.type,
      component.hash
    );

    if (otherUsers.length > 1) {
      // Create alert for suspicious activity
      await createSuspiciousActivity({
        type: "SHARED_COMPONENT",
        severity: otherUsers.length > 2 ? "HIGH" : "MEDIUM",
        userIds: [userId, ...otherUsers],
        component: component
      });
    }
  }
}
```

---

### telegramBot.ts

**Purpose:** Telegram OAuth and bot commands

**Setup:**
```typescript
import { Telegraf } from 'telegraf';
import { LoginPayload } from 'telegraf/types';

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN);

// Webhook: POST /telegram/webhook
bot.launch({
  webhook: {
    domain: process.env.TELEGRAM_WEBHOOK_URL,
    port: 5000
  }
});
```

**OAuth Flow:**
```typescript
// Step 1: User clicks login in launcher
// Launcher redirects to Telegram:
const deepLink = `https://t.me/${BOT_USERNAME}?start=${authCode}`;

// Step 2: Bot /start handler
bot.start((ctx) => {
  const authCode = ctx.startPayload;

  // Store user's Telegram ID
  const updates = {
    telegramId: ctx.from.id,
    username: ctx.from.username
  };

  // Mark auth code as approved in Redis
  await redis.set(`auth:${authCode}:approved`, JSON.stringify(updates), 'EX', 600);

  // Send confirmation to user
  ctx.reply('✅ Account linked! Return to DQ Launcher to continue.');
});

// Step 3: Launcher polls GET /launcher/status/:authCode
// Server checks Redis for approval, returns token
```

**Bot Commands:**

#### /help
List available commands

#### /account
Show account info:
```
🎮 Account: vados2343
📱 Devices: 3 (2 active, 1 suspended)
📅 Member since: Dec 15, 2025
🔒 Trust level: VERIFIED
```

#### /devices
List linked devices:
```
1️⃣ Main PC (ACTIVE)
   CPU: Intel i7 | RAM: 32GB | Last: 2h ago

2️⃣ Laptop (VERIFIED)
   CPU: AMD R5 | RAM: 16GB | Last: 5d ago

3️⃣ Old Setup (BLOCKED)
   Status: Suspended until Jan 30
```

#### /bind
Re-bind new device (sends auth code)

#### /unbind
Remove device from account

#### /support
Link to support page

---

### uploadProgressService.ts

**Purpose:** Real-time progress tracking for large uploads

**Implementation:**
```typescript
class UploadProgressService {
  private uploads: Map<string, UploadSession> = new Map();

  startSession(uploadId: string, fileSize: number) {
    this.uploads.set(uploadId, {
      uploadId,
      fileSize,
      bytesUploaded: 0,
      startTime: Date.now(),
      chunks: new Map()
    });
  }

  updateProgress(uploadId: string, chunkIndex: number, chunkSize: number) {
    const session = this.uploads.get(uploadId);
    if (!session) return;

    session.chunks.set(chunkIndex, chunkSize);
    session.bytesUploaded = Array.from(session.chunks.values())
      .reduce((sum, size) => sum + size, 0);

    // Emit progress via WebSocket
    this.emitProgress(uploadId, {
      bytesUploaded: session.bytesUploaded,
      totalBytes: session.fileSize,
      percentComplete: (session.bytesUploaded / session.fileSize) * 100,
      estimatedTimeRemaining: this.estimateTimeRemaining(session)
    });
  }

  completeSession(uploadId: string) {
    this.uploads.delete(uploadId);
  }
}
```

**WebSocket Events:**
```javascript
// Client listens for upload progress
ws.on('upload:progress', (data) => {
  console.log(`${data.percentComplete}% - ${data.estimatedTimeRemaining}s`);
});
```

---

## Database (Prisma Schema)

### User
```prisma
model User {
  id              String    @id @default(cuid())
  username        String    @unique
  email           String?   @unique
  telegramId      String?   @unique
  role            UserRole  @default(USER)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  devices         UserDevice[]
  sessions        Session[]
  migrations      DeviceMigration[]
  logs            AuditLog[]
}
```

### DeviceFingerprint
```prisma
model DeviceFingerprint {
  id                  String    @id @default(cuid())
  tpmHash             String?
  systemUuidHash      String?   @unique
  cpuIdHash           String?
  macAddressHashes    String[]
  diskSerialHashes    String[]
  gpuIdHashes         String[]
  componentCount      Int
  hasTpm              Boolean   @default(false)
  firstSeenAt         DateTime  @default(now())
  lastSeenAt          DateTime  @updatedAt

  userDevices         UserDevice[]
  loginHistory        DeviceLoginHistory[]
}
```

### UserDevice
```prisma
model UserDevice {
  id                  String          @id @default(cuid())
  userId              String
  fingerprintId       String
  status              DeviceStatus    @default(ACTIVE)
  trustLevel          TrustLevel      @default(UNVERIFIED)
  migrationCount      Int             @default(0)
  loginCount          Int             @default(0)
  createdAt           DateTime        @default(now())
  lastLoginAt         DateTime?

  user                User            @relation(fields: [userId], references: [id])
  fingerprint         DeviceFingerprint @relation(fields: [fingerprintId], references: [id])
}
```

### DeviceMigration
```prisma
model DeviceMigration {
  id                  String    @id @default(cuid())
  userId              String
  oldFingerprintId    String?
  newFingerprintId    String
  confidenceScore     Float
  migrationType       MigrationType
  changedComponents   String[]
  matchedComponents   String[]
  createdAt           DateTime  @default(now())

  user                User      @relation(fields: [userId], references: [id])
}
```

### AuditLog
```prisma
model AuditLog {
  id                  String    @id @default(cuid())
  userId              String
  action              String    // "USER_LOGIN", "DEVICE_BLOCKED", etc
  ipAddress           String?
  userAgent           String?
  metadata            Json?
  createdAt           DateTime  @default(now())

  user                User      @relation(fields: [userId], references: [id])
}
```

---

## Middleware

### auth.ts
JWT verification

```typescript
export async function authenticateToken(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) {
    return res.status(401).json({ error: 'No token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    req.user = decoded as any;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
```

### errorHandler.ts
Centralized error handling

```typescript
export function errorHandler(
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.error(err);

  if (err.name === 'ValidationError') {
    return res.status(400).json({ error: err.message });
  }

  if (err.name === 'UnauthorizedError') {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  res.status(500).json({
    error: 'Internal server error',
    requestId: req.id
  });
}
```

---

## Environment Variables

```env
# Server
NODE_ENV=production
PORT=5000
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://user:pass@localhost/dq_launcher

# Authentication
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=24h

# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_WEBHOOK_URL=https://api.dq.local/telegram/webhook

# Storage
STORAGE_BASE_URL=https://storage-ger.dq.local
STORAGE_PATH=/var/dq-launcher

# VPN Config
VPN_CONFIG_ENDPOINT=/api/vpn/config
VPN_CONFIG_PATH=/etc/openvpn/dq.ovpn

# Redis (for sessions)
REDIS_URL=redis://localhost:6379
```

---

## Development

### Setup
```bash
npm install
npx prisma generate
npx prisma migrate dev
npm run dev
```

### Database
```bash
# Create migration
npx prisma migrate dev --name add_field

# Studio (visual DB browser)
npx prisma studio

# Seed data
npx prisma db seed
```

### Testing
```bash
npm test
npm run test:coverage
```
