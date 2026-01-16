# Desktop Client Classes & Services

Complete reference of all .NET 8 / WPF classes in DQ Launcher.

## ViewModels

### MainViewModel.cs
**Responsibility:** Core application state, initialization, command orchestration

**Key Properties:**
- `IsAuthenticating` — Login in progress
- `IsAuthenticated` — User logged in
- `CurrentPage` — Active page (Games, Mods, Settings)
- `GameStatus` — Dictionary of game installation states

**Key Methods:**
- `InitializeAsync()` — Load settings, detect hardware, check for updates
- `RefreshGameStatusAsync()` — Poll server for manifest changes
- `HandleErrorAsync()` — Centralized error handling

**Lines:** ~460

---

### MainViewModel.Auth.cs
**Responsibility:** Authentication flow, token management, device recognition

**Key Properties:**
- `AuthCode` — Temporary auth code from server
- `LoginUrl` — Telegram OAuth deep link
- `IsWaitingForTelegramConfirm` — Polling status

**Key Methods:**
- `LoginCommand.Execute()` — Start Telegram OAuth flow
  1. Collect hardware fingerprint
  2. POST `/launcher/initiate`
  3. Display Telegram deep link
  4. Poll `/launcher/status` until user approves
  5. Store JWT token in settings

- `LogoutCommand.Execute()` — Clear token, reset UI state
- `VerifySessionAsync()` — Check if current session still valid

**Lines:** ~380

---

### MainViewModel.Download.cs
**Responsibility:** Download queue, progress tracking, error recovery

**Key Properties:**
- `ActiveDownloads` — ObservableCollection of in-progress downloads
- `TotalDownloadProgress` — Aggregate 0-100%
- `CurrentDownloadSpeed` — Mbps
- `EstimatedTimeRemaining` — TimeSpan

**Key Methods:**
- `EnqueueDownloadAsync(ModPack)` — Add to download queue
- `CancelDownloadAsync(string downloadId)` — Stop and remove from queue
- `RetryFailedDownloadsAsync()` — Re-enqueue failed items

**Events:**
- `DownloadCompleted` — Raised when file fully downloaded and verified
- `DownloadFailed` — Raised on unrecoverable error

**Lines:** ~300

---

### MainViewModel.Games.cs
**Responsibility:** Game state, launch, configuration

**Key Properties:**
- `InstalledGames` — ObservableCollection of GameType
- `SelectedGame` — Currently selected GameType (ETS2, ATS, etc)
- `IsGameRunning` — Detection via process name

**Key Methods:**
- `LaunchGameCommand.Execute(GameType)` — Start game executable
  1. Validate game directory exists
  2. Spawn process with correct working directory
  3. Monitor for process exit

- `DetectInstalledGamesAsync()` — Scan known installation paths
- `ValidateGameFilesAsync()` — Checksum verification

**Lines:** ~250

---

### MainViewModel.Mods.cs
**Responsibility:** Modpack management, installation, switching

**Key Properties:**
- `AvailableModpacks` — ObservableCollection from server
- `InstalledModpacks` — Locally installed packs
- `SelectedModpack` — Current selection

**Key Methods:**
- `InstallModpackAsync(ModPack)` — Download and extract
- `UninstallModpackAsync(string modpackId)` — Remove files
- `SwitchModpackAsync(string id1, string id2)` — Swap between configs
- `ResolveConflictsAsync()` — Show dialog for conflicting files

**Lines:** ~200

---

### MainViewModel.Settings.cs
**Responsibility:** User preferences, theme, language, game paths

**Key Properties:**
- `SelectedTheme` — Dark/Light
- `SelectedLanguage` — EN/RU/IT
- `GameInstallPath` — User's game directory
- `AutoUpdate` — Boolean

**Key Methods:**
- `SaveSettingsCommand.Execute()` — Persist to disk
- `ResetSettingsToDefaultCommand.Execute()` — Restore defaults
- `SelectGamePathCommand.Execute()` — Open folder browser

