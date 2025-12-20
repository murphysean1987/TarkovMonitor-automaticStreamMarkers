# YouTube Live Streaming Integration Research

## Executive Summary

This document outlines the research and recommendations for integrating automatic stream markers into TarkovMonitor for YouTube Live Streaming. The integration would automatically create stream markers at key game events (raid start, raid end, task completion, etc.) during live streams.

## YouTube Live Streaming API Overview

### YouTube Live Streaming API v3

The YouTube Live Streaming API enables developers to:
- Create, update, and manage live broadcasts
- Insert cue points and markers during live streams
- Access broadcast metadata and statistics

**Documentation**: https://developers.google.com/youtube/v3/live/docs

### Key Endpoints for Stream Markers

#### 1. LiveBroadcasts
- **Purpose**: Manage live broadcast resources
- **Endpoint**: `https://www.googleapis.com/youtube/v3/liveBroadcasts`
- **Key Methods**:
  - `list`: Retrieve broadcast information
  - `insert`: Create a new broadcast
  - `update`: Modify broadcast settings
  - `transition`: Change broadcast status (testing, live, complete)

#### 2. LiveCuePoints (Stream Markers)
- **Purpose**: Insert cue points/markers into a live broadcast
- **Endpoint**: `https://www.googleapis.com/youtube/v3/liveBroadcasts/cuepoint`
- **Method**: POST
- **Parameters**:
  - `id`: Broadcast ID
  - `part`: Properties to include (snippet, etc.)
  - Request Body:
    ```json
    {
      "settings": {
        "cueType": "ad",  // or "sponsor"
        "durationSecs": 60,
        "offsetTimeMs": 0,
        "walltime": "2024-01-01T12:00:00.000Z"
      },
      "snippet": {
        "description": "Raid Started - Customs"
      }
    }
    ```

**Note**: The YouTube API doesn't have explicit "stream marker" endpoints like Twitch. However, cue points can serve a similar purpose and appear in the stream timeline.

## Authentication: OAuth 2.0

YouTube API requires OAuth 2.0 authentication following Google's identity platform.

### OAuth 2.0 Flow for Desktop Applications

#### 1. Application Registration
- Register app in Google Cloud Console
- Enable YouTube Data API v3
- Create OAuth 2.0 credentials (Desktop app type)
- Obtain Client ID and Client Secret
- Configure redirect URI: `http://localhost:PORT` or `urn:ietf:wg:oauth:2.0:oob` (for manual copy-paste)

#### 2. Authorization Code Flow

**Step 1: Authorization Request**
```
https://accounts.google.com/o/oauth2/v2/auth?
  client_id={CLIENT_ID}&
  redirect_uri=http://localhost:PORT&
  response_type=code&
  scope=https://www.googleapis.com/auth/youtube https://www.googleapis.com/auth/youtube.force-ssl&
  access_type=offline&
  prompt=consent
```

**Step 2: User Authorization**
- User logs in to Google account
- Grants permissions to the application
- Redirected to redirect_uri with authorization code

**Step 3: Token Exchange**
```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

code={AUTHORIZATION_CODE}&
client_id={CLIENT_ID}&
client_secret={CLIENT_SECRET}&
redirect_uri=http://localhost:PORT&
grant_type=authorization_code
```

**Response**:
```json
{
  "access_token": "ya29.a0AfH6...",
  "expires_in": 3599,
  "refresh_token": "1//0gH...",
  "scope": "https://www.googleapis.com/auth/youtube...",
  "token_type": "Bearer"
}
```

**Step 4: Using Access Token**
```http
GET https://www.googleapis.com/youtube/v3/liveBroadcasts?part=id,snippet&mine=true
Authorization: Bearer {ACCESS_TOKEN}
```

**Step 5: Refreshing Tokens**
```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

refresh_token={REFRESH_TOKEN}&
client_id={CLIENT_ID}&
client_secret={CLIENT_SECRET}&
grant_type=refresh_token
```

### Required Scopes

- `https://www.googleapis.com/auth/youtube` - Manage YouTube account
- `https://www.googleapis.com/auth/youtube.force-ssl` - Manage YouTube data (required for live streaming)

## Comparison with Existing TarkovMonitor Auth Patterns

### Current Implementations

