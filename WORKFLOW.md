# TarkovMonitor Workflow and Event Flow

## Application Lifecycle

### 1. Application Startup

```
Program.Main()
    ├─> Show Splash Screen (2 seconds or skip if configured)
    └─> Launch MainBlazorUI
        ├─> Initialize Services (DI Container)
        │   ├─> GameWatcher (eft)
        │   ├─> MessageLog
        │   ├─> LogRepository
        │   ├─> GroupManager
        │   ├─> TimersManager
        │   └─> LocalizationService
        ├─> Setup Blazor WebView with MudBlazor UI
        ├─> Subscribe to GameWatcher events
        ├─> Start GameWatcher
        │   ├─> Detect logs folder from registry
        │   ├─> Get latest log folder
        │   ├─> Start LogMonitor for application.log
        │   ├─> Start LogMonitor for notifications.log
        │   ├─> Setup screenshot watcher
        │   └─> Start EFT process monitoring timer
        └─> Check for updates (GitHub API)
```

### 2. Initial Log Processing

```
LogMonitor.Start()
    ├─> Read current file size
    ├─> Emit InitialReadComplete event
    └─> Enter monitoring loop (every 5 seconds)
        ├─> Check for new bytes
        ├─> Read new data
        └─> Emit NewLogData event

GameWatcher receives InitialReadComplete
    ├─> Update TarkovDev API data
    │   ├─> Fetch tasks (quests)
    │   ├─> Fetch maps
    │   ├─> Fetch items
    │   ├─> Fetch traders
    │   └─> Fetch hideout stations
    ├─> Initialize TarkovTracker profile
    │   ├─> Migrate old token if exists
    │   ├─> Set profile ID
    │   ├─> Test token validity
    │   └─> Retrieve progress data
    └─> Update player names from TarkovDev
```

## Game Event Workflows

### 3. Raid Start Workflow

```
EFT Game Log Entries:
1. "Session mode: Regular" or "Session mode: PVE"
   └─> Sets CurrentProfile.Type

2. "SelectProfile ProfileId:xxx AccountId:xxx"
   └─> Sets CurrentProfile.Id and AccountId
   └─> Emit ProfileChanged event

3. "scene preset path:maps/{bundle}.bundle"
   └─> Create new RaidInfo
   └─> Parse map bundle name
   └─> Emit MapLoading event
       ├─> MainBlazorUI checks for failed restartable tasks
       │   └─> Play "restart_failed_tasks" sound if configured
       ├─> Play "air_filter_on" sound if hideout has air filter
       └─> Play "quest_items" reminder if configured

4. "LocationLoaded" 
   └─> Record map load time

5. "MatchingCompleted"
   └─> Record queue time

6. "TRACE-NetworkGameCreate profileStatus"
   └─> Parse map, online status, raid ID
   └─> Check if reconnecting to existing raid
   └─> Emit MatchFound event (if new raid)
       └─> MainBlazorUI plays "match_found" sound
   └─> Emit MapLoaded event
       └─> SocketClient navigates tarkov.dev remote to map

7. "GameStarting" (PMC countdown or sometimes Scav)
   └─> Record StartingTime
   └─> Emit RaidStarting event
       └─> MainBlazorUI plays "raid_starting" sound

8. "GameStarted" (raid actually begins)
   └─> Record StartedTime
   └─> Emit RaidStarted event
       ├─> MainBlazorUI
       │   ├─> Log raid start message
       │   ├─> Record raid in Stats database
       │   ├─> Add "Report Goons" button if applicable
       │   ├─> Start runthrough timer if PMC/PVE
       │   └─> Submit queue time to TarkovDev if enabled
       └─> TimersManager
           ├─> Start raid timer (time in raid)
           └─> Start runthrough countdown timer
```

### 4. During Raid

```
Screenshot Taken in Game
    ├─> ScreenshotWatcher detects new .png file
    ├─> Parse filename for position and rotation data
    │   └─> Extract x, y, z, and quaternion rotation
    ├─> Convert quaternion to yaw angle
    └─> Emit PlayerPosition event
        └─> MainBlazorUI
            ├─> Log position to messages
            └─> Send to SocketClient
                └─> Send WebSocket message to tarkov.dev
                    ├─> Update player position on map
                    └─> Navigate to map (if configured)

Runthrough Timer Expires (after ~10 minutes)
    └─> MainBlazorUI
        ├─> Play "runthrough_over" sound
        └─> Log message
```

### 5. Raid End Workflow

```
EFT Game Log Entry: "Got notification | UserMatchOver"
    └─> Parse location and raidId
    └─> Emit RaidExited event
        ├─> MainBlazorUI logs "Exited {map} raid"
        ├─> GroupManager marks group as stale
        ├─> Stop runthrough timer
        └─> Mark inRaid = false

Next Profile Selection (return to main menu)
    └─> "SelectProfile ProfileId:xxx"
        └─> Emit RaidEnded event
            ├─> MainBlazorUI
            │   ├─> Log "Ended {map} raid"
            │   ├─> Handle screenshot cleanup
            │   │   ├─> Automatic deletion if configured
            │   │   └─> Add "Delete Screenshots" button
            │   ├─> Stop runthrough timer
            │   └─> Start scav cooldown timer if was Scav/PVE raid
            └─> TimersManager
                ├─> Stop raid timer
                ├─> Stop runthrough timer
                └─> Start scav cooldown timer if applicable

"Init: pstrGameVersion:" (fully back to menu)
    └─> Emit ExitedPostRaidMenus event
        └─> MainBlazorUI plays "air_filter_off" sound if has air filter
```