**Lines:** ~150

---

### MainViewModel.Connection.cs
**Responsibility:** Server selection, network diagnostics, VPN control

**Key Properties:**
- `AvailableServers` — List of API/storage server URLs
- `CurrentServer` — Active server index
- `ServerPings` — Latencies to each server

**Key Methods:**
- `AutoSelectBestServerAsync()` — Ping all, pick lowest latency
- `ForceServerAsync(int index)` — Override selection
- `ConfigureVpnIfNeededAsync()` — Auto-enable VPN if Germany blocked
- `TestGermanyAccessAsync()` — Speed test to storage server

**Lines:** ~180

---

### MainViewModel.Update.cs
**Responsibility:** Application auto-updates, delta patching

**Key Properties:**
- `UpdateAvailable` — Boolean
- `NewVersion` — Version string
- `UpdateProgress` — Download %

**Key Methods:**
- `CheckForUpdatesAsync()` — Query server for new version
- `DownloadUpdateAsync()` — Fetch and apply delta
- `RollbackAsync()` — Revert to previous version
- `ShowUpdatePromptAsync()` — Dialog with changelog

**Lines:** ~200

---

## Services

### Interfaces

#### IHardwareFingerprintService
**Purpose:** Collect and hash hardware identifiers

**Methods:**
```csharp
DeviceFingerprint GetFingerprint()
  // Returns struct with:
  // - TpmHash: string
  // - SystemUuidHash: string
  // - CpuIdHash: string
  // - MacAddressHashes: string[]
  // - DiskSerialHashes: string[]
  // - GpuIdHashes: string[]

void ClearCache()
  // Clear cached fingerprint (for testing)
```

---

#### ILauncherApiService
**Purpose:** HTTP communication with backend

**Methods:**
```csharp
Task<AuthInitResult> InitiateAuthAsync(
  string hwid,
  string? legacyHwid = null,
  DeviceFingerprint? fingerprint = null)
  // POST /launcher/initiate
  // Returns: { authCode, visibleCode, deepLink, expiresIn }

Task<AuthCheckResult> CheckAuthStatusAsync(
  string authCode, string hwid)
  // GET /launcher/status?code=...&hwid=...
  // Returns: { status, token, downloadToken, user }

Task<List<ApiModPack>> GetModPacksAsync()
  // GET /launcher/modpacks
  // Returns: List of available modpacks

Task<UpdateInfo?> CheckForUpdatesAsync()
  // GET /launcher/version
  // Returns: { latestVersion, downloadUrl, changelog, mandatory }

Task<bool> VerifySessionAsync()
  // GET /launcher/verify
  // Validate current JWT token

Task<AccountBindingInfo?> GetAccountBindingInfoAsync(string hwid)
  // GET /launcher/binding?hwid=...
  // Check if account linked to Telegram

Task<(bool IsAccessible, long PingMs)> MeasurePingAsync(string serverUrl)
  // TCP ping measurement

Task<bool> ResetAccountBindingAsync(string hwid)
  // POST /launcher/reset-binding
  // Unbind Telegram account
```

**Properties:**
```csharp
string? AuthToken { get; }
string? DownloadToken { get; }
UserInfo? CurrentUser { get; }
bool IsAuthenticated { get; }
bool IsUsingFallbackServer { get; }
string CurrentStorageBaseUrl { get; }
```

---

#### IDownloadService
**Purpose:** File download with retry, checksum verification

**Methods:**
```csharp
Task<bool> DownloadFileAsync(
  string url,
  string filePath,
  string? expectedChecksum = null,
  IProgress<DownloadProgress>? progress = null,
  CancellationToken cancellationToken = default)
  // Single file download with optional checksum validation

Task DownloadManifestAsync(
  List<ManifestFile> files,
  string targetDirectory,
  IProgress<ManifestProgress>? progress = null)
  // Concurrent downloads (4 parallel) with retry logic
```

---

#### IModManagerService
**Purpose:** Modpack installation, switching, cleanup