#### TarkovTracker
```csharp
// Simple API Key (Bearer Token)
- Token Type: Static API key
- Storage: Per-profile in app settings (JSON serialized dictionary)
- Validation: Single test endpoint
- Header: Authorization: Bearer {token}
- Expiration: No automatic expiration
- Refresh: Manual token replacement only
```

#### TarkovDev
```csharp
// No authentication
- GraphQL API: Open access
- WebSocket: Session ID only
- Data submission: No auth required
```

### YouTube OAuth 2.0 Differences

| Aspect | TarkovTracker | YouTube OAuth 2.0 |
|--------|---------------|-------------------|
| **Token Type** | Static API key | Dynamic access token + refresh token |
| **Expiration** | No expiration | Access token expires in ~60 minutes |
| **Refresh** | Manual replacement | Automatic refresh using refresh token |
| **User Flow** | Copy-paste from website | OAuth browser redirect flow |
| **Storage** | Single string | Access token + refresh token + expiry |
| **Security** | API key in plaintext | Tokens should be encrypted |
| **Scopes** | Implicit permissions | Explicit scope declaration |

## Recommended Implementation Approach

### 1. OAuth 2.0 Library Integration

**Recommended Library**: `Google.Apis.Auth` NuGet package

```csharp
// Add to TarkovMonitor.csproj
<PackageReference Include="Google.Apis.Auth" Version="1.68.0" />
<PackageReference Include="Google.Apis.YouTube.v3" Version="1.68.0.3393" />
```

This official Google library handles:
- OAuth 2.0 flow
- Token storage and refresh
- Request signing
- Error handling

### 2. Create YouTubeAuth.cs Class

Pattern similar to `TarkovTracker.cs` but with OAuth support:

```csharp
namespace TarkovMonitor
{
    internal class YouTubeAuth
    {
        private static UserCredential? credential;
        private static YouTubeService? service;
        
        // OAuth 2.0 Configuration
        private static readonly ClientSecrets clientSecrets = new ClientSecrets
        {
            ClientId = "{CLIENT_ID}",      // From Google Cloud Console
            ClientSecret = "{CLIENT_SECRET}" // From Google Cloud Console
        };
        
        private static readonly string[] scopes = new[] {
            YouTubeService.Scope.Youtube,
            YouTubeService.Scope.YoutubeForceSsl
        };
        
        public static async Task<bool> AuthenticateAsync()
        {
            try
            {
                // Token storage path
                string tokenPath = Path.Join(
                    Application.UserAppDataPath, 
                    "YouTubeTokens"
                );
                
                // Local HTTP server for OAuth callback
                credential = await GoogleWebAuthorizationBroker.AuthorizeAsync(
                    clientSecrets,
                    scopes,
                    "user",
                    CancellationToken.None,
                    new FileDataStore(tokenPath, true)
                );
                
                // Create YouTube service
                service = new YouTubeService(new BaseClientService.Initializer
                {
                    HttpClientInitializer = credential,
                    ApplicationName = "TarkovMonitor"
                });
                
                return true;
            }
            catch (Exception ex)
            {
                // Handle auth errors
                return false;
            }
        }
        
        public static async Task<string?> GetActiveBroadcastId()
        {
            if (service == null) return null;
            
            var request = service.LiveBroadcasts.List("id,snippet,status");
            request.BroadcastStatus = LiveBroadcastsResource.ListRequest.BroadcastStatusEnum.Active;
            request.Mine = true;
            
            var response = await request.ExecuteAsync();
            return response.Items?.FirstOrDefault()?.Id;
        }
        
        public static async Task<bool> InsertStreamMarker(string broadcastId, string description)
        {
            if (service == null) return false;
            
            try
            {
                var cuepoint = new Cuepoint
                {
                    Snippet = new CuepointSnippet
                    {
                        Description = description
                    },
                    Settings = new CuepointSettings
                    {
                        CueType = "sponsor", // or "ad"
                        DurationSecs = 0,
                        OffsetTimeMs = 0L,
                        Walltime = DateTime.UtcNow
                    }
                };
                
                var request = service.LiveBroadcasts.Cuepoint(cuepoint, broadcastId);
                request.Part = "snippet,settings";
                
                await request.ExecuteAsync();
                return true;
            }
            catch
            {
                return false;
            }
        }
        
        public static void Logout()
        {
            credential?.RevokeTokenAsync(CancellationToken.None);
            credential = null;
            service = null;
        }
    }
}
```

