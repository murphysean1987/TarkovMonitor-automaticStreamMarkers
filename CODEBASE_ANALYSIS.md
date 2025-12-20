# TarkovMonitor Codebase Analysis

## Overview
TarkovMonitor is a C# Windows Forms application built with Blazor WebView that monitors Escape from Tarkov game logs and provides real-time notifications, task tracking, and third-party service integrations.

## Core Architecture

### Entry Point
- **Program.cs**
  - `Main()`: Application entry point that initializes the app, shows a splash screen, and launches the main UI

### Main Components

## 1. GameWatcher.cs
**Purpose**: Core log monitoring and game event detection engine

### Key Methods:
- `Start()`: Initializes log monitoring, process watching, and screenshot detection
- `GetDefaultLogsFolder()`: Retrieves EFT installation path from Windows Registry
- `WatchLogsFolder(string folderPath)`: Monitors a specific log folder for new entries
- `GameWatcher_NewLogData(object? sender, NewLogDataEventArgs e)`: Main event handler that parses log data using regex patterns
- `GetLogDetails(string folderPath)`: Extracts profile and session information from logs
- `GetLogBreakpoints(string profileId)`: Retrieves historical log breakpoints for a profile
- `ProcessLogsFromBreakpoint(LogDetails breakpoint)`: Processes historical logs from a specific point
- `UpdateProcess()`: Monitors EFT process state (running/stopped)
- `ScreenshotWatcher_Created(object sender, FileSystemEventArgs e)`: Processes screenshot files to extract player position
- `QuarternionsToYaw(float x, float z, float y, float w)`: Converts quaternion rotation to yaw angle

### Events Raised:
- `GameStarted`: EFT process detected
- `MapLoading`: Map bundle loading detected
- `MatchFound`: Matching completed, locked to server
- `MapLoaded`: Map fully loaded
- `RaidStarting`: Raid countdown started (PMC) or immediate start (Scav)
- `RaidStarted`: Raid begins
- `RaidExited`: Player exited raid
- `RaidEnded`: Raid ended
- `ExitedPostRaidMenus`: Player returned to main menu
- `TaskStarted`, `TaskFailed`, `TaskFinished`: Quest status changes
- `FleaSold`, `FleaOfferExpired`: Flea market events
- `PlayerPosition`: Screenshot with position data detected
- `GroupInviteAccept`, `GroupUserLeave`, `GroupDisbanded`, etc.: Group events

## 2. LogMonitor.cs
**Purpose**: Monitors individual log files for new data

### Key Methods:
- `Start()`: Begins asynchronous monitoring of a log file
- `Stop()`: Stops monitoring

### Events:
- `InitialReadComplete`: Log file initial read finished
- `NewLogData`: New log data detected
- `Exception`: Error occurred during monitoring

## 3. MainBlazorUI.cs
**Purpose**: Main application window and event coordinator

### Key Methods:
- `MainBlazorUI()`: Constructor that initializes services, dependency injection, and event handlers
- `UpdateTarkovDevApiData()`: Fetches game data from tarkov.dev API
- `InitializeProgress()`: Sets up TarkovTracker integration
- `Eft_RaidStart(object? sender, RaidInfoEventArgs e)`: Handles raid start events
- `Eft_TaskFinished(object? sender, LogContentEventArgs<TaskStatusMessageLogContent> e)`: Updates quest completion
- `Eft_FleaSold(object? sender, LogContentEventArgs<FleaSoldMessageLogContent> e)`: Logs flea market sales
- `Eft_PlayerPosition(object? sender, PlayerPositionEventArgs e)`: Sends position to tarkov.dev remote
- `Handle_Screenshots(RaidInfoEventArgs e, MonitorMessage monMessage)`: Manages screenshot cleanup
- `AddGoonsButton(MonitorMessage monMessage, RaidInfo raidInfo)`: Adds goons reporting functionality

### Dependency Injection Services:
- `GameWatcher`: Log monitoring
- `MessageLog`: User-facing messages
- `LogRepository`: Raw log storage
- `GroupManager`: Group member tracking
- `TimersManager`: In-raid timers
- `LocalizationService`: Multi-language support

## 4. TarkovTracker.cs
**Purpose**: Integration with TarkovTracker.io API for quest tracking

### Key Methods:
- `InitAPI()`: Initializes Refit REST client with authentication
- `TestToken(string apiToken)`: Validates API token
- `SetProfile(string profileId)`: Switches between PVP/PVE profiles
- `GetProgress()`: Retrieves user's quest progress
- `SetTaskStatus(string questId, TaskStatus status)`: Updates quest status
- `SetTaskComplete(string questId)`: Marks quest as complete
- `SetTaskFailed(string questId)`: Marks quest as failed
- `HasAirFilter()`: Checks if user has hideout air filter

### Authentication:
- Uses **Bearer token** authentication
- Tokens stored per profile ID in application settings
- Token validation requires "WP" (write permission) scope
- Rate limiting: 15 requests per minute (noted in comments)

## 5. TarkovDev.cs
**Purpose**: Integration with tarkov.dev GraphQL API and data submission endpoints

### Key Methods:
- `GetTasks()`: Fetches quest data via GraphQL
- `GetMaps()`: Retrieves map information
- `GetItems()`: Gets item database
- `GetTraders()`: Fetches trader information
- `GetHideout()`: Retrieves hideout station data
- `UpdateApiData()`: Refreshes all API data
- `PostQueueTime(...)`: Submits queue time statistics
- `PostGoonsSighting(...)`: Reports goons location
- `GetExperience(int accountId)`: Retrieves player experience
- `ScavCooldownSeconds()`: Calculates scav cooldown based on hideout/karma

