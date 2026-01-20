# OPC UA Status Sync Issue - Analysis & Solutions

**Issue Date:** January 7, 2026  
**Status:** Investigation Complete  
**Priority:** High  
**Module:** OPC UA Building Controls

---

## Problem Statement

Building control status updates are not syncing in real-time to the frontend. Users must manually refresh the page to see updated equipment status (doors, lights, sensors, etc.). This impacts the system's ability to provide real-time monitoring of building automation systems.

### Symptoms

- Equipment status displays stale data
- Control actions appear to succeed but UI doesn't reflect changes
- Manual page refresh shows correct status
- Intermittent updates - sometimes works, sometimes doesn't
- No error messages in UI when sync fails

---

## Root Cause Analysis

### 1. Critical Bug: WebSocket State Update (🔴 CRITICAL)

**Location:** [src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/opcuawidget.js](../src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/opcuawidget.js#L111-L115)

**Current Code:**
```javascript
webSocket.bind("node.updated", "opcua_widget", (nodeObj) => {
    if (opcuaNodeObj.Id == nodeObj.Id) {
        setOpcuaNodeObj(...opcuaNodeObj)  // ❌ BUG: Spreads OLD state, not new data
    }
});
```

**Issue:** 
- The spread operator `...opcuaNodeObj` creates a shallow copy of the **existing** state
- New data from `nodeObj` is completely ignored
- React state never updates with new values from WebSocket
- This is likely the **primary cause** of the sync issue

**Impact:** HIGH - Prevents all WebSocket updates from reflecting in UI

---

### 2. Aggressive Response Caching (🔴 CRITICAL)

**Location:** Multiple controllers in [src/opcua/src/OpcuaApi/Controllers/](../src/opcua/src/OpcuaApi/Controllers/)

**Current Implementation:**
```csharp
[ResponseCache(Location = ResponseCacheLocation.Any, Duration = 3600, VaryByHeader = "tenant")]
public class OpcuaController : ControllerBase
```

**Issue:**
- API responses cached for **3600 seconds (1 hour)**
- Even after database updates, cached responses are served
- Manual refresh may still show stale data if cache hasn't expired
- Applies to:
  - `DataController.cs` - Line 20
  - `OpcuaController.cs` - Line 24
  - `SchemaController.cs` - Line 17

**Impact:** HIGH - Causes stale data to persist even after manual refresh

---

### 3. Conditional WebSocket Broadcasting (⚠️ MEDIUM)

**Location:** [src/opcua/src/OpcuaApi/Controllers/OpcuaController.cs](../src/opcua/src/OpcuaApi/Controllers/OpcuaController.cs#L253-L259)

**Current Code:**
```csharp
if (updateWebsocket) {
    if (checkValue && node.LatestValues != node.PreviousValues) {
        _notifSvc.WebSocket("opcua", "node.updated", result);
    } else if (!checkValue) {
        _notifSvc.WebSocket("opcua", "node.updated", result);
    }
}
```

**Issue:**
- WebSocket broadcast only happens if:
  - `updateWebsocket = true` (default: true)
  - `checkValue = false` OR value actually changed
- If external system sends duplicate values, no broadcast occurs
- Calling code might pass `updateWebsocket = false`

**Scenarios Where No Broadcast Occurs:**
1. Status remains "On" → Update to "On" (with `checkValue=true`)
2. API called with `updateWebsocket=false` parameter
3. Server status updates may skip broadcast

**Impact:** MEDIUM - Some legitimate updates may not broadcast

---

### 4. WebSocket Connection Reliability (⚠️ MEDIUM)

**Location:** [src/core-frontend/MozartCoreFrontend/ClientApp/src/services/WebSocketHelper.js](../src/core-frontend/MozartCoreFrontend/ClientApp/src/services/WebSocketHelper.js)

**Current Implementation:**
```javascript
this._retryCount = 10;
// ...
if ($self._attempt >= $self._retryCount) {
    console.warn(`Websocket failed to connect after ${$self._retryCount} retries.`);
    $self._connected = false;
    return;  // Stops trying
}
```

**Issues:**
- After 10 failed connection attempts, WebSocket stops retrying
- No user notification when connection fails permanently
- 5-second retry delay may be too long for real-time monitoring
- Users unaware that real-time updates are disabled

**Impact:** MEDIUM - Silent failures leave users with stale data

---

### 5. Database Context Caching (⚠️ MEDIUM)

**Location:** Multiple locations using `CacheGet<T>()`

**Example:**
```csharp
public IEnumerable<Node> Get_Nodes() => _dbContext.CacheGet<Node>().ToList();
```

**Issue:**
- In-memory cache may serve stale data
- Cache not invalidated after updates
- Multiple cache layers (response cache + entity cache)

**Impact:** MEDIUM - Compounds caching issues

---

### 6. Message Queue Processing (⚠️ LOW)

**Location:** [src/opcua/src/OpcuaApi/Services/OpcuaServiceListener.cs](../src/opcua/src/OpcuaApi/Services/OpcuaServiceListener.cs#L59-L72)

**Current Code:**
```csharp
private void mapEquipmentStatusMqSubscriber_OnMessageReceived(string routingKey, string jsonData)
{
    try {
        HandleMessageAsync(routingKey, jsonData, CancellationToken.None)
            .GetAwaiter().GetResult();  // Blocking async call
    }
    catch (Exception ex) {
        Serilog.Log.Error(ex, "Unhandled error while processing message.");
    }
}
```

**Issues:**
- Blocking async operations can cause delays
- Exception handling might not ACK/NACK properly
- Message processing failures may go unnoticed

**Impact:** LOW - May cause occasional delays but not primary issue

---

## Solutions & Implementation Plan

### Phase 1: Critical Fixes (Immediate - Day 1)

#### Fix 1.1: Correct WebSocket State Update
**Priority:** 🔴 CRITICAL  
**Estimated Time:** 15 minutes  
**File:** `src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/opcuawidget.js`

**Change:**
```javascript
// BEFORE (Line 111-115)
webSocket.bind("node.updated", "opcua_widget", (nodeObj) => {
    if (opcuaNodeObj.Id == nodeObj.Id) {
        setOpcuaNodeObj(...opcuaNodeObj)  // ❌ Wrong
    }
});

// AFTER
webSocket.bind("node.updated", "opcua_widget", (nodeObj) => {
    if (opcuaNodeObj.Node?.Id === nodeObj.Node?.Id) {
        setOpcuaNodeObj(nodeObj);  // ✅ Correct
    }
});
```

**Testing:**
1. Open browser console
2. Trigger node status change from backend
3. Verify `opcuaNodeObj` state updates in React DevTools
4. Confirm UI reflects new status without refresh

---

#### Fix 1.2: Disable Cache for Status Endpoints
**Priority:** 🔴 CRITICAL  
**Estimated Time:** 30 minutes  
**Files:** 
- `src/opcua/src/OpcuaApi/Controllers/DataController.cs`
- `src/opcua/src/OpcuaApi/Controllers/OpcuaController.cs`

**Changes:**

**DataController.cs** - Line 77:
```csharp
// Add new attribute for real-time status endpoints
[HttpGet("servers")]
[ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
public IEnumerable<ServerNoSensitive> Get_Servers() => ...

[HttpGet("nodes/{id}/details")]
[ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
public NodeWithStatus Get_Node_Status(int id) => ...
```

**OpcuaController.cs** - Keep class-level cache but override for status methods:
```csharp
[HttpGet("nodes/{id}/status")]
[ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
public IActionResult Get_Node_Current_Status(int id) {
    var node = _dataController.Get_Node_Status(id);
    return Ok(node);
}
```

**Testing:**
1. Make API call: `GET /nodes/1/details`
2. Check response headers for `Cache-Control: no-store, no-cache`
3. Update node status in database
4. Verify immediate API call returns new value

---

#### Fix 1.3: Clear Cache After Updates
**Priority:** 🔴 CRITICAL  
**Estimated Time:** 20 minutes  
**File:** `src/opcua/src/OpcuaApi/Controllers/OpcuaController.cs`

**Change at Line 248:**
```csharp
private async Task<IActionResult> Update_Node_Function(Node node, string value, bool updateWebsocket, bool checkValue) {
    if (node == null) {
        return NotFound("Node not found.");
    }
    
    node.PreviousValues = node.LatestValues;
    node.PreviousValuesTimestamp = node.LatestValuesTimestamp;
    node.LatestValues = value;
    node.LatestValuesTimestamp = DateTime.Now;

    await _dbContext.SaveChangesAsync();
    
    // ✅ ADD: Clear cache after update
    _dbContext.CacheClear<Node>();
    
    NodeWithStatus result = _dataController.Get_Node_Status(node.Id);

    if (updateWebsocket) {
        if (checkValue && node.LatestValues != node.PreviousValues) {
            _notifSvc.WebSocket("opcua", "node.updated", result);
        } else if (!checkValue) {
            _notifSvc.WebSocket("opcua", "node.updated", result);
        }
    }

    return Ok(result);
}
```

**Similar changes needed in:**
- `Update_Server_Function` (Line 85)

**Testing:**
1. Update node via API
2. Immediately call GET endpoint
3. Verify returned data reflects update

---

### Phase 2: Reliability Improvements (Week 1)

#### Fix 2.1: Add WebSocket Connection Status Indicator
**Priority:** ⚠️ HIGH  
**Estimated Time:** 2 hours  
**Files:** 
- `src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/opcuawidget.js`
- `src/core-frontend/MozartCoreFrontend/ClientApp/src/pages/map/statusMap.js`

**Implementation:**
```javascript
// In opcuawidget.js
const [wsConnected, setWsConnected] = useState(false);
const [lastUpdateTime, setLastUpdateTime] = useState(null);

useEffect(() => {
    const checkConnection = setInterval(() => {
        const connected = webSocket.conn?.connected || false;
        setWsConnected(connected);
        
        if (!connected) {
            console.warn('OPC UA WebSocket disconnected');
        }
    }, 5000);
    
    return () => clearInterval(checkConnection);
}, [webSocket]);

// Add visual indicator in UI
{!wsConnected && (
    <Alert severity="warning">
        Real-time updates unavailable. Data may be stale.
        <Button onClick={refreshData}>Refresh Manually</Button>
    </Alert>
)}
```

**Testing:**
1. Disconnect network
2. Verify warning appears after detection
3. Reconnect network
4. Verify indicator updates

---

#### Fix 2.2: Implement Polling Fallback
**Priority:** ⚠️ HIGH  
**Estimated Time:** 3 hours  
**File:** `src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/opcuawidget.js`

**Implementation:**
```javascript
useEffect(() => {
    let pollInterval = null;
    
    if (!wsConnected) {
        // Fallback to polling when WebSocket disconnected
        console.log('Starting polling fallback (10s interval)');
        pollInterval = setInterval(async () => {
            try {
                const updatedNode = await getOpcuaNodeDetails(opcuaAssetObj.NodeId);
                setOpcuaNodeObj(updatedNode);
                setLastUpdateTime(new Date());
            } catch (err) {
                console.error('Polling failed:', err);
            }
        }, 10000);
    } else {
        console.log('WebSocket connected, polling disabled');
    }
    
    return () => {
        if (pollInterval) {
            clearInterval(pollInterval);
        }
    };
}, [wsConnected, opcuaAssetObj.NodeId]);
```

**Testing:**
1. Disconnect WebSocket
2. Verify polling starts (check console)
3. Update node status
4. Confirm UI updates within 10 seconds
5. Reconnect WebSocket
6. Verify polling stops

---

#### Fix 2.3: Improve WebSocket Broadcasting Logic
**Priority:** ⚠️ MEDIUM  
**Estimated Time:** 1 hour  
**File:** `src/opcua/src/OpcuaApi/Controllers/OpcuaController.cs`

**Change at Line 253:**
```csharp
private async Task<IActionResult> Update_Node_Function(Node node, string value, bool updateWebsocket, bool checkValue) {
    if (node == null) {
        return NotFound("Node not found.");
    }
    
    node.PreviousValues = node.LatestValues;
    node.PreviousValuesTimestamp = node.LatestValuesTimestamp;
    node.LatestValues = value;
    node.LatestValuesTimestamp = DateTime.Now;

    await _dbContext.SaveChangesAsync();
    _dbContext.CacheClear<Node>();
    
    NodeWithStatus result = _dataController.Get_Node_Status(node.Id);

    // ✅ IMPROVED: Always broadcast with metadata
    if (updateWebsocket) {
        var broadcastData = new {
            Node = result,
            ValueChanged = node.LatestValues != node.PreviousValues,
            UpdatedAt = node.LatestValuesTimestamp
        };
        
        _notifSvc.WebSocket("opcua", "node.updated", broadcastData);
        
        Serilog.Log.Information(
            "WebSocket broadcast for Node {NodeId}: {NodeName}, ValueChanged: {Changed}",
            node.Id, node.Name, broadcastData.ValueChanged
        );
    }

    return Ok(result);
}
```

**Benefits:**
- Always broadcasts updates (unless explicitly disabled)
- Includes metadata about whether value changed
- Improves logging for debugging

**Testing:**
1. Send duplicate status update
2. Verify WebSocket broadcast occurs
3. Check logs for broadcast confirmation
4. Verify frontend receives message

---

#### Fix 2.4: Enhance WebSocket Retry Logic
**Priority:** ⚠️ MEDIUM  
**Estimated Time:** 1 hour  
**File:** `src/core-frontend/MozartCoreFrontend/ClientApp/src/services/WebSocketHelper.js`

**Changes:**
```javascript
constructor(url) {
    // ... existing code ...
    this._retryCount = -1;  // ✅ Infinite retries
    this._retryDelay = 5000;
    this._maxRetryDelay = 60000;  // Max 60 seconds
    
    // ... rest of constructor ...
}

this._ws.onclose = function (wsEvent) {
    // ✅ IMPROVED: Exponential backoff
    const delay = Math.min(
        $self._retryDelay * Math.pow(1.5, Math.min($self._attempt, 10)),
        $self._maxRetryDelay
    );
    
    console.warn(`Websocket disconnected. Retrying in ${delay/1000}s... (attempt ${$self._attempt + 1})`);
    $self._connected = false;
    setTimeout($self._initWS, delay);
}
```

**Benefits:**
- Never gives up reconnecting
- Exponential backoff prevents server overload
- Better user experience

**Testing:**
1. Start app with WebSocket server down
2. Verify increasing retry delays in console
3. Start WebSocket server
4. Verify connection succeeds
5. Disconnect after connection
6. Verify retry behavior

---

### Phase 3: Advanced Improvements (Week 2-3)

#### Enhancement 3.1: Migrate to SignalR
**Priority:** 💡 NICE-TO-HAVE  
**Estimated Time:** 2-3 days  
**Benefits:**
- Automatic reconnection with exponential backoff
- Better error handling
- Connection state management
- Cross-platform compatibility
- Transport fallback (WebSocket → Server-Sent Events → Long Polling)

**Implementation Outline:**
```csharp
// Backend: Add SignalR hub
public class OpcuaHub : Hub {
    public async Task SubscribeToNode(int nodeId) {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"node_{nodeId}");
    }
}

// Broadcast updates
await _hubContext.Clients.Group($"node_{nodeId}")
    .SendAsync("NodeUpdated", result);
```

```javascript
// Frontend: SignalR client
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
    .withUrl("/opcuahub")
    .withAutomaticReconnect()
    .build();

connection.on("NodeUpdated", (data) => {
    setOpcuaNodeObj(data);
});
```

---

#### Enhancement 3.2: Event Sourcing & Replay
**Priority:** 💡 NICE-TO-HAVE  
**Estimated Time:** 1 week  
**Benefits:**
- Track all state changes
- Replay missed events after disconnect
- Audit trail for compliance
- Eventual consistency guaranteed

**Implementation Outline:**
```csharp
// Store events
public class NodeStatusEvent {
    public int Id { get; set; }
    public int NodeId { get; set; }
    public string Value { get; set; }
    public DateTime Timestamp { get; set; }
    public long SequenceNumber { get; set; }
}

// On reconnect, frontend requests missed events
[HttpGet("nodes/{id}/events/since/{sequenceNumber}")]
public IEnumerable<NodeStatusEvent> GetEventsSince(int id, long sequenceNumber) {
    return _dbContext.NodeStatusEvents
        .Where(e => e.NodeId == id && e.SequenceNumber > sequenceNumber)
        .OrderBy(e => e.SequenceNumber);
}
```

---

#### Enhancement 3.3: Health Check Endpoints
**Priority:** 💡 NICE-TO-HAVE  
**Estimated Time:** 4 hours  

**Implementation:**
```csharp
// OpcuaController.cs
[HttpGet("health/websocket")]
public IActionResult GetWebSocketHealth() {
    return Ok(new {
        WebSocketEnabled = true,
        ActiveConnections = _notifSvc.ConnectionCount,
        LastBroadcast = _notifSvc.LastBroadcastTime,
        Status = "healthy"
    });
}

[HttpGet("health/messagequeue")]
public IActionResult GetMessageQueueHealth() {
    return Ok(new {
        Connected = _mapEquipmentStatusMq.IsConnected,
        MessagesProcessed = _messageCounter,
        LastMessage = _lastMessageTime,
        Status = "healthy"
    });
}
```

---

## Testing Strategy

### Unit Tests

```csharp
[Fact]
public async Task Update_Node_Should_Clear_Cache() {
    // Arrange
    var node = CreateTestNode();
    _dbContext.Nodes.Add(node);
    await _dbContext.SaveChangesAsync();
    
    // Act
    await _controller.Update_Node_Id_String("new_value", node.Id);
    
    // Assert
    var cachedNodes = _dbContext.CacheGet<Node>();
    Assert.Empty(cachedNodes); // Cache should be cleared
}

[Fact]
public async Task Update_Node_Should_Broadcast_WebSocket() {
    // Arrange
    var node = CreateTestNode();
    var mockNotifSvc = new Mock<NotificationService>();
    var controller = new OpcuaController(_dbContext, mockNotifSvc.Object);
    
    // Act
    await controller.Update_Node_Id_String("new_value", node.Id);
    
    // Assert
    mockNotifSvc.Verify(
        x => x.WebSocket("opcua", "node.updated", It.IsAny<NodeWithStatus>()),
        Times.Once
    );
}
```

### Integration Tests

```javascript
describe('OPC UA Widget Real-time Updates', () => {
    it('should update UI when WebSocket message received', async () => {
        // Arrange
        const { getByText } = render(<OPCUAWidget opcuaAssetId={1} />);
        await waitFor(() => expect(getByText(/Status/)).toBeInTheDocument());
        
        // Act
        mockWebSocket.triggerMessage({
            subject: 'node.updated',
            body: { Node: { Id: 1, LatestValues: 'On' } }
        });
        
        // Assert
        await waitFor(() => {
            expect(getByText(/On/)).toBeInTheDocument();
        });
    });
    
    it('should fallback to polling when WebSocket disconnected', async () => {
        // Arrange
        mockWebSocket.disconnect();
        const { getByText } = render(<OPCUAWidget opcuaAssetId={1} />);
        
        // Act
        jest.advanceTimersByTime(10000); // Fast-forward 10s
        
        // Assert
        expect(mockApi.get).toHaveBeenCalledWith(
            expect.stringContaining('/nodes/1/details')
        );
    });
});
```

### Manual Testing Checklist

- [ ] **Test 1:** Normal WebSocket Updates
  1. Open status map page
  2. Trigger node status change via external system
  3. Verify UI updates within 2 seconds
  4. No manual refresh required

- [ ] **Test 2:** Cache Invalidation
  1. Update node status
  2. Immediately call API endpoint
  3. Verify fresh data returned (no cache)

- [ ] **Test 3:** WebSocket Disconnect Handling
  1. Open status map
  2. Disconnect network
  3. Verify warning indicator appears
  4. Verify polling fallback starts
  5. Reconnect network
  6. Verify WebSocket reconnects
  7. Verify warning clears

- [ ] **Test 4:** Multiple Concurrent Users
  1. Open status map on 3 different browsers
  2. Update node status
  3. Verify all 3 browsers update simultaneously

- [ ] **Test 5:** High-Frequency Updates
  1. Send rapid status changes (5 per second)
  2. Verify UI handles updates without freezing
  3. Verify no memory leaks in browser

- [ ] **Test 6:** Server Restart Recovery
  1. Open status map (connected)
  2. Restart OPC UA API server
  3. Verify WebSocket reconnects automatically
  4. Verify updates resume working

---

## Monitoring & Observability

### Key Metrics to Track

1. **WebSocket Metrics:**
   - Active connections count
   - Connection failure rate
   - Reconnection attempts
   - Average reconnection time
   - Messages broadcast per minute

2. **Cache Metrics:**
   - Cache hit rate
   - Cache invalidation frequency
   - Average response time (cached vs uncached)

3. **Message Queue Metrics:**
   - Messages processed per minute
   - Processing failures
   - Queue depth
   - Average processing time

### Logging Enhancements

```csharp
// Add structured logging
Serilog.Log.Information(
    "Node status updated: NodeId={NodeId}, PrevValue={PrevValue}, NewValue={NewValue}, WebSocketBroadcast={Broadcast}",
    node.Id, node.PreviousValues, node.LatestValues, updateWebsocket
);

Serilog.Log.Warning(
    "WebSocket broadcast skipped: NodeId={NodeId}, Reason={Reason}",
    node.Id, checkValue && node.LatestValues == node.PreviousValues ? "NoValueChange" : "Disabled"
);
```

### Frontend Diagnostics

```javascript
// Add debug panel (dev mode only)
if (process.env.NODE_ENV === 'development') {
    window.opcuaDebug = {
        wsConnected: () => webSocket.conn?.connected,
        lastUpdate: () => lastUpdateTime,
        forceRefresh: () => refreshData(),
        wsHandlers: () => webSocket._handlers
    };
}
```

---

## Rollback Plan

If issues occur after deployment:

1. **Immediate Rollback (< 5 minutes):**
   ```bash
   git revert <commit-hash>
   docker-compose restart opcua-api
   docker-compose restart core-frontend
   ```

2. **Partial Rollback:**
   - Revert frontend changes only (if backend stable)
   - Revert cache changes only (if WebSocket fixes stable)

3. **Emergency Mitigation:**
   - Reduce cache duration to 60 seconds (instead of disabling)
   - Keep WebSocket fixes (low risk)
   - Revert polling fallback if causing performance issues

---

## Success Criteria

### Must Have (Phase 1)
- ✅ UI updates within 2 seconds of status change
- ✅ No manual refresh required
- ✅ 99% of status updates sync successfully
- ✅ Cache headers correct on status endpoints
- ✅ WebSocket broadcasts logged

### Should Have (Phase 2)
- ✅ Connection status visible to users
- ✅ Polling fallback works when WebSocket fails
- ✅ WebSocket reconnects automatically
- ✅ No silent failures

### Nice to Have (Phase 3)
- ✅ SignalR implementation
- ✅ Event sourcing
- ✅ Health check endpoints
- ✅ Comprehensive monitoring dashboard

---

## Timeline

| Phase | Duration | Start | End | Status |
|-------|----------|-------|-----|--------|
| Phase 1: Critical Fixes | 1-2 hours | Day 1 AM | Day 1 PM | 🔄 Planned |
| Testing & Validation | 2-3 hours | Day 1 PM | Day 1 EOD | 🔄 Planned |
| Phase 2: Reliability | 3-4 days | Week 1 | Week 1 | 🔄 Planned |
| Phase 3: Advanced | 2-3 weeks | Week 2 | Week 3-4 | 💡 Optional |

---

## Related Documentation

- [Building Controls Documentation](building-controls-documentation.md) - Module overview
- [OPC UA Module README](../src/opcua/README.md) - Technical details
- [WebSocket Helper Implementation](../src/core-frontend/MozartCoreFrontend/ClientApp/src/services/WebSocketHelper.js) - WebSocket logic

---

## Appendix: Code References

### Files Requiring Changes

**Critical (Phase 1):**
1. `src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/opcuawidget.js` - Line 111
2. `src/opcua/src/OpcuaApi/Controllers/DataController.cs` - Line 20, 77
3. `src/opcua/src/OpcuaApi/Controllers/OpcuaController.cs` - Line 24, 248, 85

**Medium Priority (Phase 2):**
4. `src/core-frontend/MozartCoreFrontend/ClientApp/src/pages/map/statusMap.js` - Add connection indicator
5. `src/core-frontend/MozartCoreFrontend/ClientApp/src/services/WebSocketHelper.js` - Line 8 (retry logic)
6. `src/opcua/src/OpcuaApi/Services/OpcuaServiceListener.cs` - Error handling

### Testing Locations

- Unit tests: `tests/opcua/OpcuaControllerTests.cs` (to be created)
- Integration tests: `tests/integration/OpcuaWebSocketTests.cs` (to be created)
- Frontend tests: `src/core-frontend/MozartCoreFrontend/ClientApp/src/components/controls/__tests__/opcuawidget.test.js` (to be created)

---

**Document Version:** 1.0  
**Last Updated:** January 7, 2026  
**Author:** SCCS Backend Team  
**Review Status:** Ready for Implementation