### 3. Token Storage Strategy

**Following TarkovMonitor Patterns**:

1. **Profile-Based Storage** (like TarkovTracker):
   ```csharp
   // Store per-profile YouTube credentials
   Dictionary<string, YouTubeCredential> youtubeTokens = new();
   
   public class YouTubeCredential
   {
       public string AccessToken { get; set; }
       public string RefreshToken { get; set; }
       public DateTime ExpiresAt { get; set; }
   }
   ```

2. **Settings Storage**:
   ```csharp
   // In Properties.Settings
   [UserScopedSetting]
   [DefaultSettingValue("{}")]
   public string youtubeTokens { get; set; }
   ```

3. **Encryption** (recommended):
   ```csharp
   // Use Windows Data Protection API (DPAPI)
   using System.Security.Cryptography;
   
   string encrypted = Convert.ToBase64String(
       ProtectedData.Protect(
           Encoding.UTF8.GetBytes(jsonTokens),
           null,
           DataProtectionScope.CurrentUser
       )
   );
   ```

### 4. UI Integration

Add settings page similar to TarkovTracker token configuration:

```
Settings Page:
- [Button] "Connect YouTube Account"
  └─> Opens browser for OAuth flow
  └─> Handles callback on localhost
  └─> Stores tokens
  
- [Button] "Test Connection"
  └─> Attempts to fetch active broadcast
  └─> Shows success/failure message
  
- [Button] "Disconnect YouTube"
  └─> Revokes tokens
  └─> Clears stored credentials

- [Checkbox] "Auto-create stream markers"
  └─> Enable/disable feature

- [Dropdown] "Marker Events"
  └─> Select which events trigger markers:
      - Raid Started
      - Raid Ended
      - Task Completed
      - Match Found
      - etc.
```

### 5. Event Integration

Add to `MainBlazorUI.cs`:

```csharp
// Subscribe to events
eft.RaidStarted += Eft_RaidStarted_YouTube;
eft.RaidEnded += Eft_RaidEnded_YouTube;
eft.TaskFinished += Eft_TaskFinished_YouTube;

private async void Eft_RaidStarted_YouTube(object? sender, RaidInfoEventArgs e)
{
    if (!Properties.Settings.Default.youtubeMarkersEnabled)
        return;
        
    var broadcastId = await YouTubeAuth.GetActiveBroadcastId();
    if (broadcastId == null)
        return;
        
    var mapName = e.RaidInfo.Map;
    var map = TarkovDev.Maps.Find(m => m.nameId == mapName);
    if (map != null) mapName = map.name;
    
    var description = $"Raid Started - {mapName} ({e.RaidInfo.RaidType})";
    
    var success = await YouTubeAuth.InsertStreamMarker(broadcastId, description);
    if (success)
    {
        messageLog.AddMessage($"YouTube marker created: {description}", "youtube");
    }
}
```

## Security Considerations

### 1. Client Secret Protection

**Problem**: Desktop apps cannot keep client secrets completely secure
**Mitigation**:
- Use "Desktop app" OAuth type (less sensitive than "Web app")
- Store secrets in compiled code (obfuscation)
- Consider using Google's public client flow (no client secret)
- Rotate client secrets periodically

### 2. Token Storage

**Recommendations**:
1. Encrypt tokens using Windows DPAPI (current user scope)
2. Store in user-specific AppData folder
3. Never log tokens to MessageLog or debug output
4. Clear tokens on application uninstall

### 3. Scope Minimization

Only request required scopes:
- `youtube.force-ssl` for live streaming control
- Avoid `youtube.upload` or other unnecessary permissions

### 4. Token Refresh Error Handling

```csharp
public static async Task<bool> EnsureValidToken()
{
    try
    {
        // Google.Apis.Auth handles automatic refresh
        await credential.RefreshTokenAsync(CancellationToken.None);
        return true;
    }
    catch (TokenResponseException ex)
    {
        if (ex.Error.Error == "invalid_grant")
        {
            // Refresh token expired or revoked
            // User must re-authenticate
            Logout();
            return false;
        }
        throw;
    }
}
```

## Implementation Checklist

