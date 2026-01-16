# Classi & Servizi Desktop (.NET 8 / WPF)

Riferimento completo di tutte le classi .NET in DQ Launcher.

## ViewModels

### MainViewModel.cs
**Responsabilità:** Stato applicazione core, inizializzazione, orchestrazione comandi

**Proprietà Principali:**
- `IsAuthenticating` — Login in progress
- `IsAuthenticated` — Utente connesso
- `CurrentPage` — Pagina attiva (Games, Mods, Settings)
- `GameStatus` — Dizionario stati installazione gioco

**Metodi Principali:**
- `InitializeAsync()` — Carica settings, rileva hardware, controlla aggiornamenti
- `RefreshGameStatusAsync()` — Poll server per cambio manifest
- `HandleErrorAsync()` — Gestione errori centralizzata

---

### MainViewModel.Auth.cs
**Responsabilità:** Flusso autenticazione, gestione token, riconoscimento dispositivo

**Proprietà:**
- `AuthCode` — Codice auth temporaneo dal server
- `LoginUrl` — Link deep link OAuth Telegram
- `IsWaitingForTelegramConfirm` — Stato polling

**Metodi:**
- `LoginCommand.Execute()` — Avvia flusso OAuth Telegram
- `LogoutCommand.Execute()` — Cancella token, reset UI
- `VerifySessionAsync()` — Verifica se sessione valida

---

### MainViewModel.Download.cs
**Responsabilità:** Coda download, tracciamento progresso, recovery errori

**Proprietà:**
- `ActiveDownloads` — ObservableCollection download in corso
- `TotalDownloadProgress` — Progresso aggregato 0-100%
- `CurrentDownloadSpeed` — Mbps
- `EstimatedTimeRemaining` — TimeSpan

---

### MainViewModel.Games.cs
**Responsabilità:** Stato gioco, avvio, configurazione

**Metodi:**
- `LaunchGameCommand.Execute(GameType)` — Avvia eseguibile gioco
- `DetectInstalledGamesAsync()` — Scansione percorsi installazione
- `ValidateGameFilesAsync()` — Verifica checksum

---

### MainViewModel.Mods.cs
**Responsabilità:** Gestione modpack, installazione, switch

**Metodi:**
- `InstallModpackAsync(ModPack)` — Scarica ed estrae
- `UninstallModpackAsync(string id)` — Rimuove file
- `SwitchModpackAsync(string id1, string id2)` — Scambia tra config

---

### MainViewModel.Settings.cs
**Responsabilità:** Preferenze utente, tema, lingua, percorsi gioco

**Proprietà:**
- `SelectedTheme` — Dark/Light
- `SelectedLanguage` — EN/RU/IT
- `GameInstallPath` — Cartella gioco
- `AutoUpdate` — Boolean

---

### MainViewModel.Connection.cs
**Responsabilità:** Selezione server, diagnostica rete, controllo VPN

**Metodi:**
- `AutoSelectBestServerAsync()` — Ping, seleziona latenza minore
- `ForceServerAsync(int index)` — Override selezione
- `ConfigureVpnIfNeededAsync()` — Auto-abilita VPN se Germania bloccata

---

### MainViewModel.Update.cs
**Responsabilità:** Auto-aggiornamenti applicazione, delta patching

**Metodi:**
- `CheckForUpdatesAsync()` — Query server per versione nuova
- `DownloadUpdateAsync()` — Scarica e applica delta
- `RollbackAsync()` — Ripristina versione precedente

---

## Servizi - Interfacce

### IHardwareFingerprintService
**Scopo:** Raccoglie e hash identificatori hardware

```csharp
DeviceFingerprint GetFingerprint()
  // Ritorna SHA256 hash di:
  // - TPM, UUID, CPU, MAC, Disk, GPU

void ClearCache()
  // Cancella cache (testing)
```

---

### ILauncherApiService
**Scopo:** Comunicazione HTTP con backend

**Metodi:**
- `InitiateAuthAsync()` — Avvia OAuth, riceve auth code
- `CheckAuthStatusAsync()` — Poll stato autenticazione
- `GetModPacksAsync()` — Lista modpack disponibili
- `CheckForUpdatesAsync()` — Controlla versione nuova
- `VerifySessionAsync()` — Valida JWT token corrente

---

### IDownloadService
**Scopo:** Download file con retry, verifica checksum

**Metodi:**
- `DownloadFileAsync()` — Download singolo file con validazione
- `DownloadManifestAsync()` — Download concorrenti (4 paralleli)

---

### IModManagerService
**Scopo:** Installazione modpack, switching, cleanup

**Metodi:**
- `InstallModPackAsync()` — Download ed estrazione
- `UninstallModPackAsync()` — Rimozione file
- `CheckForConflictsAsync()` — Trova file conflittuali

---

### IGameService
**Scopo:** Rilevamento gioco, avvio, monitoraggio processo

**Metodi:**
- `DetectInstalledGamesAsync()` — Scansione Registry
- `LaunchGameAsync(GameType)` — Avvia eseguibile
- `IsGameRunningAsync(GameType)` — Controlla processo

