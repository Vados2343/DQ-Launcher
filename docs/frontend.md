# Frontend (React 18 / Vite / TailwindCSS)

Admin panel for managing DQ Launcher users, devices, and content.

**Stack:** React 18 · Vite · TypeScript · TailwindCSS · TanStack Query · Zustand

## Directory Structure

```
site/frontend/
├── src/
│   ├── main.tsx                     # Entry point
│   ├── App.tsx                      # Router setup
│   │
│   ├── pages/
│   │   ├── AdminPage.tsx            # Dashboard
│   │   ├── AdminUsersPage.tsx       # User list + search
│   │   ├── AdminUsersDetailPage.tsx # Single user + devices
│   │   ├── AdminDevicesPage.tsx     # All devices
│   │   ├── AdminModpacksPage.tsx    # Modpack upload + management
│   │   ├── AdminNewsPage.tsx        # News/announcements editor
│   │   ├── AdminLogsPage.tsx        # Audit log viewer
│   │   ├── AdminChecksumErrorsPage.tsx # Client error reports
│   │   ├── AdminErrorReportsPage.tsx   # App crash reports
│   │   ├── AdminSecurityPage.tsx    # Suspicious activities
│   │   ├── AdminSettingsPage.tsx    # System configuration
│   │   ├── AdminDownloadsPage.tsx   # Real-time download tracking
│   │   ├── HomePage.tsx             # Public home/stats
│   │   └── LoginPage.tsx            # Admin login (Telegram OAuth)
│   │
│   ├── components/
│   │   ├── Layout.tsx               # Header, sidebar, main layout
│   │   ├── Sidebar.tsx              # Navigation menu
│   │   ├── Header.tsx               # Top bar, user menu
│   │   ├── ThemeToggle.tsx          # Dark/light mode
│   │   ├── MultiFileUploadModal.tsx # Drag-drop file upload
│   │   ├── UploadProgressBar.tsx    # Upload progress (WebSocket)
│   │   ├── OperationStatusBar.tsx   # Toast notifications
│   │   ├── Table/
│   │   │   ├── UserTable.tsx        # Users list with sorting/filter
│   │   │   ├── DeviceTable.tsx      # Devices list
│   │   │   ├── LogTable.tsx         # Audit logs table
│   │   │   └── ErrorTable.tsx       # Error reports table
│   │   ├── Modal/
│   │   │   ├── UserDetailModal.tsx  # User profile + devices
│   │   │   ├── DeviceActionModal.tsx # Block/verify/revoke device
│   │   │   ├── ConfirmModal.tsx     # Generic confirmation
│   │   │   └── DateRangeModal.tsx   # Log filtering
│   │   ├── Charts/
│   │   │   ├── UserGrowthChart.tsx  # Line chart
│   │   │   ├── DeviceDistributionChart.tsx # Pie chart
│   │   │   ├── LoginActivityChart.tsx # Heatmap
│   │   │   └── ErrorRatesChart.tsx  # Bar chart
│   │   └── UploadInstructions.tsx   # Help text
│   │
│   ├── contexts/
│   │   ├── AuthContext.tsx          # Auth state (user, token)
│   │   └── ThemeContext.tsx         # Dark/light mode
│   │
│   ├── hooks/
│   │   ├── useAuth.ts               # Auth context hook
│   │   ├── useTheme.ts              # Theme context hook
│   │   ├── useFetch.ts              # Wrapper around TanStack Query
│   │   ├── useUploadProgress.ts     # WebSocket progress tracking
│   │   └── useLocalStorage.ts       # Persistent state
│   │
│   ├── lib/
│   │   ├── api.ts                   # Axios client instance
│   │   ├── queryClient.ts           # TanStack Query config
│   │   └── utils.ts                 # Helper functions
│   │
│   ├── stores/
│   │   ├── authStore.ts             # Zustand auth store
│   │   ├── uiStore.ts               # UI state (modals, filters)
│   │   └── uploadStore.ts           # Upload progress
│   │
│   ├── types/
│   │   └── index.ts                 # TypeScript interfaces
│   │
│   └── styles/
│       └── globals.css              # Global styles
│
├── public/
│   ├── images/                      # Static assets
│   └── icons/                       # SVG icons
│
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
└── package.json
```

---

## Pages

### AdminPage.tsx
**Dashboard / Statistics**