### Phase 1: Basic OAuth Integration
- [ ] Add Google.Apis NuGet packages
- [ ] Register app in Google Cloud Console
- [ ] Create YouTubeAuth.cs class
- [ ] Implement OAuth 2.0 flow with local HTTP server
- [ ] Add token storage with encryption
- [ ] Create UI for authentication (Settings page)

### Phase 2: Stream Detection
- [ ] Add method to detect active YouTube broadcast
- [ ] Implement broadcast ID caching (avoid repeated API calls)
- [ ] Add broadcast status monitoring
- [ ] Handle "no active broadcast" scenario gracefully

### Phase 3: Marker Creation
- [ ] Implement InsertStreamMarker method
- [ ] Add event handlers for raid start/end
- [ ] Add event handlers for task completion
- [ ] Add event handlers for match found
- [ ] Add configurable event selection in settings
- [ ] Test marker creation and verify in YouTube Studio

### Phase 4: Chapter Generation
- [ ] Create YouTubeStreamRecorder class for event tracking
- [ ] Implement timestamp collection during stream
- [ ] Add chapter filtering logic (minimum gaps, event types)
- [ ] Implement chapter description builder
- [ ] Add video description update method
- [ ] Create UI for chapter preferences
- [ ] Test chapter generation on completed streams

### Phase 5: Kill Tracking Investigation
- [ ] Examine log files for kill-related data
- [ ] Search for session result/statistics entries
- [ ] Document kill data structure if found
- [ ] Implement KillDetected event if data available
- [ ] Add kill event parsing in GameWatcher
- [ ] Create kill-based markers and chapters
- [ ] Add kill tracking settings/preferences

### Phase 6: Error Handling & Polish
- [ ] Handle rate limiting (quota: 10,000 units/day)
- [ ] Add retry logic with exponential backoff
- [ ] Implement token refresh error handling
- [ ] Add user-facing error messages
- [ ] Add logging for debugging
- [ ] Test multi-profile scenarios
- [ ] Implement quota monitoring and warnings

### Phase 7: Testing & Documentation
- [ ] Test OAuth flow end-to-end
- [ ] Test token refresh after expiration
- [ ] Test with actual YouTube live stream
- [ ] Verify markers appear in YouTube Studio
- [ ] Verify chapters appear on VOD
- [ ] Test both features independently and together
- [ ] Document setup process for users
- [ ] Create troubleshooting guide

## Rate Limiting and Quotas

### YouTube API Quotas

**Daily Quota**: 10,000 units per day (default, can request increase)

**Operation Costs**:
- LiveBroadcasts.list: 1 unit
- LiveBroadcasts.cuepoint: 50 units

**Maximum Markers per Session**:
- 10,000 / 50 = 200 markers per day (if only creating markers)
- Realistically: ~150-180 markers accounting for broadcast checks

**Chapter Generation Costs**:
- Videos.list: 1 unit (get current description)
- Videos.update: 50 units (set new description)
- **Total per stream**: 51 units

**Combined Usage Scenarios**:

*Scenario 1: Markers Only (No Chapters)*
- Broadcast check: 1 unit
- 10 markers per stream: 500 units
- **Total**: ~500 units per stream
- **Daily capacity**: ~20 streams

*Scenario 2: Chapters Only (No Markers)*
- Chapter generation: 51 units per stream
- **Daily capacity**: ~196 streams (effectively unlimited)

*Scenario 3: Both Markers + Chapters (Recommended)*
- Broadcast check: 1 unit
- 5-10 selective markers: 250-500 units
- Chapter generation: 51 units
- **Total**: ~300-550 units per stream
- **Daily capacity**: ~18-33 streams

*Scenario 4: Heavy Usage (Markers + Chapters + Kills)*
- Broadcast check: 1 unit
- 15-20 markers (including kills): 750-1000 units
- Chapter generation: 51 units
- **Total**: ~800-1050 units per stream
- **Daily capacity**: ~9-12 streams

**Mitigation Strategies**:
- Cache active broadcast ID (refresh every 5 minutes)
- Only check for active broadcast on first marker attempt
- Implement local rate limiting (max 1 marker per minute per event type)
- Add user notification when approaching quota limit
- **Smart filtering**: Prioritize major events for markers, save all events for chapters
- **Kill batching**: Group multiple kills into single marker if enabled
- **Quota monitoring**: Track daily usage and warn user at 80% capacity