### 6. Quest Status Workflow

```
EFT Log: "Got notification | ChatMessageReceived"
    └─> Parse JSON for message type
    └─> If TaskStarted/TaskFailed/TaskFinished:
        ├─> Emit TaskModified event
        ├─> Emit specific event (TaskStarted/TaskFailed/TaskFinished)
        └─> MainBlazorUI
            ├─> Find task in TarkovDev.Tasks list
            ├─> Log message with task name
            └─> Update TarkovTracker if valid token
                ├─> SetTaskStarted() for restarting failed tasks
                ├─> SetTaskFailed() for failed tasks
                └─> SetTaskComplete() for finished tasks
                    └─> Also mark dependent tasks as failed if applicable
```

### 7. Flea Market Workflow

```
EFT Log: "Got notification | ChatMessageReceived" (Flea Market type)
    └─> Parse message templateId

If "5bdabfb886f7743e152e867e 0" (Item Sold):
    └─> Emit FleaSold event
        └─> MainBlazorUI
            ├─> Record sale in Stats database
            ├─> Find item details in TarkovDev.Items
            ├─> Format currency/items received
            └─> Log "{buyer} purchased {count} {item} for {price}"

If "5bdabfe486f7743e1665df6e 0" (Offer Expired):
    └─> Emit FleaOfferExpired event
        └─> MainBlazorUI
            ├─> Find item in TarkovDev.Items
            └─> Log "Your offer for {item} (x{count}) expired"
```

### 8. Group/Squad Workflow

```
Group Invite Accepted:
    └─> "Got notification | GroupMatchInviteAccept"
        └─> Emit GroupInviteAccept event
            └─> MainBlazorUI logs "{nickname} ({side} {level}) accepted group invite"

Group Raid Settings:
    └─> "Got notification | GroupMatchRaidSettings"
        └─> Emit GroupRaidSettings event
            └─> GroupManager clears current group (ready for new ready status)

Group Member Ready:
    └─> "Got notification | GroupMatchRaidReady"
        └─> Emit GroupMemberReady event
            ├─> GroupManager updates member status
            └─> MainBlazorUI logs "{nickname} ({side} {level}) ready"

Group User Leave:
    └─> "Got notification | GroupMatchUserLeave"
        └─> Emit GroupUserLeave event
            ├─> GroupManager removes member (if not "You")
            └─> MainBlazorUI logs "{nickname} left the group"

Group Disbanded:
    └─> "Got notification | GroupMatchWasRemoved"
        └─> Emit GroupDisbanded event
            └─> GroupManager clears all members

Matching Aborted or Game Started:
    └─> GroupManager marks group as stale
```

## Data Synchronization Workflows

### 9. TarkovDev API Update Workflow

```
Initial Load or Timer (every 20 minutes):
    └─> TarkovDev.UpdateApiData()
        ├─> GetTasks() - GraphQL query for quests
        ├─> GetMaps() - GraphQL query for maps
        ├─> GetItems() - GraphQL query for items
        ├─> GetTraders() - GraphQL query for traders
        └─> GetHideout() - GraphQL query for hideout stations
        
After Update:
    └─> Log "Retrieved data from tarkov.dev: X items, Y maps, etc."
```

### 10. TarkovTracker Profile Workflow

```
Profile Change Detected:
    └─> TarkovTracker.SetProfile(profileId)
        ├─> Check if profile already active
        ├─> Get token for new profile
        ├─> If token exists and different:
        │   └─> TestToken(token)
        │       ├─> API call to /token endpoint
        │       ├─> Check for "WP" permission
        │       ├─> If valid:
        │       │   ├─> Set ValidToken = true
        │       │   ├─> GetProgress() to fetch quest progress
        │       │   └─> Emit TokenValidated event
        │       └─> If invalid:
        │           ├─> Set ValidToken = false
        │           └─> Emit TokenInvalid event
        └─> Return progress

Quest Update:
    └─> TarkovTracker.SetTaskStatus(questId, status)
        ├─> API call to POST /progress/task/{id}
        ├─> Update local Progress cache
        └─> Handle rate limiting (15 req/min)
```

### 11. Statistics Recording Workflow

```
Raid Started:
    └─> Stats.AddRaid(RaidInfoEventArgs)
        └─> INSERT INTO raids (profile_id, map, raid_type, queue_time, raid_id)

Flea Sale:
    └─> Stats.AddFleaSale(FleaSoldMessageLogContent, Profile)
        └─> INSERT INTO flea_sales (profile_id, item_id, buyer, count, currency, price)

Stats Retrieval:
    ├─> Stats.GetTotalSales(currency) - SUM(price) for currency
    ├─> Stats.GetTotalRaids(mapNameId) - COUNT raids for map
    └─> Stats.GetTotalRaidsPerMap(raidType) - COUNT raids grouped by map
```

