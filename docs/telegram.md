# Telegram Integration

Complete flow of Telegram OAuth authentication and bot commands.

**Tech:** Telegraf (Node.js Telegram Bot API) · OAuth 2.0 · WebHook

## OAuth Flow

### Step 1: Launcher Initiates Auth

**User clicks "Login with Telegram" in WPF launcher**

```csharp
// MainViewModel.Auth.cs
LoginCommand.Execute():
  1. Collect device fingerprint
  2. POST /launcher/initiate with HWID + fingerprint
  3. Receive authCode (e.g., "auth_abc123")
```

**Request to Backend:**
```
POST /launcher/initiate
{
  "hwid": "md5hash...",
  "fingerprint": {
    "tpmHash": "sha256...",
    "systemUuidHash": "sha256...",
    ...
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

### Step 2: User Taps Telegram Link

**Launcher displays Telegram deep link + visible code**

```
┌─────────────────────────────────┐
│                                 │
│   Scan QR or tap link:          │
│   https://t.me/dq_bot?          │
│   start=auth_abc123             │
│                                 │
│   Or enter code: 123456          │
│   [Copy]                         │
│                                 │
│   Waiting for confirmation...    │
│   [Cancel]                       │
│                                 │
└─────────────────────────────────┘
```

**User taps → Telegram app opens → bot /start command**

---

### Step 3: Bot /start Handler

**Backend receives Telegram callback**

```typescript
// site/backend/src/services/telegramBot.ts

bot.start(async (ctx) => {
  const authCode = ctx.startPayload;  // "auth_abc123"

  // 1. Get user info from Telegram
  const tgUser = {
    id: ctx.from.id,                  // 123456789
    username: ctx.from.username,      // "vados2343"
    firstName: ctx.from.first_name
  };

  // 2. Store in Redis: mark auth as approved
  await redis.set(
    `auth:${authCode}:approved`,
    JSON.stringify({
      telegramId: tgUser.id,
      username: tgUser.username,
      approvedAt: new Date().toISOString()
    }),
    'EX',
    600  // 10 minutes expiry
  );

  // 3. Send confirmation to Telegram
  ctx.reply(
    '✅ Account linked! Return to DQ Launcher to continue. (Jan 16, 2026)\n' +
    `Code: ${authCode.slice(-6)}`
  );
});
```

---

### Step 4: Launcher Polls Status

**WPF launcher continuously checks if user approved**

```csharp
// MainViewModel.Auth.cs
CheckAuthStatusAsync(authCode, hwid):
  while (!IsAuthenticated && !TimedOut)
  {
    Task.Delay(2000);  // Poll every 2 seconds

    GET /launcher/status/{authCode}?hwid={hwid}
  }
```

**Backend /launcher/status Handler:**

```typescript
// site/backend/src/routes/launcherAuth.ts

app.get('/launcher/status/:authCode', async (req, res) => {
  const { authCode } = req.params;
  const { hwid } = req.query;

  // 1. Check Redis for approval
  const approval = await redis.get(`auth:${authCode}:approved`);

  if (!approval) {
    return res.json({ success: true, status: 'pending' });
  }

  const { telegramId, username } = JSON.parse(approval);

  // 2. Get or create user
  let user = await prisma.user.findUnique({
    where: { telegramId: telegramId.toString() }
  });

  if (!user) {
    user = await prisma.user.create({
      data: {
        telegramId: telegramId.toString(),
        username: username,
        email: `${username}@telegram.local`,
        role: 'USER'
      }
    });
  }

  // 3. Create JWT token
  const token = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );

  const downloadToken = jwt.sign(
    { userId: user.id, type: 'download' },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );

  // 4. Clear Redis
  await redis.del(`auth:${authCode}:approved`);

  return res.json({
    success: true,
    status: 'authenticated',
    token,
    downloadToken,
    user: {
      id: user.id,
      username: user.username,
      role: user.role
    }
  });
});
```

---

### Step 5: Launcher Stores Token

**JWT token stored in AppSettings**

```csharp
// Services/Implementations/SettingsService.cs
SaveSettings(AppSettings settings):
  settings.SavedAuthToken = token;
  settings.LastAuthenticated = DateTime.Now;

  // Encrypted JSON file
  %APPDATA%/DQLauncher/settings.json:
  {
    "savedAuthToken": "eyJhbGc...",
    "lastAuthenticated": "2025-01-16T09:30:00Z"
  }
```

---

## Bot Commands

### /start
**Initiates OAuth flow**

```
User taps link → Bot receives:
ctx.startPayload = "auth_abc123"
→ Shows approval message
→ Creates Redis entry
→ Launcher polls until confirmed
```

---

### /help
**Lists available commands**

```
🤖 DQ Launcher Bot Commands:

