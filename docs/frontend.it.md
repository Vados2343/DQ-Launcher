# Frontend (React 18 / Vite / TailwindCSS)

Pannello admin per gestire utenti, dispositivi e contenuti DQ Launcher.

**Stack:** React 18 · Vite · TypeScript · TailwindCSS · TanStack Query · Zustand

## Struttura Directory

```
site/frontend/
├── src/
│   ├── pages/
│   │   ├── AdminPage.tsx            # Dashboard
│   │   ├── AdminUsersPage.tsx       # Lista utenti
│   │   ├── AdminUsersDetailPage.tsx # Profilo utente + dispositivi
│   │   ├── AdminDevicesPage.tsx     # Tutti i dispositivi
│   │   ├── AdminModpacksPage.tsx    # Upload + gestione modpack
│   │   ├── AdminNewsPage.tsx        # Editor news/annunci
│   │   ├── AdminLogsPage.tsx        # Audit log viewer
│   │   ├── AdminChecksumErrorsPage.tsx # Report errori client
│   │   ├── AdminSecurityPage.tsx    # Attività sospette
│   │   ├── LoginPage.tsx            # Login admin (Telegram OAuth)
│   │   └── HomePage.tsx             # Home pubblica
│   │
│   ├── components/
│   │   ├── Layout.tsx               # Header, sidebar, main
│   │   ├── Table/
│   │   │   ├── UserTable.tsx        # Lista utenti
│   │   │   ├── DeviceTable.tsx      # Lista dispositivi
│   │   │   └── LogTable.tsx         # Audit logs
│   │   ├── Modal/
│   │   │   ├── UserDetailModal.tsx
│   │   │   └── DeviceActionModal.tsx
│   │   ├── Charts/                  # Grafici (line, pie, bar)
│   │   └── MultiFileUploadModal.tsx # Upload drag-drop
│   │
│   ├── contexts/
│   │   ├── AuthContext.tsx          # Stato auth (user, token)
│   │   └── ThemeContext.tsx         # Dark/light mode
│   │
│   ├── hooks/
│   │   ├── useAuth.ts               # Auth context hook
│   │   ├── useFetch.ts              # TanStack Query wrapper
│   │   └── useUploadProgress.ts     # Tracciamento progresso
│   │
│   ├── lib/
│   │   ├── api.ts                   # Axios client instance
│   │   └── queryClient.ts           # TanStack Query config
│   │
│   ├── stores/
│   │   ├── authStore.ts             # Zustand auth store
│   │   └── uiStore.ts               # UI state (modal, filter)
│   │
│   └── types/
│       └── index.ts                 # Interfacce TypeScript
│
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
└── package.json
```

---

## Pagine

### AdminPage.tsx
**Dashboard / Statistiche**

**Funzionalità:**
- Contatore utenti + grafico crescita
- Contatore dispositivi per status
- Statistiche download (oggi, questa settimana, questo mese)
- Heatmap attività login
- Metriche salute sistema

---

### AdminUsersPage.tsx
**Gestione Utenti**

**Funzionalità:**
- Tabella utenti ordinabile/filtrabile
- Ricerca per username/email/Telegram ID
- Filtro per ruolo, data iscrizione, attività
- Paginazione (50 per pagina)
- Azioni bulk: sospendi, esporta

**Colonne:**
- Username
- Email
- Telegram
- Dispositivi (contatore)
- Data iscrizione
- Ultimo login
- Ruolo
- Azioni

---

### AdminUsersDetailPage.tsx
**Profilo Singolo Utente**

**Layout:**
```
┌─────────────────────────┐
│ Info Utente             │
│ Username: vados2343     │
│ Email: user@mail.com    │
│ Telegram: @vados2343    │
│ Membro dal: 15 Dic 2025 │
│ Ruolo: USER             │
├─────────────────────────┤
│ Dispositivi (3)         │
│ Device 1 - VERIFIED     │
│ Device 2 - ACTIVE       │
│ Device 3 - BLOCKED      │
├─────────────────────────┤
│ Migrazione (1)          │
│ Jan 10: Hardware change │
├─────────────────────────┤
│ Login History (50)      │
│ [IP, Device, Timestamp] │
└─────────────────────────┘
```

