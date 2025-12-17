# Video Feature Documentation - SCCS Backend

## Overview

The SCCS (Security Command and Control System) backend provides comprehensive video capabilities for both **live viewing** and **playback** of security camera footage. The system integrates with two primary video sources:
1. **Milestone Mobile Service** - Returns video as image frames
2. **Dahua DSS Pro** - Returns RTSP URLs that are converted to image frames

The video streaming architecture uses **WebSocket** connections to efficiently transmit image frames from backend services to the frontend, enabling real-time video display in the web browser.

---

## System Architecture

### Components
- **Frontend (React 17)**: Web application displaying video streams
- **LegacyAPI Service**: API gateway (communication layer between frontend and video services) - *Currently not actively used for video, frontend directly connects to video services*
- **MilestoneMediaServices**: Microservice handling Milestone video playback
- **DahuaMediaServices**: Microservice handling Dahua video playback and live view
- **Video Sources**: 
  - Milestone Mobile Server (image frames)
  - Dahua DSS Pro (RTSP streams)

### Communication Flow

```mermaid
graph LR
    A[React Frontend] -->|HTTP API| B[Video API Endpoint Discovery]
    A -->|WebSocket| C[MilestoneMediaServices]
    A -->|WebSocket| D[DahuaMediaServices]
    C -->|Mobile SDK| E[Milestone Server]
    D -->|HTTP API| F[Dahua DSS Pro]
    F -->|RTSP| D
    D -->|FFmpeg| D
    E -->|Image Frames| C
    C -->|WebSocket Image Frames| A
    D -->|WebSocket Image Frames| A
```

---

## Step-by-Step Processing Flow

### Step 1: Frontend Initialization and Camera Configuration Discovery

**Description**: When a user accesses a video page (playback or live view), the frontend component loads camera configuration from the core API and determines which video controller to use.

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/pages/video/playback.js`]
- [`core-frontend/MozartCoreFrontend/ClientApp/src/pages/video/liveview.js`]
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/camera.js`]

**Code Snippet** (Playback Page Initialization):

```javascript
// From playback.js
const getCameraGroups = async () => {
    let disco = await Discovery.get();
    let url = `${disco['mozart-video-api_endpoint']}/cameragroups`;
    return mozartApi.get(url, true)
};

const getLocations = async () => {
    let disco = await Discovery.get();
    let url = `${disco['mozart-core-api_endpoint']}/locations`;
    return mozartApi.get(url, true);
};

// Camera component loads configuration
const getCameraConfig = async (assetId) => {
    let disco = await Discovery.get();
    let url = `${disco['mozart-video-api_endpoint']}/cameraconfigs/asset/${assetId}`;
    return mozartApi.get(url, true);
}

const getPlaybackControllers = async () => {
    let disco = await Discovery.get();
    let url = `${disco['mozart-video-api_endpoint']}/playbackcontrollers`;
    return mozartApi.get(url, true);
}
```

**Explanation**: 
- The Discovery service provides endpoints for different microservices (core-api, video-api)
- Camera configuration includes controller IDs that determine whether to use Milestone or Dahua
- `PlaybackController` or `LiveViewController` configuration determines which video service to connect to

---

### Step 2: WebSocket Connection Setup

**Description**: Before requesting video playback, the frontend obtains a WebSocket URL and session ID from the video service. This establishes a dedicated channel for streaming video frames.

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]

**Code Snippet** (Frontend WebSocket Connection):

```javascript
// From playbackcameraframes_byimageframes.js
const getPlaybackUrl = async (cameraConfigId, SessionId) => {
    if (cameraConfigId == null) {
        return;
    }

    let disco = await Discovery.get(); 
    let url = `${disco['mozart-video-api_endpoint']}/playbackurl?cameraConfigId=${cameraConfigId}&sessionId=${SessionId}`;
    
    return mozartApi.get(url, false); 
}

const connectWebSocket = async () => {
    try {
        const result = await getPlaybackUrl(assetId, playBackSessionId.current);
        const jsonObject = JSON.parse(result);
        const url = jsonObject.url;  // WebSocket URL
        const sessionId = jsonObject.sessionId;  // Unique session ID
        playBackSessionId.current = sessionId;
        const ws = new WebSocket(url);

        ws.onopen = () => {
            setSocketStatus('connected');
            let time = moment(startDate).add(startSecond, 'seconds').format('YYYY-MM-DD HH:mm');

            if (assetId) {
                sendPlaybackRequest(time, sessionId, assetId);
            }
        }

        ws.onmessage = (event) => {
            const messageData = JSON.parse(event.data);
            
            if (!messageData.hasOwnProperty('image') || !messageData.hasOwnProperty('time')) {
                return;
            }

            const imageUrl = "data:image/png;base64," + messageData.image;
            const date = new Date(messageData.time);
            const totalSeconds = (date.getHours() * 3600) + (date.getMinutes() * 60) + date.getSeconds();
            currentPlayTime.current = totalSeconds; 

            setTimeRangeValue([totalSeconds, 86399]);
            setMessage(imageUrl);  // Display the frame
        }

        ws.onclose = () => {
            handleStopPlayback(); 
            setSocketStatus('disconnected');
            console.log('WebSocket disconnected');
        }

        ws.onerror = (error) => {
            handleStopPlayback(); 
            setSocketStatus('error');
            console.error('WebSocket error:', error);
            setError(error);
        }

        socketRef.current = ws;
    } catch (error) {
        console.error('Error connecting to WebSocket:', error);
    }
}
```

**Corresponding Source Files** (Backend):
- [`video/MilestoneMediaServices/Controllers/MediaController.cs`]
- [`video/DahuaMediaServices/DahuaMediaServices/Controllers/MediaController.cs`]

**Code Snippet** (Backend - Milestone):

