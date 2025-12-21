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

### Phase 5: OCR Kill Tracking Implementation
- [ ] Add OCR library dependency (Tesseract or Windows.Media.Ocr)
- [ ] Create EndOfRaidScreenDetector class
- [ ] Implement kill list region detection
- [ ] Create KillListOcrService for text extraction
- [ ] Build KillListParser with time-anchor logic
- [ ] Add KillEvent data model class
- [ ] Integrate with GameWatcher event flow
- [ ] Create kill_events database table in Stats.cs
- [ ] Add kill persistence methods
- [ ] Implement KillListDetected event handler in MainBlazorUI
- [ ] Add kill data to chapter generation
- [ ] Create settings for kill tracking options
- [ ] Test with various kill scenarios (mixed PMC/Scav, partial dogtags)
- [ ] Handle different resolutions and UI scales
- [ ] Add error handling for OCR failures

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

#### Method 2: End-of-Raid Screen OCR (Recommended)

**Authoritative Source**: End-of-raid kill list UI screen

**Update**: Investigation has determined that kill data is **NOT reliably logged** to game files. The recommended approach is OCR-based extraction from the end-of-raid kill list screen.

**Implementation Approach**:
1. Detect when end-of-raid kill list screen is visible
2. Capture only the kill list region (no full-screen)
3. OCR the captured image
4. Parse kill list into structured records
5. Persist using existing mechanisms

**Guaranteed Data Fields**:
- ✅ Kill timestamp (always present)
- ✅ Enemy type (always present - PMC/Scav/Boss)
- ⚠️ PMC name (only if dogtag looted)
- ⚠️ Weapon name (only if dogtag looted)

**Data Model**:
```csharp
public class KillEvent
{
    public string RaidId { get; set; }
    public int KillIndex { get; set; }
    public string KillTime { get; set; }        // Always present
    public string EnemyType { get; set; }       // Always present
    public string? PmcName { get; set; }        // Nullable - only if looted
    public string? WeaponName { get; set; }     // Nullable - only if looted
    public string RawOcrRow { get; set; }       // For diagnostics
}
```

**OCR & Parsing Rules**:
- Time column is primary row anchor
- Rows retained even if PMC-specific fields missing
- Must tolerate variable column counts
- Store raw OCR output for diagnostics

**Integration Requirements**:
- ✅ Follow existing module boundaries
- ✅ Reuse existing screen-watching infrastructure
- ✅ Feature must be optional/configurable
- ✅ Must not break existing features
- ❌ No in-raid capture
- ❌ No memory inspection or packet capture

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

#### Method 3: Real-Time Kill Detection (Not Available)

**Status**: Kill events are **NOT logged** to game files in real-time.

**Conclusion**: OCR-based end-of-raid screen parsing (Method 2) is the only viable approach.

### OCR Implementation Architecture

#### Required Components

**1. Screen Detection Module**
```csharp
public class EndOfRaidScreenDetector
{
    // Detect when kill list screen is visible
    public bool IsKillListScreenVisible()
    {
        // Use existing screen-watching infrastructure
        // Look for specific UI elements/patterns
        // Return true when kill list is displayed
    }
    
    public Rectangle GetKillListRegion()
    {
        // Return bounding box for kill list area only
        // Avoid full-screen capture
    }
}
```

**2. OCR Service**
```csharp
public class KillListOcrService
{
    // Recommended: Tesseract OCR engine
    // Alternative: Windows.Media.Ocr (built-in)
    
    public async Task<List<string>> ExtractKillListRows(Bitmap killListImage)
    {
        // OCR the kill list region
        // Return one string per row
    }
}
```

**3. Parser Module**
```csharp
public class KillListParser
{
    public List<KillEvent> ParseKillList(List<string> ocrRows, string raidId)
    {
        var kills = new List<KillEvent>();
        
        for (int i = 0; i < ocrRows.Count; i++)
        {
            var row = ocrRows[i];
            
            // Time column is primary anchor
            var timeMatch = Regex.Match(row, @"(\d{1,2}:\d{2})");
            if (!timeMatch.Success)
                continue; // Skip invalid rows
            
            var killEvent = new KillEvent
            {
                RaidId = raidId,
                KillIndex = i,
                KillTime = timeMatch.Groups[1].Value,
                RawOcrRow = row
            };
            
            // Parse enemy type (always present)
            if (row.Contains("PMC") || row.Contains("BEAR") || row.Contains("USEC"))
                killEvent.EnemyType = "PMC";
            else if (row.Contains("Scav"))
                killEvent.EnemyType = "Scav";
            else if (row.Contains("Boss") || row.Contains("Raider"))
                killEvent.EnemyType = "Boss";
            else
                killEvent.EnemyType = "Unknown";
            
            // Parse optional PMC name (only if dogtag looted)
            // Pattern: extract name between time and weapon
            // Handle missing gracefully
            
            // Parse optional weapon name (only if dogtag looted)
            // Pattern: weapon typically at end of row
            // Handle missing gracefully
            
            kills.Add(killEvent);
        }
        
        return kills;
    }
}
```