### APIs Used:
- GraphQL endpoint: `https://api.tarkov.dev/graphql`
- Data submission: `https://manager.tarkov.dev/api`
- Player data: `https://player.tarkov.dev`

## 6. SocketClient.cs
**Purpose**: WebSocket client for tarkov.dev remote control feature

### Key Methods:
- `Send(List<JsonObject> messages)`: Sends commands via WebSocket
- `UpdatePlayerPosition(PlayerPositionEventArgs e)`: Sends player position to website
- `NavigateToMap(TarkovDev.Map map)`: Controls website map navigation
- `GetPlayerPositionMessage(PlayerPositionEventArgs e)`: Creates position update message
- `GetNavigateToMapMessage(TarkovDev.Map map)`: Creates navigation command

### WebSocket:
- URL: `wss://socket.tarkov.dev`
- Session ID from user settings appended as query parameter
- No authentication beyond session ID

## 7. Sound.cs
**Purpose**: Audio notification playback

### Key Methods:
- `Play(string key)`: Plays notification sound (from resources or custom)
- `SetCustomSound(string key, string path)`: Sets custom notification sound
- `RemoveCustomSound(string key)`: Removes custom sound
- `IsCustom(string key)`: Checks if custom sound is set
- `GetPlaybackDevices()`: Lists available audio devices

### Sound Types:
- air_filter_off, air_filter_on
- match_found, raid_starting
- restart_failed_tasks
- runthrough_over
- scav_available
- quest_items

## 8. Stats.cs
**Purpose**: Local SQLite database for statistics tracking

### Key Methods:
- `AddFleaSale(FleaSoldMessageLogContent e, Profile profile)`: Records flea market sale
- `GetTotalSales(string currency)`: Retrieves total sales for a currency
- `AddRaid(RaidInfoEventArgs e)`: Records raid statistics
- `GetTotalRaids(string mapNameId)`: Gets raid count for a map
- `GetTotalRaidsPerMap(RaidType raidType)`: Retrieves raids per map
- `ClearData()`: Wipes all statistics

### Database:
- SQLite database at `{UserAppDataPath}/TarkovMonitor.db`
- Tables: `flea_sales`, `raids`
- Automatic schema migration support

## 9. TimersManager.cs
**Purpose**: Manages in-raid timers and scav cooldown

### Key Methods:
- Constructor subscribes to `RaidStarted` and `RaidEnded` events
- `TimerRaid_Elapsed(object state)`: Updates time in raid
- `TimerRunThrough_Elapsed(object state)`: Counts down runthrough timer
- `timerScavCooldown_Elapsed(object state)`: Tracks scav cooldown

### Events:
- `RaidTimerChanged`: Time in raid updated
- `RunThroughTimerChanged`: Runthrough timer updated
- `ScavCooldownTimerChanged`: Scav cooldown updated

## 10. MessageLog.cs
**Purpose**: User-facing message/notification system

### Key Methods:
- `AddMessage(MonitorMessage message)`: Adds structured message
- `AddMessage(string message, string? type, string? url)`: Adds simple message

### Events:
- `newMessage`: New message added to log

## 11. LogRepository.cs
**Purpose**: Stores raw log data for debugging/analysis

### Key Methods:
- `AddLog(LogLine message)`: Adds log entry
- `AddLog(string message, string? type)`: Adds log with type

### Events:
- `newLog`: New log line added

## 12. GroupManager.cs
**Purpose**: Tracks group members and loadouts

### Key Methods:
- `UpdateGroupMember(GroupMatchRaidReadyLogContent member)`: Updates member status
- `RemoveGroupMember(string name)`: Removes member from group
- `ClearGroup()`: Clears all members

### Events:
- `GroupMemberChanged`: Group composition changed

## 13. UpdateCheck.cs
**Purpose**: Checks for application updates

### Key Methods:
- `CheckForNewVersion()`: Queries GitHub API for latest release

### Events:
- `NewVersion`: New version available
- `Error`: Error checking for updates

### API:
- GitHub Releases API: `https://api.github.com/repos/the-hideout/TarkovMonitor/releases/latest`

## Supporting Classes

### MonitorMessage.cs
- Represents user-facing notification with buttons, selects, and confirmation dialogs

### LogLine.cs
- Represents a single raw log entry with type classification

### LogMessageTypes.cs
- Data classes for various log message types (tasks, flea market, groups, etc.)

### LocalizationService.cs
- Multi-language support using .NET resource files

### Splash.cs
- Splash screen displayed on startup

### GetProcessFilename.cs
- Utility for retrieving process executable paths

## Data Flow

1. **Log Detection**:
   ```
   GameWatcher → LogMonitor → NewLogData event → GameWatcher_NewLogData
   ```

2. **Event Processing**:
   ```
   Regex parsing → Event emission (MatchFound, RaidStarted, etc.)
   ```

3. **UI Updates**:
   ```
   MainBlazorUI event handlers → MessageLog → Blazor components
   ```

4. **External Services**:
   ```
   Events → TarkovTracker/TarkovDev API calls → Response handling
   ```

5. **Statistics**:
   ```
   Events → Stats.cs → SQLite database
   ```

## Authentication Patterns

### TarkovTracker
- **Type**: Bearer Token (API key)
- **Storage**: Per-profile in application settings (JSON serialized)
- **Validation**: Test endpoint checks permissions
- **Authorization Header**: `Authorization: Bearer {token}`
- **Token Management**: Manual token creation on TarkovTracker website
- **Scopes**: Requires "WP" (write permission)

### TarkovDev
- **GraphQL API**: No authentication required
- **Data Submission**: No authentication required
- **WebSocket Remote**: Session ID only (no OAuth)

### Pattern Summary
- Simple API key-based authentication
- Tokens stored in user settings
- No OAuth 2.0 flow currently implemented
- Per-profile token storage for multi-account support