**Methods:**
```csharp
Task<bool> InstallModPackAsync(
  ApiModPack pack,
  IProgress<double>? progress = null)
  // Download and extract modpack

Task<bool> UninstallModPackAsync(string modPackId)
  // Remove modpack files and metadata

Task<bool> IsModPackInstalledAsync(string modPackId)
  // Check if pack exists locally

Task<List<ConflictingFile>> CheckForConflictsAsync(
  string modPack1Id,
  string modPack2Id)
  // Find overlapping files between packs

Task SwitchModPackAsync(
  string fromModPackId,
  string toModPackId)
  // Atomically swap mods
```

---

#### IGameService
**Purpose:** Game detection, launch, process monitoring

**Methods:**
```csharp
Task DetectInstalledGamesAsync()
  // Scan Windows Registry for game installations

Task<bool> LaunchGameAsync(GameType gameType)
  // Start game executable in foreground

Task<bool> IsGameRunningAsync(GameType gameType)
  // Check process table for game.exe

Task<bool> ValidateGameFilesAsync(
  GameType gameType,
  IProgress<double>? progress = null)
  // Hash verification against manifest
```

---

#### ISettingsService
**Purpose:** Persistent application configuration

**Methods:**
```csharp
AppSettings LoadSettings()
  // Read from %APPDATA%/DQLauncher/settings.json

void SaveSettings(AppSettings settings)
  // Write settings to disk

void ResetToDefaults()
  // Restore factory defaults
```

**Settings Properties:**
- Theme (Dark/Light)
- Language (EN/RU/IT)
- GameInstallPath
- AutoUpdate (bool)
- SavedAuthToken
- ServerPreference (index)
- VpnSettings

---

#### IVpnService
**Purpose:** VPN configuration and control

**Methods:**
```csharp
Task<bool> LoadConfigFromServerAsync(string endpoint)
  // Fetch .ovpn or WireGuard config

Task<bool> StartAsync()
  // Activate VPN tunnel

Task<bool> StopAsync()
  // Disable VPN

Task<bool> IsConnectedAsync()
  // Check tunnel status

string GetStatus()
  // Connection status string
```

---

#### IReliableUpdateService
**Purpose:** Application self-updates with rollback

**Methods:**
```csharp
Task<UpdateInfo?> CheckForUpdateAsync()
  // Query /launcher/version endpoint

Task<bool> DownloadAndApplyUpdateAsync(
  UpdateInfo info,
  IProgress<double>? progress = null)
  // Download, verify, extract, replace current version

Task<bool> RollbackToPreviousAsync()
  // Restore backup from previous version

bool IsUpdateRequired(string minVersion)
  // Check if current version < minVersion
```

---

### Implementations

#### HardwareFingerprintService
**File:** `Services/Implementations/HardwareFingerprintService.cs`

**Responsibility:** WMI queries for hardware components

**Private Methods:**
- `CollectRawData()` — WMI queries:
  - Win32_ComputerSystemProduct → UUID
  - Win32_Processor → CPU serial
  - Win32_NetworkAdapterConfiguration → MAC addresses
  - Win32_LogicalDisk → Disk serials
  - Win32_VideoController → GPU IDs
  - Win32_Tpm → TPM info

- `HashFingerprint()` — SHA256 hash all values
- `CacheResult()` — Store in-memory cache

**Data Structure:**
```csharp
public class DeviceFingerprint
{
    public string? TpmHash { get; set; }
    public string? SystemUuidHash { get; set; }
    public string? CpuIdHash { get; set; }
    public string[]? MacAddressHashes { get; set; }
    public string[]? DiskSerialHashes { get; set; }
    public string[]? GpuIdHashes { get; set; }
}
```

---

#### LauncherApiService
**File:** `Services/Implementations/LauncherApiService.cs`

**Responsibility:** HTTP client for backend communication

**Key Logic:**
- **Multi-server failover:** Try primary, fallback to secondary (5 min cooldown)
- **JWT management:** Load from settings, auto-refresh on 401
- **Server selection:** User preference > latency-based > fallback
- **VPN detection:** Test Germany server accessibility, auto-enable if blocked