```csharp
// From MilestoneMediaServices MediaController.cs
[HttpGet("playbackurl")]
public Task<ActionResult<dynamic>> GetPlaybackurl(string? sessionId)
{
    sessionId = Guid.NewGuid().ToString();

    // Generate the URL for the WebSocket endpoint 
    string endpointUrl = $"{_configuration["WebSocketServerUrl"]}/{sessionId}";

    // Create a dynamic object to hold the endpoint URL and session ID
    var result = new
    {
        Url = endpointUrl,
        SessionId = sessionId,
    };

    // Return the WebSocket endpoint URL and session ID to the client as a JSON object
    return Task.FromResult<ActionResult<dynamic>>(Ok(result));
}

[HttpGet("ws/{sessionId}")]
public async Task<IActionResult> HandleWebSocket(string sessionId)
{
    if (HttpContext.WebSockets.IsWebSocketRequest)
    {
        using var webSocket = await HttpContext.WebSockets.AcceptWebSocketAsync();
        await _webSocketServer.AddConnectionAsync(sessionId, webSocket);
        return Ok();
    }
    else
    {
        return BadRequest("This endpoint only accepts WebSocket requests.");
    }
}
```

**Explanation**: 
- The backend generates a unique session ID for each video stream request
- Returns a WebSocket URL that the frontend connects to
- When frontend connects, the WebSocket is stored in a concurrent dictionary keyed by session ID
- This dedicated WebSocket channel is used to stream video frames

---

### Step 3: Start Playback Request

**Description**: After WebSocket connection is established, the frontend sends an HTTP POST request to start video playback with specific parameters (camera, time range, resolution).

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]

**Code Snippet** (Frontend):

```javascript
// From playbackcameraframes_byimageframes.js
const sendPlaybackRequest = async (time, sessionId, id,) => { 
    let disco = await Discovery.get();
    let url = `${disco['mozart-video-api_endpoint']}/startplayback`;

    let obj = {
        SessionId: sessionId,
        CameraConfigId: id,  // Camera reference
        StartTime: time,  
        EndTime: time,  
        Width: 1920,  
        Height: 1080,  
        Speed: playSpeed  
    };

    return mozartApi.post(url, obj);
};

const handleRefreshPlayback = () => {
    setIsPlaying(true);
    setIsStopped(false);
    
    let time = moment(startDate).add(startSecond, 'seconds').format('YYYY-MM-DD HH:mm');

    if (socketStatus == "disconnected") { 
        if (assetId == null) {
            return;
        }
        connectWebSocket();
    } else if (assetId) {
        sendPlaybackRequest(time, playBackSessionId.current, assetId);
    }
}
```

**Corresponding Source Files** (Backend):
- [`video/MilestoneMediaServices/Controllers/MediaController.cs`]
- [`video/DahuaMediaServices/DahuaMediaServices/Controllers/MediaController.cs`]

**Code Snippet** (Backend - Milestone):

```csharp
// From MilestoneMediaServices MediaController.cs
[HttpPost("startplayback")]
public Task<IActionResult> Startplayback(PlaybackInput playbackInput)
{
    if (string.IsNullOrEmpty(playbackInput.SessionId)) return Task.FromResult<IActionResult>(Ok());

    _logger.LogInformation($"Startplayback - sessionid={playbackInput.SessionId}");

    _milestoneHelper.StartVideo(playbackInput);

    return Task.FromResult<IActionResult>(Ok());
}

public class PlaybackInput
{
    public string SessionId { get; set; }
    public int CameraConfigId { get; set; }
    public string ExternalCameraRef { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public int Width { get; set; } = 800;
    public int Height { get; set; } = 600;
    public int Speed { get; set; } = 1;
}
```

**Code Snippet** (Backend - Dahua):

```csharp
// From DahuaMediaServices MediaController.cs
[HttpPost("startplayback")]
public async Task<IActionResult> Startplayback(PlaybackInput playbackInput)
{
    _logger.LogWarning($"Startplayback sessionid={playbackInput.SessionId},ExternalCameraRef={playbackInput.ExternalCameraRef},StartTime={playbackInput.StartTime}, EndTime={playbackInput.EndTime} ");

    // Validate SessionId
    if (string.IsNullOrEmpty(playbackInput.SessionId))
    {
        return BadRequest(new { error = "SessionId is required" });
    }

    // Validate ExternalCameraRef
    if (string.IsNullOrWhiteSpace(playbackInput.ExternalCameraRef))
    {
        _logger.LogWarning($"Startplayback failed - ExternalCameraRef={playbackInput.ExternalCameraRef}, ExternalCameraRef is missing");
        return BadRequest(new { error = "ExternalCameraRef is required" });
    }

    // Validate time parameters...
    _logger.LogInformation($"Startplayback - sessionid={playbackInput.SessionId}");

    bool resultok = await _dahuaHelper.StartVideo(playbackInput);

    return Ok();
}
```

**Explanation**: 
- Frontend sends playback parameters including session ID, camera ID, time range, and resolution
- Backend validates the request and triggers the video streaming process
- The session ID links this playback request to the previously established WebSocket connection

---

### Step 4A: Milestone Video Processing (Image Frames)

**Description**: For Milestone cameras, the service uses the Milestone Mobile SDK to connect to the Milestone server and retrieve video frames directly as JPEG images.

**Corresponding Source Files**:
- [`video/MilestoneMediaServices/MilestoneHelper.cs`]

**Code Snippet**:

```csharp
// From MilestoneHelper.cs
public class MilestoneHelper
{
    private readonly ConcurrentDictionary<string, MilestoneConnection> MilestoneConnectionList = new();
    private readonly WebSocketServer _webSocketServer;
    private readonly MilestoneConfiguration _milestoneConfiguration;
    private readonly ILogger<MilestoneHelper> _logger;

    public MilestoneHelper(IOptions<MilestoneConfiguration> milestoneConfigOptions, 
                          WebSocketServer webSocketServer, 
                          ILogger<MilestoneHelper> logger)
    {
        _webSocketServer = webSocketServer;
        _milestoneConfiguration = milestoneConfigOptions.Value;
        _logger = logger;
    }

    public void StartVideo(PlaybackInput playbackInput)
    {
        MilestoneConnection milestoneConnection;
        if (MilestoneConnectionList.TryGetValue(playbackInput.SessionId, out milestoneConnection))
            milestoneConnection.ReStartVideo(playbackInput); // Existing playback, user selected new playback attribute
        else
        {
            milestoneConnection = new MilestoneConnection(playbackInput, _milestoneConfiguration, _webSocketServer, _logger);
            milestoneConnection.ReStartVideo();
        }
    }

    public void CloseVideo(string sessionId)
    {
        MilestoneConnection milestoneConnection;
        if (MilestoneConnectionList.TryGetValue(sessionId, out milestoneConnection))
            milestoneConnection.CloseVideo();
    }
}
```