**Recommended Configuration**:
```
Default Settings:
- Live Markers: Major events only (Raid Start/End, Quest Complete)
- Chapters: All events (comprehensive timeline)
- Kill Markers: Disabled by default (to preserve quota)
- Kill Chapters: Enabled if data available
```

## Alternative Approaches

### 1. YouTube Data API v3 (Chapters)

~~Instead of cue points during live stream, create chapters after stream~~
**Now Recommended**: Use chapters IN ADDITION to markers:
- **Pros**: Better user experience, visible in YouTube player, lower quota cost
- **Cons**: Post-stream only, requires VOD processing
- **Use case**: Primary viewer-facing feature, complementary to live markers

### 2. YouTube Studio API (Limited Access)

Direct integration with YouTube Studio features:
- **Pros**: More native marker support
- **Cons**: Requires special API access, not generally available

### 3. Third-Party Services

Services like Restream or StreamElements:
- **Pros**: Multi-platform support
- **Cons**: Additional dependencies, subscription costs

## Automated YouTube Chapters

### Overview

YouTube chapters are timestamps in video descriptions that allow viewers to jump to specific sections. Unlike stream markers (cue points), chapters are:
- **Visible in the player timeline** - Shown as segments with titles
- **User-facing** - Appear in the progress bar and description
- **Post-stream** - Applied after the stream ends to the VOD
- **More discoverable** - Improve viewer experience and engagement

### YouTube Chapters API

**Endpoint**: `https://www.googleapis.com/youtube/v3/videos`
**Method**: PUT (update)

**Implementation Approach**:
1. Collect timestamps during the stream (similar to markers)
2. After stream ends, update video description with chapter timestamps
3. Format: `0:00 Intro\n5:32 Raid Started - Customs\n15:45 Task Completed`

**Chapter Requirements**:
- First chapter must start at 0:00
- Minimum 3 chapters required
- Minimum 10 seconds between chapters
- Maximum 100 characters per chapter title

### Dual Implementation: Markers + Chapters

**Recommended Architecture**:

```csharp
public class YouTubeStreamRecorder
{
    private List<StreamEvent> eventTimestamps = new();
    private DateTime streamStartTime;
    private string? broadcastId;
    
    public class StreamEvent
    {
        public DateTime Timestamp { get; set; }
        public TimeSpan RelativeTime { get; set; }
        public string EventType { get; set; }
        public string Description { get; set; }
    }
    
    // During stream: Create markers (if enabled)
    public async Task RecordEvent(string eventType, string description)
    {
        var now = DateTime.UtcNow;
        var relativeTime = now - streamStartTime;
        
        var streamEvent = new StreamEvent
        {
            Timestamp = now,
            RelativeTime = relativeTime,
            EventType = eventType,
            Description = description
        };
        
        eventTimestamps.Add(streamEvent);
        
        // Create live marker if enabled
        if (Settings.EnableLiveMarkers && broadcastId != null)
        {
            await YouTubeAuth.InsertStreamMarker(broadcastId, description);
        }
    }
    
    // After stream: Generate chapters (if enabled)
    public async Task GenerateChapters(string videoId)
    {
        if (!Settings.EnableAutoChapters)
            return;
            
        // Filter events for chapters (avoid too many)
        var chapterEvents = FilterEventsForChapters(eventTimestamps);
        
        // Build chapter description
        var chapters = BuildChapterDescription(chapterEvents);
        
        // Update video description
        await UpdateVideoDescription(videoId, chapters);
    }
    
    private List<StreamEvent> FilterEventsForChapters(List<StreamEvent> events)
    {
        // Keep major events, remove spam
        var filtered = events.Where(e => 
            e.EventType == "RaidStarted" ||
            e.EventType == "RaidEnded" ||
            e.EventType == "TaskCompleted" ||
            e.EventType == "Kill" // if available
        ).ToList();
        
        // Ensure minimum 10 seconds between chapters
        var result = new List<StreamEvent>();
        StreamEvent? lastEvent = null;
        
        foreach (var evt in filtered)
        {
            if (lastEvent == null || 
                (evt.RelativeTime - lastEvent.RelativeTime).TotalSeconds >= 10)
            {
                result.Add(evt);
                lastEvent = evt;
            }
        }
        
        return result;
    }
    
    private string BuildChapterDescription(List<StreamEvent> events)
    {
        var sb = new StringBuilder();
        
        // First chapter at 0:00
        sb.AppendLine("0:00 Stream Start");
        
        foreach (var evt in events)
        {
            var timestamp = FormatTimestamp(evt.RelativeTime);
            var title = evt.Description;
            
            // Truncate to 100 characters
            if (title.Length > 100)
                title = title.Substring(0, 97) + "...";
                
            sb.AppendLine($"{timestamp} {title}");
        }
        
        return sb.ToString();
    }
    
    private string FormatTimestamp(TimeSpan time)
    {
        if (time.TotalHours >= 1)
            return $"{(int)time.TotalHours}:{time.Minutes:D2}:{time.Seconds:D2}";
        else
            return $"{time.Minutes}:{time.Seconds:D2}";
    }
}
```