**Private Methods:**
- `CreateHttpClient()` — Setup HttpClient with timeouts, headers
- `LoadServerPreferenceFromSettings()` — Restore server index
- `SaveServerPreference()` — Persist server choice
- `LoadSavedToken()` — Load JWT from encrypted settings
- `TestGermanyAccessAsync()` — Speed test to storage server
- `SwitchServerAsync(int index)` — Rotate to fallback server

**Server Lists:**
```csharp
ApiServers = new[] {
  new ServerInfo { Url = "https://api.dq.local:5000", Region = "Local" },
  new ServerInfo { Url = "https://api.backup.dq.local:5001", Region = "Fallback" }
};

StorageServers = new[] {
  "https://storage-ger.dq.local:5000",
  "https://storage-eu.dq.local:5000"
};
```

---

#### ManifestDownloadService
**File:** `Services/Implementations/ManifestDownloadService.cs`

**Responsibility:** Download game files with concurrency and verification

**Algorithm:**
```
1. Fetch manifest: List<ManifestFile> with size + SHA256
2. For each file:
   a. Compare local vs remote checksum
   b. If different: delete local (no resume)
   c. Download with 4 concurrent streams
   d. WriteThrough to disk (synchronous write)
   e. Verify: size check first, then SHA256
   f. If mismatch: delete + retry

3. Progress reporting every 500ms
4. Retry with exponential backoff on failure
```

**Key Methods:**
- `DownloadManifestAsync()` — Main orchestrator
- `DownloadSingleFileAsync()` — Per-file logic
- `DownloadFileWithRetryAsync()` — Retry wrapper
- `VerifyFileChecksumAsync()` — Hash verification

**Concurrency Control:**
```csharp
private const int MaxConcurrentDownloads = 4;
private readonly SemaphoreSlim _downloadSemaphore =
  new(MaxConcurrentDownloads);
```

---

#### ModManagerService
**File:** `Services/Implementations/ModManagerService.cs`

**Responsibility:** Local modpack installation and switching

**Storage Structure:**
```
%APPDATA%/DQLauncher/
├── mods/
│   ├── modpack-id-1/
│   │   ├── files/...
│   │   └── metadata.json
│   └── modpack-id-2/
└── active-modpack.json
```

**Key Methods:**
- `InstallModPackAsync()` — Extract to local folder
- `UninstallModPackAsync()` — Recursive delete + cleanup
- `SwitchModPackAsync()` — Backup configs, apply new ones
- `ValidateIntegrityAsync()` — Verify all files present

---

#### GameService
**File:** `Services/Implementations/GameService.cs`

**Responsibility:** Game detection and launch

**Detection Paths:**
```csharp
WindowsRegistry paths:
- HKLM\SOFTWARE\SCS Software\Euro Truck Simulator 2
- HKLM\SOFTWARE\SCS Software\American Truck Simulator
- HKLM\SOFTWARE\Rockstar Games\GTA V
```

**Process Monitoring:**
```csharp
// Detect running game
Process.GetProcessesByName("game.exe").Any()

// Launch with correct working directory
ProcessStartInfo {
  FileName = gameExePath,
  WorkingDirectory = gameDirectory,
  UseShellExecute = false
}
```

---

#### DownloadService
**File:** `Services/Implementations/DownloadService.cs`

**Responsibility:** Low-level HTTP file download

**Features:**
- Resume on partial downloads
- Timeout handling
- Progress reporting via `IProgress<DownloadProgress>`
- Automatic retry on transient errors

```csharp
public struct DownloadProgress
{
    public long BytesReceived { get; set; }
    public long TotalBytes { get; set; }
    public double PercentComplete { get; set; }
    public TimeSpan TimeRemaining { get; set; }
}
```

---

#### SettingsService
**File:** `Services/Implementations/SettingsService.cs`

**Responsibility:** Persistent configuration storage