---

### ISettingsService
**Scopo:** Configurazione persistente applicazione

**Metodi:**
- `LoadSettings()` — Legge da %APPDATA%/settings.json
- `SaveSettings()` — Salva su disco
- `ResetToDefaults()` — Ripristina factory defaults

---

### IVpnService
**Scopo:** Configurazione e controllo VPN

**Metodi:**
- `LoadConfigFromServerAsync()` — Scarica config .ovpn
- `StartAsync()` — Attiva tunnel VPN
- `StopAsync()` — Disabilita VPN
- `IsConnectedAsync()` — Verifica status tunnel

---

### IReliableUpdateService
**Scopo:** Auto-aggiornamenti applicazione con rollback

**Metodi:**
- `CheckForUpdateAsync()` — Query endpoint versione
- `DownloadAndApplyUpdateAsync()` — Scarica, verifica, estrae
- `RollbackToPreviousAsync()` — Ripristina backup versione precedente

---

## Servizi - Implementazioni

### HardwareFingerprintService
**File:** `Services/Implementations/HardwareFingerprintService.cs`

**Responsabilità:** Query WMI per componenti hardware

**Query WMI:**
- Win32_ComputerSystemProduct → UUID
- Win32_Processor → CPU serial
- Win32_NetworkAdapterConfiguration → indirizzi MAC
- Win32_LogicalDisk → seriali dischi
- Win32_VideoController → ID GPU
- Win32_Tpm → info TPM

**Tutti i valori hashati con SHA256**

---

### LauncherApiService
**File:** `Services/Implementations/LauncherApiService.cs`

**Responsabilità:** Client HTTP per comunicazione backend

**Logica Chiave:**
- **Failover multi-server:** Prova primario, fallback secondario (cooldown 5 min)
- **Gestione JWT:** Carica da settings, auto-refresh su 401
- **Selezione server:** Preferenza utente > latenza-based > fallback
- **Rilevamento VPN:** Test accessibilità server Germania, auto-abilita se bloccato

---

### ManifestDownloadService
**File:** `Services/Implementations/ManifestDownloadService.cs`

**Responsabilità:** Download file gioco con concorrenza e verifica

**Algoritmo:**
1. Fetch manifest: List<ManifestFile> con size + SHA256
2. Per ogni file:
   - Confronto checksum locale vs remoto
   - Se diverso: cancella (no resume)
   - Download 4 stream concorrenti
   - WriteThrough a disco
   - Verifica: check dimensione prima, poi SHA256
   - Su mismatch: cancella + retry

**Controllo Concorrenza:**
- MaxConcurrentDownloads = 4
- SemaphoreSlim per limitare

---

### ModManagerService
**File:** `Services/Implementations/ModManagerService.cs`

**Responsabilità:** Installazione modpack locale, switch

**Storage:**
```
%APPDATA%/DQLauncher/mods/
├── modpack-id-1/
│   ├── files/...
│   └── metadata.json
└── modpack-id-2/
```

---

### GameService
**File:** `Services/Implementations/GameService.cs`

**Responsabilità:** Rilevamento e avvio gioco

**Percorsi Registry:**
- HKLM\SOFTWARE\SCS Software\Euro Truck Simulator 2
- HKLM\SOFTWARE\SCS Software\American Truck Simulator

**Monitoraggio Processo:**
- Process.GetProcessesByName("game.exe")
- ProcessStartInfo con working directory

---

### VpnService
**File:** `Services/Implementations/VpnService.cs`

**Responsabilità:** Integrazione client VPN

**Protocolli Supportati:**
- OpenVPN (.ovpn config)
- WireGuard (interface config)

---

## Modelli

### AppSettings
Configurazione in `%APPDATA%/DQLauncher/settings.json`

```csharp
public class AppSettings
{
    public string Theme { get; set; }
    public string Language { get; set; }
    public string GameInstallPath { get; set; }
    public int ServerPreference { get; set; }
    public bool AutoUpdate { get; set; }
    public string? SavedAuthToken { get; set; }
}
```

### ModPack
Modello observable per UI binding

```csharp
public sealed partial class ModPack : ObservableObject
{
    [ObservableProperty] bool IsInstalled;
    [ObservableProperty] double DownloadProgress;
    [ObservableProperty] bool IsUpdateAvailable;
    public required string Name { get; init; }
    public required long SizeBytes { get; init; }
}
```

---

## Core

### Constants.cs
Valori configurazione centralizzati

```csharp
public static class Constants
{
    public const int RequestTimeoutMs = 30000;
    public const int MaxRetries = 3;
    public const string GermanyServer = "https://storage-ger.dq.local";
}
```

### AppPaths.cs
Gestione centralizzata percorsi

```csharp
public static string AppDataDir =>
  Path.Combine(Environment.GetFolderPath(
    Environment.SpecialFolder.ApplicationData), "DQLauncher");
```

---

Vedi [architecture.it.md](architecture.it.md) per design del sistema completo.