### Settings Configuration

Add to Settings page:

```
YouTube Stream Features:
┌─────────────────────────────────────┐
│ ☑ Enable Live Stream Markers       │
│ ☑ Enable Auto-Generated Chapters   │
│                                     │
│ Chapter Events (select which to    │
│ include):                           │
│ ☑ Raid Started                     │
│ ☑ Raid Ended                       │
│ ☑ Task Completed                   │
│ ☑ Match Found                      │
│ ☑ Kills (if available)             │
│ ☐ Flea Market Sales                │
│ ☐ Group Events                     │
└─────────────────────────────────────┘
```

### Implementation Phases

**Phase 1: During Stream (Live Markers)**
1. Detect stream start
2. Record all relevant events with timestamps
3. Create live markers if enabled (respecting quota)

**Phase 2: After Stream (Chapters)**
1. Detect stream end (broadcast status change)
2. Filter events for chapter generation
3. Build chapter description string
4. Retrieve video ID from broadcast
5. Update video description with chapters
6. Validate chapters meet YouTube requirements

**Phase 3: User Options**
1. Allow users to enable/disable each feature independently
2. Provide preview of chapter list before applying
3. Allow manual editing of chapter titles
4. Option to prepend or append to existing description

### API Quota Impact

**Chapter Generation**:
- Videos.list: 1 unit (to get current description)
- Videos.update: 50 units (to set new description)
- **Total per stream**: 51 units

**Combined Usage** (Markers + Chapters):
- ~5-10 markers per stream: 250-500 units
- Chapter generation: 51 units
- **Total**: ~300-550 units per stream
- **Daily capacity**: ~18-33 streams per day

## Kill Tracking Research

### Current Log Analysis

Based on code analysis, TarkovMonitor currently tracks:
- Raid start/end times
- Map information
- Profile type (PMC/Scav)
- Quest completions
- Flea market transactions
- Group/squad events

**Kill Data in Logs**:

The codebase includes `LoadoutItemPropertiesDogtag` class (LogMessageTypes.cs, lines 113-125) which contains:
```csharp
public class LoadoutItemPropertiesDogtag
{
    public string AccountId { get; set; }
    public string ProfileId { get; set; }
    public string Side { get; set; }
    public int Level { get; set; }
    public string Time { get; set; }           // Time of kill
    public string Status { get; set; }
    public string KillerAccountId { get; set; }
    public string KillerProfileId { get; set; }
    public string KillerName { get; set; }
    public string WeaponName { get; set; }
}
```

### Kill Detection Methods

#### Method 1: Dogtag Detection (Indirect)

**When dogtags are looted**, this information becomes available, but:
- ❌ Only captures PMC kills (not Scavs)
- ❌ Only if dogtag is looted
- ❌ Doesn't provide real-time kill notification
- ✅ Includes kill timestamp, weapon, and victim info

**Not recommended** for real-time stream markers as it requires post-raid inventory analysis.

#### Method 2: End-of-Raid Screen Parsing

**The user mentions**: "there is a screen at the end of a raid that shows all kills and their time"

**Investigation needed**:
1. Does EFT log this data when the end-of-raid screen is shown?
2. If so, what log entry pattern contains this information?
3. Format: JSON with kill list and timestamps?

**Potential log patterns to search for**:
- Session results/statistics
- Raid summary/report
- Kill list notifications
- Experience gain breakdown (which includes kill XP)