**Storage:**
```
%APPDATA%/DQLauncher/settings.json
(JSON with encrypted sensitive fields)
```

**Data Model:**
```csharp
public class AppSettings
{
    public string Theme { get; set; } = "Dark";
    public string Language { get; set; } = "EN";
    public string GameInstallPath { get; set; } = "";
    public string? SavedAuthToken { get; set; }
    public int ServerPreference { get; set; } = 0;
    public bool AutoUpdate { get; set; } = true;
    public Dictionary<string, string> GamePaths { get; set; }
    public VpnSettings VpnSettings { get; set; }
}
```

---

#### VpnService
**File:** `Services/Implementations/VpnService.cs`

**Responsibility:** VPN client integration

**Supported Protocols:**
- OpenVPN (.ovpn config)
- WireGuard (interface config)
- Built-in VPN tunneling

**Control Flow:**
```
1. Fetch config from server
2. Parse OpenVPN/WireGuard format
3. Launch VPN client or configure tunnel
4. Monitor connection status
5. Auto-reconnect on timeout
```

---

## Models

### AppSettings
**File:** `Models/AppSettings.cs`

Configuration stored in `%APPDATA%/DQLauncher/settings.json`

```csharp
public class AppSettings
{
    public string Theme { get; set; }
    public string Language { get; set; }
    public string GameInstallPath { get; set; }
    public int ServerPreference { get; set; }
    public bool AutoUpdate { get; set; }
    public string? SavedAuthToken { get; set; }
    public Dictionary<string, string> GamePaths { get; set; }
}
```

---

### ModPack
**File:** `Models/ModPack.cs`

Observable model for UI binding

```csharp
public sealed partial class ModPack : ObservableObject
{
    [ObservableProperty] bool IsInstalled;
    [ObservableProperty] bool IsDownloading;
    [ObservableProperty] double DownloadProgress;
    [ObservableProperty] bool IsUpdateAvailable;
    [ObservableProperty] string? InstalledVersion;

    // Immutable properties
    public required string Id { get; init; }
    public required string Name { get; init; }
    public required long SizeBytes { get; init; }
}
```

---

### ManifestFile
**File:** `Models/ManifestFile.cs`

Single file in manifest

```csharp
public class ManifestFile
{
    public string Path { get; set; }        // Relative path
    public long Size { get; set; }
    public string Checksum { get; set; }    // SHA256
    public string? Url { get; set; }        // Download URL
}
```

---

### UpdateManifest
**File:** `Models/UpdateManifest.cs`

Version check response

```csharp
public class UpdateManifest
{
    public string Version { get; set; }
    public string DownloadUrl { get; set; }
    public string? Changelog { get; set; }
    public bool Mandatory { get; set; }
    public long FileSizeBytes { get; set; }
    public string FileChecksum { get; set; }
}
```

---

## Core

### Constants.cs
Central configuration values

```csharp
public static class Constants
{
    public static class Network
    {
        public const int RequestTimeoutMs = 30000;
        public const int VpnSpeedTestTimeoutMs = 5000;
        public const int DownloadTimeoutMs = 300000;
        public const int MaxRetries = 3;
    }

    public static class Urls
    {
        public const string GermanyServer = "https://storage-ger.dq.local";
        public const string VpnConfigEndpoint = "/api/vpn/config";
    }
}
```

---

### AppPaths.cs
Centralized path handling

```csharp
public static class AppPaths
{
    public static string AppDataDir => Path.Combine(
        Environment.GetFolderPath(
            Environment.SpecialFolder.ApplicationData),
        "DQLauncher");

    public static string SettingsFile =>
        Path.Combine(AppDataDir, "settings.json");

    public static string ModsDirectory =>
        Path.Combine(AppDataDir, "mods");
}
```

---

### AppVersion.cs
Version and build info

```csharp
public static class AppVersion
{
    public const string Current = "3.1.0";
    public static readonly DateTime BuildDate =
        new(2026, 01, 16);
}
```