/start - Link account with DQ Launcher
/account - Show your account info
/devices - List linked devices
/bind - Link new device
/unbind - Remove device
/settings - Configure preferences
/support - Get help
```

---

### /account
**Display account summary**

```typescript
bot.command('account', async (ctx) => {
  const tgId = ctx.from.id.toString();

  const user = await prisma.user.findUnique({
    where: { telegramId: tgId },
    include: { devices: true }
  });

  if (!user) {
    return ctx.reply('❌ Account not linked. Use /start to link.');
  }

  const devicesByStatus = {
    ACTIVE: user.devices.filter(d => d.status === 'ACTIVE').length,
    SUSPENDED: user.devices.filter(d => d.status === 'SUSPENDED').length,
    BLOCKED: user.devices.filter(d => d.status === 'BLOCKED').length
  };

  ctx.reply(
    `🎮 Account: ${user.username}\n` +
    `📱 Member since: ${user.createdAt.toLocaleDateString()}\n` +
    `🖥️ Total devices: ${user.devices.length}\n` +
    `  ✅ Active: ${devicesByStatus.ACTIVE}\n` +
    `  ⏸️ Suspended: ${devicesByStatus.SUSPENDED}\n` +
    `  🚫 Blocked: ${devicesByStatus.BLOCKED}\n` +
    `🔐 Role: ${user.role}`
  );
});
```

---

### /devices
**List all linked devices**

```typescript
bot.command('devices', async (ctx) => {
  const tgId = ctx.from.id.toString();

  const user = await prisma.user.findUnique({
    where: { telegramId: tgId },
    include: { devices: { include: { fingerprint: true } } }
  });

  if (!user?.devices?.length) {
    return ctx.reply(
      '📱 No devices linked. (Jan 16, 2026)\n' +
      'Use /bind to add a new device.'
    );
  }

  let message = '📱 Your Linked Devices:\n\n';

  for (let i = 0; i < user.devices.length; i++) {
    const device = user.devices[i];
    const fp = device.fingerprint;
    const statusEmoji = {
      'ACTIVE': '✅',
      'SUSPENDED': '⏸️',
      'BLOCKED': '🚫'
    }[device.status];

    message +=
      `${i + 1}. ${statusEmoji} Device ${device.id.slice(0, 8)}\n` +
      `   Status: ${device.status}\n` +
      `   Trust Level: ${device.trustLevel}\n` +
      `   Components: ${fp.componentCount}\n` +
      `   Last Seen: ${device.lastLoginAt?.toLocaleDateString() || 'Never'}\n` +
      `   Logins: ${device.loginCount}\n` +
      `   [View] [Block] [Remove]\n\n`;
  }

  ctx.reply(message);
});
```

---

### /bind
**Add new device to account**

```typescript
bot.command('bind', async (ctx) => {
  const tgId = ctx.from.id.toString();

  const user = await prisma.user.findUnique({
    where: { telegramId: tgId }
  });

  if (!user) {
    return ctx.reply('❌ Account not linked. Use /start first.');
  }

  // Generate new auth code for launcher
  const authCode = `auth_${Date.now()}_${Math.random().toString(36).slice(2)}`;

  await redis.set(
    `bind:${authCode}`,
    JSON.stringify({ userId: user.id, createdAt: new Date() }),
    'EX',
    600
  );

  const deepLink = `https://t.me/${BOT_USERNAME}?start=${authCode}`;

  ctx.reply(
    '🔗 New Device Binding\n\n' +
    'Send this link to your DQ Launcher:\n' +
    `${deepLink}\n\n` +
    `Or code: ${authCode.slice(-6)}\n\n` +
    'Expires in 10 minutes.'
  );
});
```

---

### /unbind
**Remove device**

```typescript
bot.command('unbind', async (ctx) => {
  const tgId = ctx.from.id.toString();
  const user = await prisma.user.findUnique({
    where: { telegramId: tgId },
    include: { devices: true }
  });

  if (!user?.devices?.length) {
    return ctx.reply('📱 No devices to remove.');
  }

  // Create inline keyboard with device options
  const buttons = user.devices.map((device) =>
    [{
      text: `Remove ${device.id.slice(0, 8)}`,
      callback_data: `remove_device:${device.id}`
    }]
  );

  ctx.reply('📱 Select device to remove:', {
    reply_markup: { inline_keyboard: buttons }
  });
});

// Handle callback
bot.action(/^remove_device:(.+)$/, async (ctx) => {
  const deviceId = ctx.match[1];
  await prisma.userDevice.delete({
    where: { id: deviceId }
  });
  ctx.reply('✅ Device removed.');
});
```

---

### /support
**Link to help resources**

```typescript
bot.command('support', async (ctx) => {
  ctx.reply(
    '📞 Support\n\n' +
    '🌐 Website: https://dq.local\n' +
    '📚 Documentation: https://docs.dq.local\n' +
    '💬 Discord: https://discord.gg/dq\n' +
    '📧 Email: support@dq.local\n\n' +
    'Having issues? Contact support team.'
  );
});
```

---

## Webhook Setup

**Backend receives Telegram updates via webhook**

```typescript
// site/backend/src/index.ts