**Example search pattern**:
```csharp
// In GameWatcher.cs, add new regex patterns:
if (eventLine.Contains("session result") || 
    eventLine.Contains("raid statistics") ||
    eventLine.Contains("session statistics"))
{
    // Parse JSON for kill data
    var sessionData = jsonNode?.AsObject().Deserialize<SessionResultLogContent>();
    if (sessionData?.kills != null)
    {
        foreach (var kill in sessionData.kills)
        {
            KillDetected?.Invoke(this, new KillEventArgs {
                VictimName = kill.name,
                WeaponName = kill.weapon,
                TimeOffset = kill.time,
                RaidInfo = raidInfo
            });
        }
    }
}
```

#### Method 3: Real-Time Kill Detection (Unknown)

**Requires investigation**:
- Do logs contain real-time kill notifications?
- Pattern: "You killed {name}" or similar?
- JSON structure for kill events?

**If available**, this would be ideal for:
- ✅ Real-time stream markers
- ✅ Immediate YouTube cue points
- ✅ Chapter generation with kill timestamps

### Recommended Investigation Steps

To determine if kill tracking is feasible:

1. **Capture Sample Logs**:
   ```
   - Play a raid and get several kills
   - Check application.log after raid
   - Check notifications.log after raid
   - Search for patterns containing kill/death/frag data
   ```

2. **Search for Keywords**:
   ```
   - "kill", "killed", "eliminated", "neutralized"
   - "experience", "exp" (kill XP entries)
   - "session", "statistics", "report", "result"
   - "dogtag" (for indirect detection)
   ```

3. **Analyze End-of-Raid Log Entries**:
   ```
   - Compare logs before and after viewing end-screen
   - Look for JSON blobs with kill arrays
   - Check for timestamp data matching in-raid times
   ```

4. **Test Data Structure**:
   ```csharp
   // Expected structure (if exists):
   public class SessionResultLogContent : JsonLogContent
   {
       public SessionStatistics statistics { get; set; }
   }
   
   public class SessionStatistics
   {
       public List<KillData> kills { get; set; }
       public int experience { get; set; }
       public float timeInRaid { get; set; }
   }
   
   public class KillData
   {
       public string name { get; set; }      // Victim name
       public string weapon { get; set; }     // Weapon used
       public string bodyPart { get; set; }   // Headshot, thorax, etc.
       public float time { get; set; }        // Time offset from raid start
       public int distance { get; set; }      // Distance in meters
       public int level { get; set; }         // Victim level
       public string side { get; set; }       // PMC/Scav/Boss
   }
   ```

### Implementation If Kill Data Is Available

**During Raid** (if real-time kills are logged):
```csharp
if (eventLine.Contains("Got notification | Kill") || 
    eventLine.Contains("You killed"))
{
    var killData = ParseKillEvent(jsonNode);
    Emit KillDetected event
    → Create stream marker: "Kill - {victimName} with {weapon}"
    → Record for chapter generation
}
```

**After Raid** (if end-screen provides kill summary):
```csharp
if (eventLine.Contains("session result"))
{
    var sessionData = ParseSessionResult(jsonNode);
    foreach (var kill in sessionData.kills)
    {
        // Retroactively create chapter entries
        eventTimestamps.Add(new StreamEvent {
            RelativeTime = raidStartTime.Add(kill.timeOffset),
            Description = $"Kill - {kill.name} ({kill.weapon})"
        });
    }
}
```

**Settings Configuration**:
```
Kill Tracking:
┌─────────────────────────────────────┐
│ ☑ Track Kills                      │
│   ○ Real-time (if available)       │
│   ● End-of-raid summary            │
│                                     │
│ Kill Marker Format:                 │
│ [Dropdown] "Kill - {name}"         │
│   Options:                          │
│   - "Kill - {name}"                │
│   - "Kill - {name} ({weapon})"     │
│   - "Kill with {weapon}"           │
│   - "PMC/Scav Kill"                │
│                                     │
│ ☑ Include in stream markers        │
│ ☑ Include in chapters              │
│ ☐ Minimum kills for chapter (3+)  │
└─────────────────────────────────────┘
```

### Limitations and Considerations

**If Kill Data Is NOT in Logs**:
- Cannot implement real-time kill tracking
- Cannot create kill-based stream markers
- Cannot generate kill timestamps for chapters
- **Alternative**: Track "Raid Ended" with final kill count if available in session stats