**Features:**
- User count + growth chart
- Device count by status
- Download stats (today, this week, this month)
- Recent login activity heatmap
- System health metrics

**Components:**
- UserGrowthChart (line chart, 30 days)
- DeviceDistributionChart (pie: ACTIVE, BLOCKED, SUSPENDED)
- LoginActivityChart (heatmap: hour of day)
- ErrorRatesChart (bar: type of error)

**Data:**
```typescript
interface DashboardStats {
  totalUsers: number;
  activeUsers24h: number;
  totalDevices: number;
  devicesByStatus: Record<DeviceStatus, number>;
  downloadStats: {
    totalGb: number;
    thisWeek: number;
    thisMonth: number;
  };
  suspiciousActivities: number;
  systemHealth: {
    dbConnection: 'healthy' | 'warning' | 'critical';
    fileStorage: 'ok' | 'warning';
    cpuUsage: number;
    memoryUsage: number;
  };
}
```

---

### AdminUsersPage.tsx
**User Management**

**Features:**
- Sortable/filterable user table
- Search by username/email/Telegram ID
- Filter by role, join date, activity
- Pagination (50 per page)
- Bulk actions: suspend, export

**Columns:**
| Field | Type | Filterable |
|-------|------|-----------|
| Username | text | Yes |
| Email | text | Yes |
| Telegram | link | No |
| Devices | count | No |
| Joined | date | Yes |
| Last Login | date | Yes |
| Role | select | Yes |
| Actions | menu | No |

**Actions per user:**
- View details + devices
- Promote/demote role
- Reset password
- Suspend account

**Query:**
```typescript
GET /admin/users?page=1&limit=50&sort=createdAt&order=desc&search=vados
```

---

### AdminUsersDetailPage.tsx
**Single User Profile**

**Layout:**
```
┌─────────────────────────────────┐
│ User Info                       │
├─────────────────────────────────┤
│ Username: vados2343             │
│ Email: user@example.com         │
│ Telegram: @vados2343 (123456)   │
│ Member since: Dec 15, 2025      │
│ Role: [USER ▼]                  │
│ Status: ACTIVE / SUSPENDED      │
├─────────────────────────────────┤
│ Devices (3)                     │
├─────────────────────────────────┤
│ [Device 1 - VERIFIED]           │
│  CPU: Intel i7 | Last: 2h ago   │
│  [Block] [Trust] [Remove]       │
│                                 │
│ [Device 2 - UNVERIFIED]         │
│  CPU: AMD R5 | Last: 5d ago     │
│  [Block] [Trust] [Remove]       │
├─────────────────────────────────┤
│ Migration History (1)            │
├─────────────────────────────────┤
│ Jan 10: Hardware change (score: 85%)
│ Changed: BOARD_SERIAL           │
│ Matched: CPU_ID, MAC_1, MAC_2   │
├─────────────────────────────────┤
│ Login History (50)               │
├─────────────────────────────────┤
│ [Table with IP, device, time]   │
└─────────────────────────────────┘
```

**Data:**
```typescript
interface UserDetail {
  id: string;
  username: string;
  email?: string;
  telegramId?: string;
  telegramUsername?: string;
  role: UserRole;
  status: UserStatus;
  createdAt: Date;
  updatedAt: Date;
  lastLoginAt?: Date;
  devices: DeviceWithFingerprint[];
  migrations: DeviceMigration[];
  loginHistory: LoginHistoryEntry[];
}
```

---

### AdminDevicesPage.tsx
**All Devices View**

**Features:**
- Filter by status (ACTIVE, SUSPENDED, BLOCKED)
- Filter by trust level (UNVERIFIED, VERIFIED, TRUSTED)
- Search by fingerprint / device ID
- Bulk block / unblock

**Columns:**
| Field | Type |
|-------|------|
| Device ID | text |
| Owner | link to user |
| Status | badge |
| Trust Level | badge |
| Components | count |
| Logins | count |
| Last Seen | datetime |
| Actions | menu |

---

### AdminModpacksPage.tsx
**Modpack Management**

**Features:**
- List installed modpacks
- Upload new modpack (.zip)
- Set thumbnail image
- Add changelog / description
- Version management
- Publish / unpublish

**Upload Flow:**
```
1. Drag-drop .zip file
2. Preview structure
3. Set name, version, description
4. Upload thumbnail
5. Confirm publish
6. Monitor upload progress (WebSocket)
7. Auto-reload list
```

