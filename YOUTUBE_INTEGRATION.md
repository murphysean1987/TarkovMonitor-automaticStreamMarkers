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

### Phase 4: Error Handling & Polish
- [ ] Handle rate limiting (quota: 10,000 units/day)
- [ ] Add retry logic with exponential backoff
- [ ] Implement token refresh error handling
- [ ] Add user-facing error messages
- [ ] Add logging for debugging
- [ ] Test multi-profile scenarios

### Phase 5: Testing & Documentation
- [ ] Test OAuth flow end-to-end
- [ ] Test token refresh after expiration
- [ ] Test with actual YouTube live stream
- [ ] Verify markers appear in YouTube Studio
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

**Mitigation**:
- Cache active broadcast ID (refresh every 5 minutes)
- Only check for active broadcast on first marker attempt
- Implement local rate limiting (max 1 marker per minute per event type)
- Add user notification when approaching quota limit

## Alternative Approaches

### 1. YouTube Data API v3 (Chapters)

Instead of cue points during live stream, create chapters after stream:
- **Pros**: Better user experience, visible in YouTube player
- **Cons**: Post-stream only, requires VOD processing
- **Use case**: Complementary to live markers

### 2. YouTube Studio API (Limited Access)

Direct integration with YouTube Studio features:
- **Pros**: More native marker support
- **Cons**: Requires special API access, not generally available

### 3. Third-Party Services

Services like Restream or StreamElements:
- **Pros**: Multi-platform support
- **Cons**: Additional dependencies, subscription costs

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

**Next Steps**:
1. Create Google Cloud project and obtain OAuth credentials
2. Implement YouTubeAuth.cs with Google.Apis library
3. Add OAuth flow UI to Settings page
4. Integrate with existing GameWatcher events
5. Test with live YouTube broadcasts