**Explanation**: 
- MilestoneHelper manages multiple concurrent video connections using a concurrent dictionary
- Each connection is represented by a `MilestoneConnection` object
- The MilestoneConnection uses the Milestone Mobile SDK APIs to:
  - Connect to the Milestone Mobile Server
  - Request video frames for specific camera and time range
  - Receive JPEG image frames directly from Milestone
  - Forward these frames to the WebSocket for transmission to frontend

**Data Flow**:
```
Milestone Server → Milestone Mobile SDK → MilestoneConnection → WebSocketServer → Frontend
     (JPEG frames)
```

---

### Step 4B: Dahua Video Processing (RTSP to Image Frames)

**Description**: For Dahua cameras, the service first obtains an RTSP URL from the Dahua DSS Pro server, then uses FFmpeg to convert the RTSP stream into JPEG image frames.

**Corresponding Source Files**:
- [`video/DahuaMediaServices/DahuaMediaServices/Helpers/DahuaHelper.cs`]
- [`video/DahuaMediaServices/DahuaMediaServices/Helpers/RtspToFrameConverter.cs`]

**Code Snippet** (DahuaHelper):

```csharp
// From DahuaHelper.cs
public class DahuaHelper
{
    private readonly DahuaConfig _dahuaConfig;
    private readonly DahuaApiHelper _dahuaApiHelper;
    private readonly DahuaSessionManager _dahuaSessionManager;
    private readonly WebSocketServer _webSocketServer;
    private readonly ILogger<DahuaHelper> _logger;

    // Store active RTSP converters by session
    private readonly Dictionary<string, RtspToFrameConverter> _activeConverters = new();
    private readonly Dictionary<string, DateTime> _lastSentFrameDateTime = new();
    private readonly Dictionary<string, DateTime> _initDateTime = new();
    private readonly object _lockObject = new object();

    public async Task<bool> StartVideo(PlaybackInput playbackInput)
    {
        try
        {
            // Stop any existing video for this session
            await StopVideo(playbackInput.SessionId);

            // Record init time for this session
            lock (_lockObject)
            {
                _initDateTime[playbackInput.SessionId] = DateTime.Now;
                _lastSentFrameDateTime[playbackInput.SessionId] = DateTime.MinValue;
            }

            WebSocket websocket = _webSocketServer.GetWebSocketBySessionId(playbackInput.SessionId);
            if (websocket == null || websocket.State != WebSocketState.Open)
            {
                _logger.LogWarning($"WebSocket not found or not open for session {playbackInput.SessionId}");
                return false;
            }

            // Get RTSP URL from Dahua API
            string rtsp = await _dahuaApiHelper.GetPlaybackRtspNoLogin(
                playbackInput.ExternalCameraRef,
                playbackInput.StartTime,
                playbackInput.EndTime);

            if (string.IsNullOrWhiteSpace(rtsp))
            {
                _logger.LogWarning($"Failed to get RTSP URL for session {playbackInput.SessionId}");
                return false;
            }

            _logger.LogInformation($"Starting RTSP stream for session {playbackInput.SessionId}, URL: {rtsp}");

            // Create new RTSP converter
            var converter = new RtspToFrameConverter(_logger);

            // Subscribe to frame events
            converter.NewFrame += async (frame, sessionId) =>
                await OnNewFrame(frame, sessionId);

            converter.StreamEnded += (sessionId, reason) =>
                OnStreamEnded(sessionId, reason);

            // Store converter
            lock (_lockObject)
            {
                _activeConverters[playbackInput.SessionId] = converter;
            }

            // Start streaming (15 FPS default)
            await converter.StartRtspStream(rtsp, playbackInput.SessionId, targetFps: 15);

            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Error starting video for session {playbackInput.SessionId}");
            return false;
        }
    }

    private async Task OnNewFrame(VideoFrame frame, string sessionId)
    {
        if (frame?.Data.Length <= 0) return;

        _logger.LogInformation($"OnNewFrame - SessionId={sessionId}, FrameTime={frame.MainHeader.TimeStampUtcMs}");

        // Get WebSocket for this session
        WebSocket webSocket = _webSocketServer.GetWebSocketBySessionId(sessionId);

        if (webSocket == null || webSocket.State != WebSocketState.Open)
        {
            HandleClientNotConnected(sessionId);
            return;
        }

        try
        {
            // Convert timestamp to local time
            var frameTime = ConvertTimestampToDateTime(frame.MainHeader.TimeStampUtcMs).ToLocalTime();

            // Serialize frame data efficiently
            var messageBytes = SerializeFrameData(frameTime, frame.Data);

            _logger.LogInformation($"Sending frame to session {sessionId}, FrameTime={frameTime}");

            await webSocket.SendAsync(new ArraySegment<byte>(messageBytes),
                WebSocketMessageType.Text, true, CancellationToken.None);
        }   
        catch (WebSocketException ex)
        {
            _logger.LogWarning($"WebSocket error for session {sessionId}: {ex.Message}");
            // Handle timeout logic...
        }
    }
}
```

**Code Snippet** (RtspToFrameConverter - FFmpeg Integration):