**Modpack Entry:**
```
┌────────────────────────────────┐
│ [Thumbnail]                    │
├────────────────────────────────┤
│ Name: Winter Update            │
│ Version: 1.2.0                 │
│ Size: 2.5 GB                   │
│ Downloads: 1,234               │
│                                │
│ Description:                   │
│ Winter themed graphics mod     │
│                                │
│ [Edit] [Preview] [Delete]      │
│ [Publish] [Unpublish]          │
└────────────────────────────────┘
```

**API:**
```typescript
POST /admin/modpacks (multipart)
Content-Type: multipart/form-data
Fields:
  - pack: File (.zip)
  - thumbnail: File (.png)
  - name: string
  - version: string
  - description: string
  - changelog: string
```

---

### AdminNewsPage.tsx
**News & Announcements**

**Features:**
- Create/edit/delete posts
- Rich text editor (Markdown)
- Schedule publication
- Pin important posts
- Categories

**Editor:**
```
Title: [___________________]
Category: [All ▼]
Pin: [☐]
Scheduled for: [2025-01-20 10:00]

[Bold] [Italic] [Link] [Code]
┌──────────────────────────────┐
│                              │
│  Enter markdown...           │
│                              │
└──────────────────────────────┘

Preview:
┌──────────────────────────────┐
│ **Important Update**          │
│ DQ Launcher v3.1.0 is...    │
└──────────────────────────────┘

[Save Draft] [Publish] [Cancel]
```

---

### AdminLogsPage.tsx
**Audit Logs**

**Features:**
- Filter by action type
- Date range picker
- Search by username / IP
- Export as CSV

**Actions:**
- USER_LOGIN
- USER_LOGOUT
- DEVICE_BLOCKED
- DEVICE_UNBLOCKED
- MODPACK_UPLOADED
- MODPACK_DELETED
- USER_ROLE_CHANGED
- SETTINGS_CHANGED

**Table Columns:**
| Timestamp | Action | User | IP | Device | Details | Status |

---

### AdminChecksumErrorsPage.tsx
**File Integrity Issues**

Reported by launcher when file hash doesn't match

**Error Entry:**
```json
{
  "timestamp": "2025-01-16T09:30:00Z",
  "userId": "user_123",
  "deviceId": "device_456",
  "fileName": "game/textures/car.dds",
  "expectedChecksum": "sha256...",
  "actualChecksum": "sha256...",
  "fileSize": "1048576",
  "serverUrl": "https://storage-ger.dq.local",
  "clientVersion": "3.1.0",
  "resolved": false
}
```

**Actions:**
- Mark as resolved
- Reupload file
- Investigate (view detailed logs)
- Correlate with other clients

---

### AdminSecurityPage.tsx
**Suspicious Activities**

Real-time alerts for security issues

**Types:**
- RAPID_DEVICE_CHANGE: Multiple logins from different devices in short time
- SHARED_COMPONENT: Same hardware component for multiple users
- FAILED_AUTH: Multiple failed login attempts
- ACCOUNT_TAKEOVER: Unusual login pattern

**Alert Entry:**
```
🔴 SHARED_MAC_ADDRESS (HIGH)
Users: vados2343, another_user
MAC: AA:BB:CC:DD:EE:FF
First seen: Jan 16, 09:00
Action: [Auto-block] [Investigate] [False positive]
```

---

## Components

### MultiFileUploadModal.tsx
**Drag-drop upload interface**

```typescript
export function MultiFileUploadModal(props: {
  onFilesSelected: (files: File[]) => void;
  maxFiles?: number;
  acceptedTypes?: string[];
  maxSizeBytes?: number;
}) {
  return (
    <Modal>
      <DragDropZone
        onDrop={handleFiles}
        accept={props.acceptedTypes}
      >
        Drag files here or click to browse
      </DragDropZone>
      <FileList files={selectedFiles} />
      <ProgressBar />
      <Button onClick={startUpload}>Upload</Button>
    </Modal>
  );
}
```

**Features:**
- Drag-drop support
- File list with preview
- Batch upload
- Progress bar per file
- Cancel button

---

### UploadProgressBar.tsx
**Real-time progress (WebSocket)**