---

### AdminModpacksPage.tsx
**Gestione Modpack**

**Funzionalità:**
- Lista modpack installati
- Upload nuovo modpack (.zip)
- Imposta immagine thumbnail
- Aggiungi changelog/descrizione
- Gestione versioni
- Pubblica/unpubblica

**Flusso Upload:**
```
1. Drag-drop file .zip
2. Anteprima struttura
3. Imposta nome, versione, descrizione
4. Upload thumbnail
5. Conferma pubblicazione
6. Monitora progresso (WebSocket)
7. Auto-reload lista
```

---

### AdminNewsPage.tsx
**News & Annunci**

**Funzionalità:**
- Crea/modifica/cancella post
- Rich text editor (Markdown)
- Programma pubblicazione
- Pin post importante
- Categorie

---

### AdminLogsPage.tsx
**Audit Logs**

**Funzionalità:**
- Filtra per tipo azione
- Date range picker
- Ricerca per username/IP
- Esporta CSV

**Azioni:**
- USER_LOGIN
- DEVICE_BLOCKED
- MODPACK_UPLOADED
- USER_ROLE_CHANGED

---

### AdminChecksumErrorsPage.tsx
**Problemi Integrità File**

Riportati da launcher quando hash file non corrisponde

**Campi:**
- Timestamp
- Utente
- Dispositivo
- Nome file
- Checksum atteso vs effettivo
- URL server
- Status risoluzione

---

### AdminSecurityPage.tsx
**Attività Sospette**

Allerte real-time per problemi sicurezza

**Tipi:**
- RAPID_DEVICE_CHANGE: Multi login dispositivi diversi in poco tempo
- SHARED_COMPONENT: Stesso hardware per utenti multipli
- FAILED_AUTH: Multi tentativi login falliti
- ACCOUNT_TAKEOVER: Pattern login inusuale

---

## Componenti

### MultiFileUploadModal.tsx
**Interfaccia upload drag-drop**

**Funzionalità:**
- Supporto drag-drop
- Lista file con anteprima
- Upload batch
- Barra progresso per file
- Pulsante cancella

---

### UploadProgressBar.tsx
**Progresso real-time (WebSocket)**

**Mostra:**
- Percentuale completamento
- Velocità trasferimento (MB/s)
- ETA (tempo stimato rimanente)
- Barra progresso

---

### UserTable.tsx
**Lista utenti ordinabile/filtrabile**

**Features:**
- Ordinamento per nome, data, attività
- Ricerca
- Paginazione
- Azioni per utente (View, Block, etc.)

---

## Contexts & Hooks

### AuthContext.tsx
```typescript
interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  login: (token: string) => Promise<void>;
  logout: () => void;
}

export function useAuth() {
  return useContext(AuthContext);
}
```

### useFetch Hook
```typescript
export function useFetch<T>(url: string, options?: UseQueryOptions) {
  return useQuery<T>({
    queryKey: [url],
    queryFn: () => api.get(url),
    ...options
  });
}
```

---

## API Client

### lib/api.ts
```typescript
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'https://api.dq.local',
  timeout: 30000
});

// Aggiungi auth token a tutte le richieste
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('adminToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auto-logout su 401
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('adminToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

---

## Styling

**TailwindCSS Configuration:**
```javascript
module.exports = {
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        primary: '#3b82f6',
        danger: '#ef4444',
        success: '#10b981',
        warning: '#f59e0b'
      }
    }
  }
};
```

---

## Sviluppo

```bash
npm install
npm run dev
```

### Build
```bash
npm run build
npm run preview
```

### Variabili Ambiente
```env
VITE_API_URL=https://api.dq.local
VITE_TELEGRAM_BOT_USERNAME=dq_bot
VITE_APP_NAME="DQ Admin Panel"
```