```csharp
// From RtspToFrameConverter.cs
public class RtspToFrameConverter : IDisposable
{
    private Process _ffmpegProcess;
    private readonly ILogger _logger;
    private CancellationTokenSource _cancellationTokenSource;
    private Task _readingTask;
    private string _currentSessionId;

    // Events
    public event Func<VideoFrame, string, Task> NewFrame;
    public event Action<string, string> StreamEnded;

    public async Task StartRtspStream(string rtspUrl, string sessionId, int targetFps = 15)
    {
        if (string.IsNullOrWhiteSpace(rtspUrl))
            throw new ArgumentException("RTSP URL cannot be empty", nameof(rtspUrl));

        _currentSessionId = sessionId;
        _cancellationTokenSource = new CancellationTokenSource();

        var ffmpegPath = GetFFmpegPath();
        if (string.IsNullOrEmpty(ffmpegPath))
        {
            throw new Exception("FFmpeg not found. Please install FFmpeg and ensure it's in the system PATH.");
        }

        var startInfo = new ProcessStartInfo
        {
            FileName = ffmpegPath,
            Arguments = BuildFFmpegArguments(rtspUrl, targetFps),
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true
        };

        _ffmpegProcess = new Process { StartInfo = startInfo };

        try
        {
            _ffmpegProcess.Start();
            _ffmpegProcess.BeginErrorReadLine();

            _logger.LogInformation($"Started FFmpeg process for session {sessionId}, PID: {_ffmpegProcess.Id}");

            // Start reading frames
            _readingTask = Task.Run(() => ReadFramesAsync(_cancellationTokenSource.Token));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Failed to start FFmpeg for session {sessionId}");
            throw;
        }
    }

    private string BuildFFmpegArguments(string rtspUrl, int targetFps)
    {
        // Convert RTSP to JPEG frames
        return $"-rtsp_transport tcp " +          // TCP for reliability
               $"-i \"{rtspUrl}\" " +              // Input RTSP URL
               $"-vf fps={targetFps} " +           // Target frame rate (15 FPS)
               $"-c:v mjpeg " +                    // JPEG codec
               $"-q:v 5 " +                        // Quality (2-31, lower = better)
               $"-f image2pipe " +                 // Pipe output
               $"-vcodec mjpeg " +
               $"-";                              // Output to stdout
    }

    private async Task ReadFramesAsync(CancellationToken cancellationToken)
    {
        try
        {
            var stream = _ffmpegProcess.StandardOutput.BaseStream;
            _logger.LogInformation($"Started reading frames for session {_currentSessionId}");

            while (!cancellationToken.IsCancellationRequested &&
                   !_ffmpegProcess.HasExited)
            {
                var frameData = await ReadNextJpegFrameAsync(stream, cancellationToken);

                if (frameData != null && frameData.Length > 0)
                {
                    _frameCount++;
                    _lastFrameTime = DateTime.UtcNow;

                    var videoFrame = new VideoFrame
                    {
                        Data = frameData,
                        MainHeader = new FrameHeader
                        {
                            TimeStampUtcMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
                        }
                    };

                    // Raise NewFrame event
                    await NewFrame?.Invoke(videoFrame, _currentSessionId);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Error reading frames for session {_currentSessionId}");
        }
    }
}
```

**Explanation**: 
- **Step 1**: DahuaApiHelper calls Dahua DSS Pro API to get an RTSP URL for the camera and time range
- **Step 2**: RtspToFrameConverter spawns an FFmpeg process with specific arguments:
  - Uses TCP transport for reliability
  - Converts video to MJPEG format at 15 FPS
  - Outputs JPEG frames to stdout (pipe)
- **Step 3**: ReadFramesAsync continuously reads JPEG frames from FFmpeg's stdout
- **Step 4**: Each frame is wrapped in a VideoFrame object and sent via NewFrame event
- **Step 5**: DahuaHelper's OnNewFrame handler receives the frame and sends it through WebSocket

**Data Flow**:
```
Dahua DSS Pro → RTSP URL → FFmpeg Process → JPEG Frames → RtspToFrameConverter → DahuaHelper → WebSocketServer → Frontend
   (RTSP stream)                              (stdout pipe)
```

---

### Step 5: WebSocket Frame Transmission

**Description**: Both Milestone and Dahua services use the same WebSocketServer to transmit video frames to the frontend. Frames are serialized as JSON with base64-encoded image data.

**Corresponding Source Files**:
- [`video/MilestoneMediaServices/WebSocketServer.cs`]
- [`video/DahuaMediaServices/DahuaMediaServices/WebSocketServer.cs`]

**Code Snippet**:

```csharp
// From WebSocketServer.cs (both services use similar implementation)
public class WebSocketServer
{
    private static readonly ConcurrentDictionary<string, WebSocket> _clientSockets = new ConcurrentDictionary<string, WebSocket>();
    private readonly TimeSpan _inactivityTimeout = TimeSpan.FromSeconds(30);
    private readonly System.Timers.Timer _pingTimer;
    ILogger<WebSocketServer> _logger;

    public WebSocketServer(ILogger<WebSocketServer> logger)
    {
        // Initialize a timer that triggers every 60 seconds to ping clients
        _pingTimer = new System.Timers.Timer(60000);
        _pingTimer.Elapsed += OnPingTimerElapsed;
        _pingTimer.AutoReset = true;
        _pingTimer.Start();

        _logger = logger;
    }

    // Ping all connected clients to keep connection alive
    private async void OnPingTimerElapsed(object sender, ElapsedEventArgs e)
    {
        foreach (var (sessionID, webSocket) in _clientSockets.ToList())
        {
            if (webSocket.State == WebSocketState.Open)
            {
                try
                {
                    await webSocket.SendAsync(new ArraySegment<byte>(messageBytes), 
                        WebSocketMessageType.Text, true, CancellationToken.None);
                    _logger.LogInformation($"Sent ping to client {sessionID}");
                }
                catch (Exception ex)
                {
                    _logger.LogError($"Error sending ping to client {sessionID}: {ex.Message}");
                    await CloseAndRemoveWebSocket(sessionID, webSocket);
                }
            }
            else
            {
                _logger.LogInformation($"WebSocket for client {sessionID} is not open. Closing connection.");
                await CloseAndRemoveWebSocket(sessionID, webSocket);
            }
        }
    }

    public async Task AddConnectionAsync(string sessionID, WebSocket webSocket)
    {
        _clientSockets.TryAdd(sessionID, webSocket);

        try
        {
            _logger.LogInformation($"New websocket connection - AddConnectionAsync {sessionID}");
            await HandleConnectionAsync(sessionID, webSocket);
        }
        finally
        {
            // Remove the connection when it's closed
            _clientSockets.TryRemove(sessionID, out _);
        }
    }

    private async Task HandleConnectionAsync(string sessionID, WebSocket webSocket)
    {
        var buffer = new byte[1024 * 4];
        while (webSocket.State == WebSocketState.Open)
        {
            var result = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);

            if (result.MessageType == WebSocketMessageType.Text)
            {
                var message = Encoding.UTF8.GetString(buffer, 0, result.Count);
                _logger.LogInformation($"Received message from {sessionID}: {message}");
            }
            else if (result.MessageType == WebSocketMessageType.Close)
            {
                _logger.LogInformation($"Closing websocket connection - WebSocketCloseStatus.NormalClosure - {sessionID}");
                await webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", CancellationToken.None);
            }
        }
    }

    public async Task CloseConnectionAsync(string sessionID)
    {
        if (_clientSockets.TryGetValue(sessionID, out var webSocket))
        {
            try
            {
                if (webSocket.State != WebSocketState.Closed && webSocket.State != WebSocketState.Aborted)
                {
                    await webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Session closed by server", CancellationToken.None);
                    _logger.LogInformation($"Closed WebSocket for client {sessionID}");
                }
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error closing WebSocket for client {sessionID}: {ex.Message}");
            }
            finally
            {
                _clientSockets.TryRemove(sessionID, out _);
            }
        }
    }

    // Get WebSocket by sessionID
    public WebSocket GetWebSocketBySessionId(string sessionID)
    {
        if (_clientSockets.TryGetValue(sessionID, out var webSocket))
        {
            return webSocket;
        }
        return null;
    }
}
```