**If Kill Data IS in Logs**:
- ✅ Real-time stream markers for each kill
- ✅ Accurate chapter timestamps
- ✅ Enhanced viewer engagement
- ⚠️ May create too many markers (quota management)
- ⚠️ Need filtering (minimum kills, time gaps)

**Quota Management for Kills**:
- Limit to major kills (PMC kills only, not scavs)
- Batch kills within 1-minute windows
- Set maximum kills per raid for markers (e.g., top 5)
- Always include in chapter generation (post-raid, no quota impact)

### Next Steps for Kill Tracking

1. **Verify log data availability**:
   - Play test raid with kills
   - Examine log files for kill-related entries
   - Document exact log patterns and JSON structure

2. **If kill data found**:
   - Add `KillDetected` event to GameWatcher
   - Create `KillEventArgs` class
   - Implement parsing in `GameWatcher_NewLogData`
   - Add event handler in MainBlazorUI
   - Integrate with YouTubeStreamRecorder

3. **If kill data NOT found**:
   - Document limitation
   - Suggest alternative: Total kills in end-of-raid summary
   - Consider feature request to BSG for better logging

## Recommendations

### Immediate Implementation (Phase 1-3)

1. **Use Google.Apis.Auth library** - Well-maintained, handles OAuth complexity
2. **Desktop app OAuth flow** - Appropriate for Windows Forms app
3. **Encrypted token storage** - DPAPI for Windows user-level encryption
4. **Profile-based credentials** - Consistent with TarkovTracker pattern
5. **Event-driven markers** - Subscribe to existing GameWatcher events

### Future Enhancements

1. **Multi-platform support** - Abstract streaming service interface
2. **Twitch integration** - Similar OAuth flow, better marker support
3. **Chapter generation** - Post-process VODs with timestamp chapters
4. **Stream highlights** - Automatic clip creation for key moments
5. **OBS integration** - Direct scene switching based on game events

## Conclusion

YouTube Live Streaming integration is feasible and aligns well with TarkovMonitor's existing architecture. The OAuth 2.0 flow is more complex than the current API key approach but is well-supported by Google's official libraries. The event-driven nature of TarkovMonitor makes it ideal for automatic stream marker creation.

### Dual-Feature Implementation

**Live Stream Markers**:
- Real-time cue points during broadcast
- Visible in YouTube Studio analytics
- Cost: 50 units per marker
- Use case: Live monitoring, sponsor segments

**Automated Chapters**:
- Post-stream VOD enhancement
- Visible in player timeline for viewers
- Cost: 51 units per stream (one-time)
- Use case: Improved viewer navigation and retention

**Combined Approach** (Recommended):
- Enable both features independently
- Collect events during stream for both purposes
- Create live markers for major events (quota-conscious)
- Generate comprehensive chapters after stream (all events)
- Result: Best of both worlds with manageable quota usage

### Kill Tracking Status

**Investigation Required**:
The presence of kill data in EFT logs needs verification through:
1. Log file examination during/after raids with kills
2. Analysis of session result/statistics entries
3. Documentation of any kill-related JSON structures

**If Kill Data Available**:
- ✅ Real-time kill markers during stream
- ✅ Kill timestamps in chapter generation
- ✅ Enhanced viewer engagement
- ⚠️ Requires quota management (filter/batch kills)

**If Kill Data NOT Available**:
- Focus on other rich events (raids, quests, market)
- Document limitation for future enhancement
- Consider total kill count from end-of-raid stats

**Key Differences from Existing Auth**:
- OAuth 2.0 vs static API keys
- Token refresh mechanism required
- Browser-based authentication flow
- More complex token storage

**Best Practices to Follow**:
- Encrypt stored tokens
- Handle token expiration gracefully
- Minimize API quota usage through caching
- Provide clear user feedback
- Follow existing TarkovMonitor patterns for consistency
- Allow independent enabling of markers and chapters
- Implement smart filtering for quota management

**Next Steps**:
1. Create Google Cloud project and obtain OAuth credentials
2. Implement YouTubeAuth.cs with Google.Apis library
3. Add OAuth flow UI to Settings page
4. Create YouTubeStreamRecorder class for event tracking
5. Integrate with existing GameWatcher events
6. Investigate kill data availability in logs
7. Implement chapter generation logic
8. Test with live YouTube broadcasts
9. Validate chapters on VOD playback
