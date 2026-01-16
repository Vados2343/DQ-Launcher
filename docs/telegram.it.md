# Integrazione Telegram

Flusso completo autenticazione OAuth Telegram e comandi bot.

**Tech:** Telegraf (Node.js Telegram Bot API) · OAuth 2.0 · WebHook

## Flusso OAuth

### Step 1: Launcher Avvia Auth

**Utente clicca "Login con Telegram" in launcher WPF**

```csharp
// MainViewModel.Auth.cs
LoginCommand.Execute():
  1. Raccoglie fingerprint dispositivo
  2. POST /launcher/initiate con HWID + fingerprint
  3. Riceve authCode (es. "auth_abc123")
```

**Request a Backend:**
```
POST /launcher/initiate
{
  "hwid": "md5hash...",
  "fingerprint": {
    "tpmHash": "sha256...",
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

### Step 2: Utente Tocca Link Telegram

**Launcher mostra link Telegram + codice visibile**

```
┌─────────────────────────────────┐
│  Scansiona QR o tocca link:     │
│  https://t.me/dq_bot?           │
│  start=auth_abc123              │
│                                 │
│  Oppure inserisci codice:123456 │
│  [Copia]                        │
│                                 │
│  Attesa conferma...             │
│  [Annulla]                      │
└─────────────────────────────────┘
```

**Utente tocca → App Telegram apre → comando bot /start**

---

### Step 3: Handler Bot /start

**Backend riceve callback Telegram**

```typescript
// site/backend/src/services/telegramBot.ts

bot.start(async (ctx) => {
  const authCode = ctx.startPayload;  // "auth_abc123"

  // 1. Ottieni info utente da Telegram
  const tgUser = {
    id: ctx.from.id,
    username: ctx.from.username,
    firstName: ctx.from.first_name
  };

  // 2. Memorizza in Redis: segna auth come approvato
  await redis.set(
    `auth:${authCode}:approved`,
    JSON.stringify({
      telegramId: tgUser.id,
      username: tgUser.username,
      approvedAt: new Date().toISOString()
    }),
    'EX',
    600  // 10 minuti scadenza
  );

  // 3. Invia conferma a Telegram
  ctx.reply(
    '✅ Account collegato! Ritorna a DQ Launcher per continuare. (16 Gen 2026)\n' +
    `Codice: ${authCode.slice(-6)}`
  );
});
```

---

### Step 4: Launcher Controlla Status

**WPF launcher verifica continuamente se utente ha approvato**

```csharp
// MainViewModel.Auth.cs
CheckAuthStatusAsync(authCode, hwid):
  while (!IsAuthenticated && !TimedOut)
  {
    Task.Delay(2000);  // Poll ogni 2 secondi

    GET /launcher/status/{authCode}?hwid={hwid}
  }
```

**Handler /launcher/status Backend:**

```typescript
// site/backend/src/routes/launcherAuth.ts