**Explanation**: 
- WebSocketServer maintains a concurrent dictionary of active WebSocket connections indexed by session ID
- When video frames are ready, the video helper (Milestone or Dahua) retrieves the WebSocket by session ID
- Frames are serialized to JSON format: `{"image": "<base64-encoded-jpeg>", "time": "<timestamp>"}`
- The WebSocket sends the serialized frame data to the frontend
- A ping timer keeps connections alive by sending periodic ping messages
- Connection cleanup happens when clients disconnect or errors occur

**Data Encryption**: WebSocket communication typically uses WSS (WebSocket Secure) protocol in production for encrypted transmission.

---

### Step 6: Frontend Frame Rendering

**Description**: The frontend receives WebSocket messages containing base64-encoded JPEG images and displays them in the video player component.

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]

**Code Snippet**:

```javascript
// From playbackcameraframes_byimageframes.js (WebSocket message handler)
ws.onmessage = (event) => {
    const messageData = JSON.parse(event.data);
    
    // Check if message contains image frame data
    if (!messageData.hasOwnProperty('image') || !messageData.hasOwnProperty('time')) {
        return;
    }

    // Convert base64 to data URL for display
    const imageUrl = "data:image/png;base64," + messageData.image;
    const date = new Date(messageData.time);
    const totalSeconds = (date.getHours() * 3600) + (date.getMinutes() * 60) + date.getSeconds();
    currentPlayTime.current = totalSeconds; 

    // Update timeline slider
    setTimeRangeValue([totalSeconds, 86399]);
    
    // Display the frame
    setMessage(imageUrl);
}

// Render the video frame
return (
    <Box className={classes.container}>
        {isLoaded && !error && (
            <>
                {message && (
                    <img 
                        src={message} 
                        className={classes.videoElem} 
                        alt="Video Frame"
                    />
                )}
            </>
        )}
    </Box>
);
```

**Explanation**: 
- WebSocket `onmessage` handler receives JSON data containing:
  - `image`: Base64-encoded JPEG image data
  - `time`: Timestamp of the frame
- The base64 data is converted to a data URL format that can be displayed in an `<img>` tag
- The timestamp is used to update the playback timeline/slider
- React state update triggers re-render to display the new frame
- This creates the illusion of video playback through rapid sequential image display (~15 FPS)

---

### Step 7: Stop Playback

**Description**: When the user stops playback or closes the video player, the frontend sends a stop request and closes the WebSocket connection.

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]
- [`video/MilestoneMediaServices/Controllers/MediaController.cs`]
- [`video/DahuaMediaServices/DahuaMediaServices/Controllers/MediaController.cs`]

**Code Snippet** (Frontend):

```javascript
// From playbackcameraframes_byimageframes.js
const sendPlaybackStopRequest = async (cameraConfigId, sessionId) => {
    try {
        let disco = await Discovery.get();
        let url = `${disco['mozart-video-api_endpoint']}/stopplayback?cameraConfigId=${cameraConfigId}&sessionId=${sessionId}`;  

        return await mozartApi.post(url);
    } catch (error) {
        console.error("Error sending playback stop request:", error);
        setError(error);
    }
}

const handleStopPlayback = () => { 
    setIsPlaying(false);
    setIsStopped(true);

    if (assetId && playBackSessionId.current) {  
        sendPlaybackStopRequest(assetId, playBackSessionId.current);
        playBackSessionId.current = null;

        if (socketRef.current) {
            socketRef.current.close();
            setSocketStatus('disconnected');
        }
    }
}
```

**Code Snippet** (Backend - Milestone):

```csharp
// From MilestoneMediaServices MediaController.cs
[HttpPost("stopplayback/{sessionId}")]
public Task<IActionResult> Stopplayback(string sessionId)
{
    if (string.IsNullOrEmpty(sessionId)) return Task.FromResult<IActionResult>(Ok());

    _logger.LogInformation($"stopplayback - sessionid={sessionId}");

    _milestoneHelper.CloseVideo(sessionId);

    _webSocketServer.CloseConnectionAsync(sessionId);

    return Task.FromResult<IActionResult>(Ok());
}
```

**Code Snippet** (Backend - Dahua):

```csharp
// From DahuaMediaServices MediaController.cs
[HttpPost("stopplayback/{sessionId}")]
public async Task<IActionResult> Stopplayback(string sessionId)
{
    if (string.IsNullOrEmpty(sessionId))
    {
        return BadRequest(new { error = "SessionId is required" });
    }

    _logger.LogInformation($"stopplayback - sessionid={sessionId}");

    await _dahuaHelper.CloseVideo(sessionId);

    await _webSocketServer.CloseConnectionAsync(sessionId);

    return Ok();
}

// From DahuaHelper.cs
public async Task CloseVideo(string sessionId)
{
    await StopVideo(sessionId);
}

public async Task<bool> StopVideo(string sessionId)
{
    try
    {
        RtspToFrameConverter converter = null;

        lock (_lockObject)
        {
            if (_activeConverters.TryGetValue(sessionId, out converter))
            {
                _activeConverters.Remove(sessionId);
            }
            _lastSentFrameDateTime.Remove(sessionId);
            _initDateTime.Remove(sessionId);
        }

        if (converter != null)
        {
            _logger.LogInformation($"Stopping video for session {sessionId}");

            // Unsubscribe from events
            converter.NewFrame -= async (frame, sid) => await OnNewFrame(frame, sid);
            converter.StreamEnded -= (sid, reason) => OnStreamEnded(sid, reason);

            converter.Stop();  // Stops FFmpeg process for Dahua
            converter.Dispose();
            return true;
        }

        return false;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, $"Error stopping video for session {sessionId}");
        return false;
    }
}
```