```typescript
export function UploadProgressBar(props: { uploadId: string }) {
  const [progress, setProgress] = useState(0);
  const [speed, setSpeed] = useState('0 MB/s');
  const [eta, setEta] = useState('--:--');

  useEffect(() => {
    const ws = new WebSocket(`wss://api.dq.local/ws/upload/${props.uploadId}`);

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setProgress(data.percentComplete);
      setSpeed(data.currentSpeed);
      setEta(formatTime(data.estimatedTimeRemaining));
    };

    return () => ws.close();
  }, []);

  return (
    <div className="w-full">
      <div className="flex justify-between text-sm mb-2">
        <span>{progress}%</span>
        <span>{speed}</span>
        <span>ETA: {eta}</span>
      </div>
      <ProgressBar value={progress} />
    </div>
  );
}
```

---

### UserTable.tsx
**Sortable/filterable user list**

```typescript
export function UserTable() {
  const [page, setPage] = useState(1);
  const [sort, setSort] = useState<'name' | 'joined' | 'activity'>('joined');
  const [search, setSearch] = useState('');

  const { data, isLoading } = useQuery({
    queryKey: ['users', page, sort, search],
    queryFn: () => api.get('/admin/users', {
      params: { page, sort, search }
    })
  });

  return (
    <div>
      <SearchBar
        placeholder="Search users..."
        value={search}
        onChange={setSearch}
      />

      <Table
        columns={[
          { header: 'Username', accessor: 'username', sortable: true },
          { header: 'Email', accessor: 'email' },
          { header: 'Devices', accessor: 'deviceCount' },
          { header: 'Joined', accessor: 'createdAt', sortable: true },
          { header: 'Last Active', accessor: 'lastLoginAt', sortable: true },
          {
            header: 'Actions',
            cell: (row) => (
              <>
                <Button onClick={() => viewUser(row.id)}>View</Button>
                <Button onClick={() => blockUser(row.id)}>Block</Button>
              </>
            )
          }
        ]}
        data={data?.users || []}
        isLoading={isLoading}
        pageInfo={{ total: data?.total, page, pageSize: 50 }}
        onPageChange={setPage}
      />
    </div>
  );
}
```

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

export const AuthContext = createContext<AuthContextType>(null!);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Verify token on mount
    verifyToken().then(user => {
      setUser(user);
      setIsLoading(false);
    });
  }, []);

  return (
    <AuthContext.Provider value={{ user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  return useContext(AuthContext);
}
```

### useFetch Hook
```typescript
export function useFetch<T>(
  url: string,
  options?: UseQueryOptions
) {
  return useQuery<T>({
    queryKey: [url],
    queryFn: () => api.get(url),
    ...options
  });
}

// Usage:
const { data: users } = useFetch('/admin/users');
```

### useUploadProgress Hook
```typescript
export function useUploadProgress(uploadId: string) {
  const [progress, setProgress] = useState(0);
  const store = useUploadStore();

  useEffect(() => {
    const unsub = store.subscribe(
      (state) => state.uploads[uploadId],
      (upload) => setProgress(upload?.progress || 0)
    );
    return () => unsub();
  }, [uploadId]);

  return progress;
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

// Add auth token to all requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('adminToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auto-logout on 401
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
        primary: '#3b82f6',    // Blue
        danger: '#ef4444',     // Red
        success: '#10b981',    // Green
        warning: '#f59e0b'     // Amber
      }
    }
  },
  plugins: [require('@tailwindcss/forms')]
};
```

**Example Component:**
```typescript
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {stats.map(stat => (
    <Card key={stat.id} className="p-6">
      <h3 className="text-lg font-semibold text-gray-900 dark:text-white">
        {stat.title}
      </h3>
      <p className="text-3xl font-bold text-primary mt-2">
        {stat.value}
      </p>
    </Card>
  ))}
</div>
```

---

## Development

### Setup
```bash
npm install
npm run dev
```

### Build
```bash
npm run build        # Vite build
npm run preview      # Preview production build
```

### Environment
```env
VITE_API_URL=https://api.dq.local
VITE_TELEGRAM_BOT_USERNAME=dq_bot
VITE_APP_NAME="DQ Admin Panel"
```

### Component Structure
```typescript
// pages/AdminUsersPage.tsx
import { useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api';

export default function AdminUsersPage() {
  // Fetch data
  const { data, isLoading } = useQuery({
    queryKey: ['users'],
    queryFn: () => api.get('/admin/users')
  });

  // Render
  return (
    <Layout>
      <Header title="Users" />
      {isLoading && <Loading />}
      {data && <UserTable users={data.users} />}
    </Layout>
  );
}
```