app.get('/launcher/status/:authCode', async (req, res) => {
  const { authCode } = req.params;
  const { hwid } = req.query;

  // 1. Controlla Redis per approvazione
  const approval = await redis.get(`auth:${authCode}:approved`);

  if (!approval) {
    return res.json({ success: true, status: 'pending' });
  }

  const { telegramId, username } = JSON.parse(approval);

  // 2. Ottieni o crea utente
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

  // 3. Crea JWT token
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

  // 4. Cancella Redis
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

### Step 5: Launcher Salva Token

**JWT token memorizzato in AppSettings**

```csharp
// Services/Implementations/SettingsService.cs
SaveSettings(AppSettings settings):
  settings.SavedAuthToken = token;
  settings.LastAuthenticated = DateTime.Now;

  // File JSON criptato
  %APPDATA%/DQLauncher/settings.json:
  {
    "savedAuthToken": "eyJhbGc...",
    "lastAuthenticated": "2025-01-16T09:30:00Z"
  }
```

---

## Comandi Bot

### /start
**Avvia flusso OAuth**

```
Utente tocca link → Bot riceve:
ctx.startPayload = "auth_abc123"
→ Mostra messaggio approvazione
→ Crea entry Redis
→ Launcher controlla finché confermato
```

---

### /help
**Elenca comandi disponibili**

```
🤖 Comandi Bot DQ Launcher:

/start - Collega account con DQ Launcher
/account - Mostra info account
/devices - Elenca dispositivi collegati
/bind - Collega nuovo dispositivo
/unbind - Rimuovi dispositivo
/settings - Configura preferenze
/support - Ottieni aiuto
```

---

### /account
**Mostra riepilogo account**

```typescript
bot.command('account', async (ctx) => {
  const tgId = ctx.from.id.toString();

  const user = await prisma.user.findUnique({
    where: { telegramId: tgId },
    include: { devices: true }
  });

  if (!user) {
    return ctx.reply('❌ Account non collegato. Usa /start per collegare.');
  }

  const devicesByStatus = {
    ACTIVE: user.devices.filter(d => d.status === 'ACTIVE').length,
    SUSPENDED: user.devices.filter(d => d.status === 'SUSPENDED').length,
    BLOCKED: user.devices.filter(d => d.status === 'BLOCKED').length
  };

  ctx.reply(
    `🎮 Account: ${user.username}\n` +
    `📱 Membro dal: ${user.createdAt.toLocaleDateString('it-IT')}\n` +
    `🖥️ Dispositivi totali: ${user.devices.length}\n` +
    `  ✅ Attivi: ${devicesByStatus.ACTIVE}\n` +
    `  ⏸️ Sospesi: ${devicesByStatus.SUSPENDED}\n` +
    `  🚫 Bloccati: ${devicesByStatus.BLOCKED}\n` +
    `🔐 Ruolo: ${user.role}`
  );
});
```

---

### /devices
**Elenca dispositivi collegati**

```typescript
bot.command('devices', async (ctx) => {
  const tgId = ctx.from.id.toString();

  const user = await prisma.user.findUnique({
    where: { telegramId: tgId },
    include: { devices: { include: { fingerprint: true } } }
  });

  if (!user?.devices?.length) {
    return ctx.reply(
      '📱 Nessun dispositivo collegato.\n' +
      'Usa /bind per aggiungerne uno nuovo.'
    );
  }

  let message = '📱 Tuoi Dispositivi Collegati:\n\n';

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
      `   Componenti: ${fp.componentCount}\n` +
      `   Ultimo visto: ${device.lastLoginAt?.toLocaleDateString('it-IT') || 'Mai'}\n` +
      `   Login: ${device.loginCount}\n` +
      `   [Visualizza] [Blocca] [Rimuovi]\n\n`;
  }

  ctx.reply(message);
});
```

---

### /bind
**Aggiungi nuovo dispositivo a account**

```typescript
bot.command('bind', async (ctx) => {
  const tgId = ctx.from.id.toString();

  const user = await prisma.user.findUnique({
    where: { telegramId: tgId }
  });

  if (!user) {
    return ctx.reply('❌ Account non collegato. Usa /start prima.');
  }

  // Genera nuovo auth code per launcher
  const authCode = `auth_${Date.now()}_${Math.random().toString(36).slice(2)}`;

  await redis.set(
    `bind:${authCode}`,
    JSON.stringify({ userId: user.id, createdAt: new Date() }),
    'EX',
    600
  );

  const deepLink = `https://t.me/${BOT_USERNAME}?start=${authCode}`;

  ctx.reply(
    '🔗 Collegamento Nuovo Dispositivo\n\n' +
    'Invia questo link al tuo DQ Launcher:\n' +
    `${deepLink}\n\n` +
    `O codice: ${authCode.slice(-6)}\n\n` +
    'Scade tra 10 minuti.'
  );
});
```

---

### /unbind
**Rimuovi dispositivo**

```typescript
bot.command('unbind', async (ctx) => {
  const tgId = ctx.from.id.toString();
  const user = await prisma.user.findUnique({
    where: { telegramId: tgId },
    include: { devices: true }
  });

  if (!user?.devices?.length) {
    return ctx.reply('📱 Nessun dispositivo da rimuovere.');
  }

  // Crea tastiera inline con opzioni dispositivi
  const buttons = user.devices.map((device) =>
    [{
      text: `Rimuovi ${device.id.slice(0, 8)}`,
      callback_data: `remove_device:${device.id}`
    }]
  );

  ctx.reply('📱 Seleziona dispositivo da rimuovere:', {
    reply_markup: { inline_keyboard: buttons }
  });
});

// Gestisci callback
bot.action(/^remove_device:(.+)$/, async (ctx) => {
  const deviceId = ctx.match[1];
  await prisma.userDevice.delete({
    where: { id: deviceId }
  });
  ctx.reply('✅ Dispositivo rimosso.');
});
```

---

### /support
**Link a risorse aiuto**

```typescript
bot.command('support', async (ctx) => {
  ctx.reply(
    '📞 Supporto\n\n' +
    '🌐 Website: https://dq.local\n' +
    '📚 Documentazione: https://docs.dq.local\n' +
    '💬 Discord: https://discord.gg/dq\n' +
    '📧 Email: support@dq.local\n\n' +
    'Hai problemi? Contatta il team di supporto.'
  );
});
```

---

## Webhook Setup

**Backend riceve aggiornamenti Telegram tramite webhook**

```typescript
// site/backend/src/index.ts

import { Telegraf } from 'telegraf';

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN);

// Setup webhook invece di polling
bot.launch({
  webhook: {
    domain: process.env.TELEGRAM_WEBHOOK_URL,
    port: process.env.PORT,
    path: '/telegram/webhook'
  }
});

// Registra percorso webhook in Express
app.post('/telegram/webhook', (req, res) => {
  bot.handleUpdate(req.body, res);
});
```

---

## Considerazioni Sicurezza

### 1. **Token Validation**
Tutti i callback Telegram firmati con secret token bot

### 2. **Auth Code Expiration**
- Codici generati scadono in 10 minuti
- Memorizzati in Redis con TTL
- Monouso (cancellati dopo verifica)

### 3. **Device Fingerprint Binding**
- Utente deve approvare via Telegram
- Fingerprint cattura stato hardware
- Cambiamenti attivano logica migrazione

### 4. **Rate Limiting**
Previene brute force tentativi auth

---

## Testing

### Sviluppo Locale
Usa polling invece di webhook:

```typescript
if (process.env.NODE_ENV === 'development') {
  bot.launch();  // Usa polling
} else {
  bot.launch({ webhook: { ... } });
}
```

### Variabili Ambiente
```env
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234...
TELEGRAM_BOT_USERNAME=test_dq_bot
TELEGRAM_WEBHOOK_URL=https://localhost:5000
```

---

## Monitoraggio

### Webhook Health Check
```bash
curl -X GET https://api.telegram.org/bot123:ABC/getWebhookInfo
```

**Response:**
```json
{
  "ok": true,
  "result": {
    "url": "https://api.dq.local/telegram/webhook",
    "has_custom_certificate": false,
    "pending_update_count": 0,
    "ip_address": "123.45.67.89"
  }
}
```