### 12. Remote Control Workflow (tarkov.dev)

```
User Sets Remote ID in Settings:
    └─> Properties.Settings.Default.remoteId = "{id}"

Map Navigation:
    └─> SocketClient.NavigateToMap(map)
        ├─> Create WebSocket connection to wss://socket.tarkov.dev?sessionid={id}-tm
        ├─> Build JSON message: { type: "command", data: { type: "map", value: "{map}" } }
        └─> Send message and close connection

Player Position Update:
    └─> SocketClient.UpdatePlayerPosition(PlayerPositionEventArgs)
        ├─> Create WebSocket connection
        ├─> Build JSON message with position and rotation
        └─> Send message and close connection
```

## Error Handling Workflows

### 13. Exception Handling

```
Exception in GameWatcher:
    └─> Emit ExceptionThrown event
        └─> MainBlazorUI
            └─> MessageLog.AddMessage("Error {context}: {message}")

Exception in TarkovTracker:
    ├─> Rate limit (429): Throw "Rate limited by Tarkov Tracker API"
    ├─> Unauthorized (401): Call InvalidTokenException()
    │   ├─> Set ValidToken = false
    │   ├─> Emit TokenInvalid event
    │   └─> Throw exception
    └─> Other errors: Throw with descriptive message

Exception in SocketClient:
    └─> Emit ExceptionThrown event
        └─> MainBlazorUI logs error
```

### 14. Log File Rotation Handling

```
New Log File Created (application.log or notifications.log):
    └─> LogFileCreateWatcher detects creation
        └─> StartNewMonitor(path)
            ├─> Stop existing monitor if running
            ├─> Create new LogMonitor
            ├─> Subscribe to events
            ├─> Start monitoring
            └─> Store in Monitors dictionary
```

## Timer Workflows

### 15. In-Raid Timers

```
Raid Started:
    └─> TimersManager
        ├─> Start Raid Timer (every 1 second)
        │   └─> Increment TimeInRaidTime
        │       └─> Emit RaidTimerChanged event
        └─> Start Runthrough Timer (every 1 second)
            └─> Decrement RunThroughRemainingTime
                ├─> If > 0: Emit RunThroughTimerChanged event
                └─> If = 0: Stop timer

Raid Ended:
    └─> TimersManager
        ├─> Stop Raid Timer
        ├─> Stop Runthrough Timer
        └─> If was Scav/PVE:
            └─> Start Scav Cooldown Timer (every 1 second)
                └─> Decrement ScavCooldownTime
                    ├─> If > 0: Emit ScavCooldownTimerChanged event
                    └─> If = 0:
                        ├─> Stop timer
                        └─> Reset to ScavCooldownSeconds()
```

### 16. Background Tasks

```
Process Monitor Timer (every 30 seconds):
    └─> UpdateProcess()
        ├─> Check if EFT process running
        ├─> If started: Emit GameStarted event
        └─> If stopped: Clear process reference

Update Check Timer (every 24 hours):
    └─> UpdateCheck.CheckForNewVersion()
        ├─> Query GitHub API for latest release
        ├─> Compare versions
        └─> If newer: Emit NewVersion event

TarkovDev Auto-Update Timer (every 20 minutes):
    └─> If user active in last 5 minutes:
        └─> UpdateApiData()
```

## User Interaction Workflows

### 17. Settings Changes

```
User Changes Setting:
    └─> Properties.Settings.Default.PropertyChanged event
        ├─> If stayOnTop changed:
        │   └─> Update window TopMost property
        └─> If customLogsPath changed:
            └─> Update GameWatcher.LogsPath
                └─> Restart log monitoring with new path
```

### 18. Read Past Logs Feature

```
User Clicks "Read Past Logs":
    └─> Show breakpoint selection dialog
        ├─> GetLogBreakpoints(profileId) to get available breakpoints
        ├─> User selects breakpoint
        └─> ProcessLogsFromBreakpoint(breakpoint)
            ├─> Set ReadingPastLogs = true
            ├─> For each log folder from breakpoint onward:
            │   └─> ProcessLogs(target, profiles)
            │       ├─> Read entire log files
            │       ├─> Match log entries in date range
            │       └─> Call GameWatcher_NewLogData for each entry
            └─> Updates quest progress in TarkovTracker
```

## Summary

The TarkovMonitor workflow is event-driven with clear separation of concerns:

1. **Log Monitoring**: Continuous file watching and parsing
2. **Event Emission**: GameWatcher translates log patterns to typed events
3. **Event Handling**: MainBlazorUI coordinates responses (sounds, API calls, UI updates)
4. **External Integration**: Async API calls to TarkovTracker and TarkovDev
5. **Local Persistence**: SQLite for statistics, settings for configuration
6. **UI Updates**: Blazor components react to service state changes via DI

The architecture allows for easy extension of new features by:
- Adding new regex patterns in GameWatcher
- Creating new events
- Subscribing to events in MainBlazorUI
- Adding corresponding API integrations