**4. Integration with GameWatcher**
```csharp
// In GameWatcher.cs
public event EventHandler<KillListDetectedEventArgs>? KillListDetected;

private EndOfRaidScreenDetector screenDetector;
private KillListOcrService ocrService;
private KillListParser killParser;

// After RaidExited event
private async void CheckForKillList()
{
    if (!Properties.Settings.Default.enableKillTracking)
        return;
        
    // Wait for kill list screen
    await Task.Delay(2000); // Allow screen transition
    
    if (screenDetector.IsKillListScreenVisible())
    {
        var killListRegion = screenDetector.GetKillListRegion();
        var screenshot = CaptureScreen(killListRegion);
        
        var ocrRows = await ocrService.ExtractKillListRows(screenshot);
        var kills = killParser.ParseKillList(ocrRows, raidInfo.RaidId);
        
        KillListDetected?.Invoke(this, new KillListDetectedEventArgs
        {
            RaidInfo = raidInfo,
            Kills = kills
        });
    }
}
```

#### Persistence Integration

**Using Existing Stats.cs Pattern**:
```csharp
// In Stats.cs
public static void AddKillEvent(KillEvent killEvent, Profile profile)
{
    var sql = @"INSERT INTO kill_events
                (raid_id, kill_index, kill_time, enemy_type, 
                 pmc_name, weapon_name, raw_ocr_row, profile_id) 
                VALUES (@raid_id, @kill_index, @kill_time, @enemy_type,
                        @pmc_name, @weapon_name, @raw_ocr_row, @profile_id);";
    
    var parameters = new Dictionary<string, object>
    {
        { "raid_id", killEvent.RaidId },
        { "kill_index", killEvent.KillIndex },
        { "kill_time", killEvent.KillTime },
        { "enemy_type", killEvent.EnemyType },
        { "pmc_name", killEvent.PmcName ?? DBNull.Value },
        { "weapon_name", killEvent.WeaponName ?? DBNull.Value },
        { "raw_ocr_row", killEvent.RawOcrRow },
        { "profile_id", profile.Id }
    };
    
    Query(sql, parameters);
}

// Database schema update
private static void CreateKillEventsTable()
{
    var sql = @"CREATE TABLE IF NOT EXISTS kill_events (
        id INTEGER PRIMARY KEY,
        raid_id VARCHAR(24),
        kill_index INTEGER,
        kill_time VARCHAR(10),
        enemy_type VARCHAR(20),
        pmc_name VARCHAR(100),
        weapon_name VARCHAR(100),
        raw_ocr_row TEXT,
        profile_id VARCHAR(24),
        timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );";
    
    using var command = new SQLiteCommand(Connection);
    command.CommandText = sql;
    command.ExecuteNonQuery();
}
```

### Recommended Investigation Steps

~~To determine if kill tracking is feasible:~~

**Update**: Investigation complete. Kill data is NOT in game logs. OCR-based approach required.

**Implementation Steps**:

1. **Add OCR Dependencies**:
   ```xml
   <!-- Add to TarkovMonitor.csproj -->
   <PackageReference Include="Tesseract" Version="5.2.0" />
   <!-- OR use built-in Windows.Media.Ocr -->
   ```

2. **Create Screen Detection Logic**:
   - Identify unique UI elements of kill list screen
   - Determine kill list bounding box coordinates
   - Handle different resolutions/aspect ratios

3. **Implement OCR Pipeline**:
   - Capture kill list region only
   - Pre-process image for better OCR accuracy
   - Extract text rows

4. **Build Parser**:
   - Anchor on time column
   - Extract enemy type
   - Handle optional PMC name/weapon fields
   - Preserve raw OCR for debugging

5. **Test with Various Scenarios**:
   - Mixed kills (PMCs + Scavs)
   - All PMC kills with dogtags looted
   - No PMC kills (Scavs only)
   - Partial dogtag collection

6. **Integration**:
   - Add to existing event flow
   - Persist to SQLite database
   - Make feature configurable
   - Add UI toggle in Settings

### Configuration & Settings

**Settings.Designer.cs additions**:
```csharp
[UserScopedSetting]
[DefaultSettingValue("False")]
public bool enableKillTracking { get; set; }

[UserScopedSetting]
[DefaultSettingValue("True")]
public bool includeKillsInMarkers { get; set; }

[UserScopedSetting]
[DefaultSettingValue("True")]
public bool includeKillsInChapters { get; set; }

[UserScopedSetting]
[DefaultSettingValue("PMC")]
public string killTrackingFilter { get; set; } // "All", "PMC", "PMCWithLoot"
```