**Explanation**: 
- Frontend sends HTTP POST to `/stopplayback/{sessionId}` endpoint
- Backend cleanup process:
  1. Milestone: Closes connection to Milestone Mobile Server
  2. Dahua: Stops FFmpeg process and releases resources
  3. Both: Close and remove the WebSocket connection from the dictionary
  4. Clean up all session-specific resources
- Frontend closes its WebSocket connection and updates UI state

---

## Video Playback Use Cases

### Use Case 1: Date Selection for Playback

**Description**: Users can select a specific date to view historical camera footage from that day.

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]

**Code Snippet**:

```javascript
// From playbackcameraframes_byimageframes.js
const [startDate, setStartDate] = useState(moment().format('YYYY-MM-DD'));
const [startSecond, setStartSecond] = useState(0);
const [timeRangeValue, setTimeRangeValue] = useState([0, 86399]); // 0 to 86399 seconds (24 hours)

const handleDateFromChange = event => { 
    setStartDate(event.target.value);
}

// When play is initiated
const handleRefreshPlayback = () => {
    setIsPlaying(true);
    setIsStopped(false);
    
    // Combine selected date with timeline position to get exact timestamp
    let time = moment(startDate).add(startSecond, 'seconds').format('YYYY-MM-DD HH:mm');

    if (socketStatus == "disconnected") { 
        if (assetId == null) {
            return;
        }
        connectWebSocket();
    } else if (assetId) {
        sendPlaybackRequest(time, playBackSessionId.current, assetId);
    }
}

// UI Component
<TextField 
    type="date"
    value={startDate}
    onChange={handleDateFromChange}
    inputProps={{
        max: moment().format('YYYY-MM-DD') // Prevent future dates
    }}
/>
```

**User Flow**:
1. User opens the playback page
2. User clicks on the date picker control
3. User selects a specific date from the calendar
4. The selected date is stored in `startDate` state
5. When playback starts, the system combines the date with the timeline position to request video from that specific date and time

**Technical Notes**:
- Date is stored in `YYYY-MM-DD` format using moment.js
- Maximum selectable date is today (prevents future dates)
- Date selection works in conjunction with the timeline slider to determine exact playback timestamp

---

### Use Case 2: Timeline Navigation

**Description**: Users can drag a timeline slider to navigate to specific times within the selected day's footage. The timeline represents a 24-hour period (86,399 seconds).

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]

**Code Snippet**:

```javascript
// From playbackcameraframes_byimageframes.js
const [timeRangeValue, setTimeRangeValue] = useState([0, 86399]); // Start and end in seconds
const [currentPos, setCurrentPos] = useState(0);
const currentPlayTime = useRef(0);

const time_convert = (value) => {
    const hours = Math.floor(value / 3600);
    const minutes = Math.floor((value % 3600) / 60);
    const seconds = value % 60;

    return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
}

const timeChangeHandler = (event, newValue) => { 
    setTimeRangeValue(newValue);
    setStartSecond(newValue[0]); 
}

const handleSliderChange = (event, newValue) => { 
    handleStopPlayback(); // Stop current playback before seeking
    
    const currentTime = new Date();
    const currentSecond = currentTime.getHours() * 3600 + currentTime.getMinutes() * 60 + currentTime.getSeconds();

    // Determine if the end time is being changed
    const endChanging = newValue[1] !== timeRangeValue[1];

    if (endChanging) {
        // Prevent changes to the end time
        setTimeRangeValue([timeRangeValue[0], timeRangeValue[1]]);
        return;
    }

    // Prevent start time from being set to a value beyond the current time if it's today's date
    if (startDate === moment().format('YYYY-MM-DD')) {
        if (newValue[0] >= currentSecond) {
            return;
        }
    }

    // Update state with the new values
    timeChangeHandler(event, newValue);

    // Calculate the position as a percentage based on the new values
    setCurrentPos(((newValue[0] + newValue[1]) / 2 / 86399) * 100);
}

// When frames are received, update timeline position
ws.onmessage = (event) => {
    const messageData = JSON.parse(event.data);
    
    if (!messageData.hasOwnProperty('image') || !messageData.hasOwnProperty('time')) {
        return;
    }

    const imageUrl = "data:image/png;base64," + messageData.image;
    const date = new Date(messageData.time);
    
    // Convert timestamp to seconds since midnight
    const totalSeconds = (date.getHours() * 3600) + (date.getMinutes() * 60) + date.getSeconds();
    currentPlayTime.current = totalSeconds; 

    // Update timeline slider to show current position
    setTimeRangeValue([totalSeconds, 86399]);
    
    setMessage(imageUrl);
}

// UI Component - Timeline Slider
<Slider
    value={timeRangeValue}
    onChange={handleSliderChange}
    min={0}
    max={86399}
    valueLabelDisplay="auto"
    valueLabelFormat={time_convert}
    marks={[
        { value: 0, label: '00:00' },
        { value: 21600, label: '06:00' },
        { value: 43200, label: '12:00' },
        { value: 64800, label: '18:00' },
        { value: 86399, label: '23:59' }
    ]}
/>
```