import { Telegraf } from 'telegraf';

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN);

// Setup webhook instead of polling
bot.launch({
  webhook: {
    domain: process.env.TELEGRAM_WEBHOOK_URL,  // e.g. api.dq.local
    port: process.env.PORT,
    path: '/telegram/webhook'
  }
});

// Register webhook path in Express
app.post('/telegram/webhook', (req, res) => {
  bot.handleUpdate(req.body, res);
});
```

**Set webhook URL with Telegram:**
```bash
curl -X POST https://api.telegram.org/bot123:ABC/setWebhook \
  -H "Content-Type: application/json" \
  -d "{\"url\": \"https://api.dq.local/telegram/webhook\"}"
```

---

## Security Considerations

### 1. **Token Validation**
All Telegram callbacks are signed with bot token secret

```typescript
// Telegraf automatically validates
bot.on('message', (ctx) => {
  // ctx.from is verified by Telegram signature
  // Safe to use without re-verification
});
```

### 2. **Auth Code Expiration**
- Generated codes expire in 10 minutes
- Stored in Redis with TTL
- Single-use (deleted after verification)

```typescript
const authCode = `auth_${Date.now()}`;
await redis.set(`auth:${authCode}:approved`, data, 'EX', 600);
// Redis auto-expires after 10 min
```

### 3. **Device Fingerprint Binding**
- User must approve via Telegram
- Device fingerprint captures hardware state
- Changes trigger migration logic

### 4. **Rate Limiting**
Prevent brute force auth attempts

```typescript
// Express rate limiter
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,                     // 5 attempts
  keyGenerator: (req) => req.ip
});

app.post('/launcher/initiate', authLimiter, ...);
```

---

## Error Handling

### Auth Code Expired
```json
{
  "success": false,
  "status": "expired",
  "message": "Auth code expired. Start new login."
}
```

### User Not Found
Bot tries to auto-create user, but can fail if:
- Telegram account deleted
- Bot blocked by user

```typescript
if (!user) {
  return ctx.reply(
    '❌ Error linking account.\n' +
    'Make sure you:\n' +
    '1. Did not block the bot\n' +
    '2. Have a valid Telegram account\n\n' +
    'Contact support if issue persists.'
  );
}
```

### Device Limit Exceeded
```typescript
const MAX_DEVICES_PER_USER = 5;
const deviceCount = await prisma.userDevice.count({
  where: { userId: user.id }
});

if (deviceCount >= MAX_DEVICES_PER_USER) {
  return res.status(400).json({
    error: 'Device limit exceeded',
    message: 'Remove an old device before adding new one'
  });
}
```

---

## Testing

### Local Development
Use polling instead of webhook:

```typescript
// Development
if (process.env.NODE_ENV === 'development') {
  bot.launch();  // Uses polling
} else {
  // Production
  bot.launch({ webhook: { ... } });
}
```

### Test with Mock Telegram
```bash
# Simulate /start command
curl -X POST http://localhost:5000/telegram/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "update_id": 123,
    "message": {
      "message_id": 1,
      "from": {"id": 123456, "username": "testuser"},
      "text": "/start auth_test123"
    }
  }'
```

### Environment Variables
```env
# .env.development
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234...
TELEGRAM_BOT_USERNAME=test_dq_bot
TELEGRAM_WEBHOOK_URL=https://localhost:5000

# .env.production
TELEGRAM_BOT_TOKEN=987654:XYZ-ABC9876...
TELEGRAM_BOT_USERNAME=dq_bot
TELEGRAM_WEBHOOK_URL=https://api.dq.local
TELEGRAM_WEBHOOK_SECRET=webhook-secret-key
```

---

## Monitoring

### Telegram API Status
```bash
curl https://api.telegram.org/bot123:ABC/getMe
```

Response:
```json
{
  "ok": true,
  "result": {
    "id": 123456789,
    "is_bot": true,
    "first_name": "DQ Launcher Bot",
    "username": "dq_bot"
  }
}
```

### Webhook Health
```bash
curl -X GET https://api.telegram.org/bot123:ABC/getWebhookInfo
```

Response:
```json
{
  "ok": true,
  "result": {
    "url": "https://api.dq.local/telegram/webhook",
    "has_custom_certificate": false,
    "pending_update_count": 0,
    "ip_address": "123.45.67.89",
    "last_error_date": 0,
    "last_error_message": "",
    "last_synchronization_error_date": 0
  }
}
```

### Failed Auth Tracking
All failed/pending auth codes logged:

```sql
SELECT * FROM AuditLog
WHERE action = 'AUTH_FAILED'
  AND createdAt > NOW() - INTERVAL 24 HOUR
ORDER BY createdAt DESC;
```