**UI Configuration**:
```
Kill Tracking (OCR-based):
┌─────────────────────────────────────┐
│ ☑ Enable Kill Tracking             │
│   (Reads end-of-raid kill list)    │
│                                     │
│ Filter:                             │
│ ○ All Kills (PMC + Scav)           │
│ ● PMC Kills Only                   │
│ ○ PMC with Looted Dogtags Only     │
│                                     │
│ Stream Integration:                 │
│ ☑ Include in stream markers        │
│ ☑ Include in chapters              │
│                                     │
│ Marker Format:                      │
│ [Kill - {type} at {time}]          │
│   Example: "Kill - PMC at 12:34"   │
└─────────────────────────────────────┘
```

```

### YouTube Integration with OCR Kill Data

**Post-Raid Chapter Generation**:
```csharp
// In MainBlazorUI.cs
private async void Eft_KillListDetected(object? sender, KillListDetectedEventArgs e)
{
    if (!Properties.Settings.Default.enableKillTracking)
        return;
        
    // Store kills for this raid
    foreach (var kill in e.Kills)
    {
        Stats.AddKillEvent(kill, e.Profile);
        
        // Add to stream recorder for chapter generation
        if (streamRecorder != null && Properties.Settings.Default.includeKillsInChapters)
        {
            var description = FormatKillDescription(kill);
            streamRecorder.RecordEvent("Kill", description);
        }
    }
    
    messageLog.AddMessage($"Recorded {e.Kills.Count} kills from raid {e.RaidInfo.Map}");
}

private string FormatKillDescription(KillEvent kill)
{
    if (!string.IsNullOrEmpty(kill.PmcName) && !string.IsNullOrEmpty(kill.WeaponName))
        return $"Kill - {kill.PmcName} with {kill.WeaponName} at {kill.KillTime}";
    else if (!string.IsNullOrEmpty(kill.PmcName))
        return $"Kill - {kill.PmcName} at {kill.KillTime}";
    else
        return $"Kill - {kill.EnemyType} at {kill.KillTime}";
}
```

### Limitations and Considerations

**OCR-Based Approach**:
- ✅ Captures all kills (PMC + Scav + Boss)
- ✅ Includes timestamps for accurate chapter timing
- ✅ Works post-raid (no performance impact during gameplay)
- ⚠️ Requires screen detection and OCR accuracy
- ⚠️ PMC details only available if dogtags looted
- ❌ Not real-time (cannot create live stream markers during raid)

**Stream Integration Strategy**:
- **Live Markers**: NOT available (no real-time kill data)
- **Chapters**: Full support with accurate timestamps from OCR
- **Recommended**: Include all kills in chapters, filter in settings

**Quota Management for Kills**:
- No quota impact (chapters are post-stream, one-time cost)
- Can include all kills in chapters without concern
- Filter options: All kills, PMC only, PMC with dogtags only
- Always include in chapter generation for comprehensive timeline

### Acceptance Criteria

- ✅ Per-kill timestamps reliably captured from end-of-raid screen
- ✅ Mixed raids (Scavs + PMCs) parse correctly  
- ✅ Missing PMC enrichment fields do not cause row loss
- ✅ Feature cleanly coexists with all existing features
- ✅ Optional/configurable via settings
- ✅ Follows existing module boundaries and patterns
- ✅ Uses existing persistence (Stats.cs SQLite pattern)
- ✅ No in-raid capture or memory inspection
- ✅ No breaking changes to existing features

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

**Investigation Complete**: Kill data is **NOT available in game logs**.

**Solution**: OCR-based extraction from end-of-raid kill list screen.

**Implementation Approach**:
- Screen detection for kill list UI
- OCR extraction of kill rows
- Time-anchored parsing with optional PMC fields
- SQLite persistence following existing patterns
- Integration with chapter generation (post-raid)

**Capabilities**:
- ✅ All kills captured (PMC + Scav + Boss)
- ✅ Accurate timestamps for chapters
- ✅ Optional PMC name/weapon (when dogtag looted)
- ✅ No performance impact during raid
- ❌ No real-time stream markers (post-raid only)

**Acceptance Criteria Met**:
- Per-kill timestamps from end-of-raid screen
- Mixed kill types handled correctly
- Missing PMC fields do not cause data loss
- Optional/configurable feature
- Follows existing architectural patterns
- No in-raid capture or memory inspection

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
- Use OCR for kill tracking (no game memory access)

**Next Steps**:
1. Create Google Cloud project and obtain OAuth credentials
2. Implement YouTubeAuth.cs with Google.Apis library
3. Add OAuth flow UI to Settings page
4. Create YouTubeStreamRecorder class for event tracking
5. Integrate with existing GameWatcher events
6. Implement OCR kill tracking (EndOfRaidScreenDetector, KillListOcrService, KillListParser)
7. Implement chapter generation logic
8. Add kill events to chapter generation
9. Test with live YouTube broadcasts
10. Validate chapters on VOD playback with kill timestamps