**User Flow**:
1. User sees a timeline slider representing 24 hours (00:00 to 23:59)
2. User drags the slider handle to a specific time position
3. Current playback stops automatically
4. System validates the selected time (cannot select future times for today's date)
5. User clicks play to start playback from the selected position
6. As video plays, the slider updates in real-time to show current playback position

**Technical Notes**:
- Timeline uses seconds since midnight (0 to 86,399 representing 00:00:00 to 23:59:59)
- `time_convert` function converts seconds to HH:MM:SS format for display
- Red line indicator shows current playback position
- For current day, users cannot seek beyond the current time
- Slider automatically updates as frames are received during playback
- End time handle is locked to prevent modification (playback always ends at 23:59:59)

---

### Use Case 3: Playback Speed Control

**Description**: Users can adjust the video playback speed from 1x (normal) to 4x (fast forward) to quickly review footage.

**Corresponding Source Files**:
- [`core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/playbackcameraframes_byimageframes.js`]

**Code Snippet**:

```javascript
// From playbackcameraframes_byimageframes.js
const [playSpeed, setPlaySpeed] = useState(1); // Default speed is 1x

const handleChange = (event) => {
    if (playSpeed != event.target.value) {
        setPlaySpeed(event.target.value);
    }
};

const sendPlaybackRequest = async (time, sessionId, id,) => { 
    let disco = await Discovery.get();
    let url = `${disco['mozart-video-api_endpoint']}/startplayback`;

    let obj = {
        SessionId: sessionId,
        CameraConfigId: id,
        StartTime: time,  
        EndTime: time,  
        Width: 1920,  
        Height: 1080,  
        Speed: playSpeed  // Include selected speed in request
    };

    return mozartApi.post(url, obj);
};

// UI Component - Speed Selection Dropdown
<Select
    value={playSpeed}
    onChange={handleChange}
    displayEmpty
    inputProps={{ 'aria-label': 'Playback Speed' }}
>
    <MenuItem value={1}>1x (Normal)</MenuItem>
    <MenuItem value={2}>2x (Fast)</MenuItem>
    <MenuItem value={3}>3x (Faster)</MenuItem>
    <MenuItem value={4}>4x (Fastest)</MenuItem>
</Select>
```

**Backend Processing** (Both Milestone and Dahua):

```csharp
// From MediaController.cs (both services)
public class PlaybackInput
{
    public string SessionId { get; set; }
    public int CameraConfigId { get; set; }
    public string ExternalCameraRef { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public int Width { get; set; } = 800;
    public int Height { get; set; } = 600;
    public int Speed { get; set; } = 1;  // Speed multiplier (1-4)
}
```

**User Flow**:
1. User selects desired playback date and time position
2. User clicks on the speed dropdown selector (default shows "1x")
3. User selects desired speed: 1x, 2x, 3x, or 4x
4. If video is already playing, the speed change takes effect on the next playback request
5. Backend adjusts frame serving rate according to the selected speed multiplier
6. Video playback accelerates - at 4x speed, users can review 4 minutes of footage in 1 minute

**Technical Notes**:
- Speed parameter is included in the `PlaybackInput` object sent to backend services
- Speed values: `1` (normal), `2` (double speed), `3` (triple speed), `4` (quadruple speed)
- Backend services adjust frame timing based on the speed multiplier
- For Milestone: SDK controls playback speed through Mobile Server API
- For Dahua: FFmpeg frame rate and timestamp calculations are adjusted
- Higher speeds may result in some frame dropping to maintain smooth playback
- Network bandwidth requirements remain similar (same frame size, just different timing)

**Combined Use Case Example**:

```javascript
// User workflow combining all three features:

// 1. Select date
setStartDate('2025-12-15'); // Select December 15, 2025

// 2. Drag timeline to 2:30 PM (52200 seconds since midnight)
setTimeRangeValue([52200, 86399]);
setStartSecond(52200);

// 3. Set speed to 2x
setPlaySpeed(2);

// 4. Start playback
handleRefreshPlayback(); // Plays from Dec 15, 2025 at 14:30:00 at 2x speed

// Resulting playback request:
{
    SessionId: "guid-here",
    CameraConfigId: 123,
    StartTime: "2025-12-15 14:30", // Date + Timeline position
    EndTime: "2025-12-15 14:30",
    Width: 1920,
    Height: 1080,
    Speed: 2 // 2x playback speed
}
```

---

## Flow Diagrams

### Complete Video Playback Sequence Diagram

```mermaid
sequenceDiagram
    participant User as User Browser
    participant Frontend as React Frontend
    participant VideoAPI as Video API Service
    participant WSServer as WebSocket Server
    participant VideoHelper as Video Helper<br/>(Milestone/Dahua)
    participant VideoSource as Video Source<br/>(Milestone/Dahua Server)

    User->>Frontend: Select camera and time range
    Frontend->>VideoAPI: GET /playbackurl?sessionId=xxx
    VideoAPI->>Frontend: Return {url: ws://..., sessionId: xxx}
    
    Frontend->>WSServer: Connect WebSocket (ws://.../{sessionId})
    WSServer->>Frontend: Connection established
    
    Frontend->>VideoAPI: POST /startplayback<br/>{sessionId, cameraId, startTime, endTime}
    VideoAPI->>VideoHelper: StartVideo(playbackInput)
    
    alt Milestone Source
        VideoHelper->>VideoSource: Connect via Milestone Mobile SDK
        VideoSource->>VideoHelper: Return JPEG frames
    else Dahua Source
        VideoHelper->>VideoSource: API call to get RTSP URL
        VideoSource->>VideoHelper: Return RTSP URL
        VideoHelper->>VideoHelper: Start FFmpeg process with RTSP
        Note over VideoHelper: FFmpeg converts RTSP<br/>to JPEG frames (15 FPS)
    end
    
    loop Every frame (~15 FPS)
        VideoHelper->>WSServer: Send frame via WebSocket<br/>{image: base64, time: timestamp}
        WSServer->>Frontend: WebSocket message
        Frontend->>User: Display frame in <img> tag
    end
    
    User->>Frontend: Stop playback
    Frontend->>VideoAPI: POST /stopplayback/{sessionId}
    VideoAPI->>VideoHelper: CloseVideo(sessionId)
    VideoHelper->>VideoSource: Close connection/Stop FFmpeg
    VideoAPI->>WSServer: CloseConnection(sessionId)
    WSServer->>Frontend: WebSocket closed
    Frontend->>User: Video stopped
```

### System Architecture Flowchart

```mermaid
flowchart TD
    A[User opens Video Page] --> B{Camera Type?}
    
    B -->|Milestone| C[Get Camera Config<br/>with Milestone Controller]
    B -->|Dahua| D[Get Camera Config<br/>with Dahua Controller]
    
    C --> E[Connect to<br/>MilestoneMediaServices]
    D --> F[Connect to<br/>DahuaMediaServices]
    
    E --> G[Request WebSocket URL]
    F --> G
    
    G --> H[Establish WebSocket Connection]
    
    H --> I[Send Start Playback Request]
    
    I --> J{Video Source?}
    
    J -->|Milestone| K[Connect to Milestone Server<br/>via Mobile SDK]
    J -->|Dahua| L[Get RTSP URL from<br/>Dahua DSS Pro]
    
    K --> M[Receive JPEG Frames<br/>from Milestone]
    L --> N[Start FFmpeg Process]
    N --> O[Convert RTSP to<br/>JPEG Frames]
    
    M --> P[Send Frames via<br/>WebSocket]
    O --> P
    
    P --> Q[Frontend Receives<br/>Base64 Image Data]
    
    Q --> R[Display Frame in<br/>Video Player]
    
    R --> S{Continue Playing?}
    S -->|Yes| P
    S -->|No| T[Send Stop Request]
    
    T --> U[Close Video Source<br/>Connection]
    U --> V[Close WebSocket]
    V --> W[End]
```

### Data Format Flow

```mermaid
flowchart LR
    subgraph Milestone Path
        MS[Milestone Server] -->|JPEG Frames| MSH[MilestoneHelper]
    end
    
    subgraph Dahua Path
        DH[Dahua DSS Pro] -->|RTSP Stream| FF[FFmpeg]
        FF -->|JPEG Frames<br/>stdout| DHH[DahuaHelper]
    end
    
    MSH --> WS[WebSocket Server]
    DHH --> WS
    
    WS -->|"JSON Message<br/>image: base64<br/>time: timestamp"| FE[Frontend]
    
    FE -->|data:image/png;base64| IMG[Image Tag]
    IMG -->|Rendered| USER[User Display]
```

---

## Key Technical Details

### Video Format
- **Milestone**: Direct JPEG frames from Milestone Mobile Service API
- **Dahua**: RTSP stream converted to MJPEG (Motion JPEG) at 15 FPS using FFmpeg

### Communication Protocols
- **HTTP API**: Used for control operations (start, stop, configuration)
- **WebSocket**: Used for streaming video frame data
- **WSS**: WebSocket Secure for encrypted transmission in production

### Frame Rate
- Target: 15 FPS (frames per second)
- Controlled by FFmpeg parameters for Dahua
- Milestone frame rate depends on server configuration

### Image Encoding
- Video frames are JPEG-encoded images
- Transmitted as base64-encoded strings in JSON messages
- Frontend displays using HTML `<img>` tag with data URL

### Session Management
- Each video stream has a unique session ID (GUID)
- Session ID links WebSocket connection to video source
- Multiple concurrent streams supported via concurrent dictionary

### Resource Management
- **Milestone**: Uses VideoOS Mobile SDK connection pool
- **Dahua**: Each session spawns a dedicated FFmpeg process
- Automatic cleanup on disconnect or timeout (30 seconds)
- Ping/pong mechanism keeps connections alive (60-second interval)

### Error Handling
- WebSocket timeout detection
- FFmpeg process monitoring
- Automatic reconnection attempts in frontend
- Graceful degradation and error messaging to users

---

## Configuration

### Milestone Configuration
Location: [`video/MilestoneMediaServices/appsettings.json`]

```json
{
  "MilestoneConfiguration": {
    "ServerUrl": "https://milestone-server:port",
    "Username": "...",
    "Password": "..."
  },
  "WebSocketServerUrl": "ws://localhost:5000/media/ws"
}
```

### Dahua Configuration
Location: [`video/DahuaMediaServices/DahuaMediaServices/appsettings.json`]

```json
{
  "DahuaConfiguration": {
    "ServerUrl": "https://dahua-dss-pro:port",
    "Username": "...",
    "Password": "..."
  },
  "WebSocketServerUrl": "ws://localhost:5001/media/ws"
}
```

---

## Deployment Considerations

### FFmpeg Installation (Required for Dahua)
- FFmpeg must be installed on the server running DahuaMediaServices
- Common installation paths:
  - Windows: `C:\ffmpeg\bin\ffmpeg.exe`
  - Linux: `/usr/bin/ffmpeg` or `/usr/local/bin/ffmpeg`
- The service automatically searches common paths

### Network Configuration
- WebSocket ports must be open for client connections
- RTSP ports (typically 554) must be accessible from DahuaMediaServices to Dahua servers
- Milestone Mobile Service ports must be accessible

### Performance Tuning
- **Frame Rate**: Adjust FFmpeg FPS parameter based on network bandwidth
- **Image Quality**: Adjust FFmpeg quality parameter (`-q:v`) for balance between quality and bandwidth
- **Concurrent Sessions**: Monitor server resources for multiple simultaneous video streams
- **WebSocket Timeouts**: Tune ping interval and timeout values based on network conditions

---

## Summary

The video feature in SCCS backend provides a unified interface for streaming video from both Milestone and Dahua sources. The key innovation is the use of **image frame streaming over WebSocket** rather than traditional video protocols, which provides:

1. **Browser Compatibility**: Works in all modern browsers without plugins
2. **Flexibility**: Easy to implement custom controls and timeline features
3. **Scalability**: Session-based architecture supports multiple concurrent users
4. **Simplicity**: Standard HTTP + WebSocket protocols, no specialized video streaming infrastructure

The system architecture separates concerns effectively:
- **Frontend**: React components for UI and display
- **Video Services**: Dedicated microservices for each video source type
- **LegacyAPI**: Legacy integration layer (not actively used for video currently)
- **WebSocket**: Efficient real-time frame transmission

This design provides a robust, maintainable solution for security camera video integration in the SCCS platform.

---

## Document Metadata

- **Created**: December 16, 2025
- **Author**: GitHub Copilot (AI Assistant)
- **Version**: 1.0
- **Last Updated**: December 16, 2025

