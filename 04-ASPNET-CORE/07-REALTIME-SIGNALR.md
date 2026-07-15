# REAL-TIME COMMUNICATION WITH SIGNALR
## Exam-Style Study Notes

---

## TOPIC: SignalR in ASP.NET Core

### WHY? (Problem Statement)

**Before SignalR:**
- Polling: Client asks "any updates?" every N seconds → Wasteful, latency
- Long Polling: Server holds request until data → Complex, connection limits
- WebSockets: Manual implementation, no fallback, no reconnection, no groups
- Server-Sent Events: One-way only, no binary, limited browser support

**What SignalR Solves:**
| Problem | SignalR Solution |
|---------|------------------|
| Transport selection | Auto-negotiates: WebSockets → SSE → Long Polling |
| Reconnection | Built-in with configurable retry |
| Connection lifecycle | OnConnected/Disconnected/Reconnected |
| Groups/User targeting | `Clients.Group()`, `Clients.User()` |
| Scale-out | Redis/Azure Service Bus backplane |
| Type safety | Strongly-typed hubs (TypeScript client) |
| Authorization | `[Authorize]` on hubs/methods |

**Real-World Analogy:**
- Polling = Checking mailbox every 5 minutes
- SignalR = Mail carrier rings doorbell when mail arrives

---

### HOW? (Internal Mechanism)

#### 1. **Connection Negotiation Flow**

```
Client                                         Server
  │                                              │
  ├──── /negotiate (POST) ─────────────────────►│
  │          Returns: connectionId,            │
  │          availableTransports, url          │
  │                                              │
  ├──── WebSocket Upgrade ────────────────────►│
  │         (or SSE/LongPoll)                  │
  │                                              │
  │◄──── Connected (handshake) ────────────────┤
  │                                              │
  │         Real-time messaging begins         │
  │                                              │
```

#### 2. **Transport Selection Priority**

```
1. WebSockets (Best: full-duplex, binary, low overhead)
   ↓ Fails (proxy, firewall, old browser)
2. Server-Sent Events (Good: unidirectional, auto-reconnect)
   ↓ Fails (IE, corporate proxy)
3. Long Polling (Fallback: works everywhere, high latency)
```

#### 3. **Hub Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
                        SignalR Hub
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  public class ChatHub : Hub                                    │
│  {                                                              │
│      // Lifecycle                                               │
│      public override async Task OnConnectedAsync() { }         │
│      public override async Task OnDisconnectedAsync(Exception) { }│
│                                                                 │
│      // Methods (Client → Server)                              │
│      public async Task SendMessage(string user, string msg) { }│
│      public async Task JoinGroup(string groupName) { }         │
│                                                                 │
│      // Clients (Server → Client)                              │
│      // Clients.All, Clients.Caller, Clients.Group,            │
│      // Clients.User, Clients.Client(connectionId)             │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Hub Implementation**

```csharp
// Strongly-typed Hub (Client interface)
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
    Task UserLeft(string user);
    Task GroupMessage(string group, string user, string message);
}

[Authorize]  // Require authentication
public class ChatHub : Hub<IChatClient>
{
    private readonly ILogger<ChatHub> _logger;
    private readonly IConnectionManager _connections;
    
    public ChatHub(ILogger<ChatHub> logger, IConnectionManager connections)
    {
        _logger = logger;
        _connections = connections;
    }
    
    // Connection Lifecycle
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;  // From JWT sub claim
        var userName = Context.User?.Identity?.Name;
        
        // Track connection
        await _connections.AddConnectionAsync(userId, Context.ConnectionId);
        
        // Notify others
        await Clients.Others.UserJoined(userName);
        
        _logger.LogInformation("User {User} connected: {ConnectionId}", userName, Context.ConnectionId);
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        var userName = Context.User?.Identity?.Name;
        
        await _connections.RemoveConnectionAsync(userId, Context.ConnectionId);
        
        await Clients.Others.UserLeft(userName);
        
        _logger.LogInformation("User {User} disconnected: {ConnectionId}", userName, Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
    
    // Client Methods
    public async Task SendMessage(string message)
    {
        var user = Context.User?.Identity?.Name;
        await Clients.All.ReceiveMessage(user, message);
    }
    
    public async Task SendToUser(string targetUserId, string message)
    {
        var user = Context.User?.Identity?.Name;
        await Clients.User(targetUserId).ReceiveMessage(user, message);
    }
    
    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).GroupMessage(groupName, Context.User?.Identity?.Name, "joined");
    }
    
    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).GroupMessage(groupName, Context.User?.Identity?.Name, "left");
    }
    
    public async Task SendToGroup(string groupName, string message)
    {
        await Clients.Group(groupName).GroupMessage(groupName, Context.User?.Identity?.Name, message);
    }
}
```

#### 2. **Client-Side (TypeScript/JavaScript)**

```typescript
// Automatic generated types from hub
import { HubConnection, HubConnectionBuilder, LogLevel } from "@microsoft/signalr";

class ChatService {
    private connection: HubConnection;
    
    constructor(private token: string) {
        this.connection = new HubConnectionBuilder()
            .withUrl("/hubs/chat", {
                accessTokenFactory: () => this.token,
                skipNegotiation: false,
                withCredentials: true
            })
            .withAutomaticReconnect({
                nextRetryDelayInMilliseconds: retryContext => {
                    if (retryContext.previousRetryCount < 3) return 1000;
                    if (retryContext.previousRetryCount < 5) return 5000;
                    return 30000; // Max 30s
                }
            })
            .configureLogging(LogLevel.Information)
            .build();
        
        this.registerHandlers();
    }
    
    private registerHandlers() {
        this.connection.on("ReceiveMessage", (user: string, message: string) => {
            this.onMessageReceived(user, message);
        });
        
        this.connection.on("UserJoined", (user: string) => {
            this.onUserJoined(user);
        });
        
        this.connection.on("GroupMessage", (group: string, user: string, message: string) => {
            this.onGroupMessage(group, user, message);
        });
        
        this.connection.onreconnecting(error => {
            console.log("Reconnecting...", error);
            this.onReconnecting();
        });
        
        this.connection.onreconnected(connectionId => {
            console.log("Reconnected:", connectionId);
            this.onReconnected();
        });
        
        this.connection.onclose(error => {
            console.log("Disconnected:", error);
            this.onDisconnected();
        });
    }
    
    async start() {
        try {
            await this.connection.start();
            console.log("Connected:", this.connection.connectionId);
        } catch (err) {
            console.error("Connection failed:", err);
            setTimeout(() => this.start(), 5000);
        }
    }
    
    async sendMessage(message: string) {
        await this.connection.invoke("SendMessage", message);
    }
    
    async sendToUser(targetUserId: string, message: string) {
        await this.connection.invoke("SendToUser", targetUserId, message);
    }
    
    async joinGroup(groupName: string) {
        await this.connection.invoke("JoinGroup", groupName);
    }
    
    async stop() {
        await this.connection.stop();
    }
    
    // Events
    onMessageReceived: (user: string, message: string) => void = () => {};
    onUserJoined: (user: string) => void = () => {};
    onGroupMessage: (group: string, user: string, message: string) => void = () => {};
    onReconnecting: () => void = () => {};
    onReconnected: () => void = () => {};
    onDisconnected: () => void = () => {};
}
```

#### 3. **Server Configuration (Program.cs)**

```csharp
var builder = WebApplication.CreateBuilder(args);

// SignalR with JSON protocol (or MessagePack for performance)
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    options.MaximumReceiveMessageSize = 1024 * 1024; // 1MB
    options.StreamBufferCapacity = 10;
})
.AddJsonProtocol(options =>
{
    options.PayloadSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
})
// .AddMessagePackProtocol() // For binary performance
.AddStackExchangeRedis("redis-connection", options =>
{
    options.Configuration.ChannelPrefix = "nanolite:signalr";
});

// User ID Provider (for Clients.User())
builder.Services.AddSingleton<IUserIdProvider, CustomUserIdProvider>();

// Connection tracking
builder.Services.AddSingleton<IConnectionManager, ConnectionManager>();

// Authorization
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ChatAccess", p => p.RequireClaim("chat_access"));
});

var app = builder.Build();

// Map Hubs
app.MapHub<ChatHub>("/hubs/chat", options =>
{
    options.Transports = HttpTransportType.WebSockets | HttpTransportType.ServerSentEvents;
    // options.ApplicationMaxBufferSize = 64 * 1024;
});

// Health check for load balancer
app.MapHealthChecks("/health/signalr", new HealthCheckOptions
{
    Predicate = r => r.Name.Contains("signalr")
});

app.Run();

// Custom User ID Provider
public class CustomUserIdProvider : IUserIdProvider
{
    public string GetUserId(HubConnectionContext connection)
    {
        // Use sub claim from JWT
        return connection.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value 
            ?? connection.User?.FindFirst("sub")?.Value;
    }
}
```

#### 4. **Scale-Out with Redis Backplane**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Server 1   │     │  Server 2   │     │  Server 3   │
│  (Hub)      │     │  (Hub)      │     │  (Hub)      │
│             │     │             │     │             │
│  Client A   │     │  Client B   │     │  Client C   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Redis     │
                    │  Pub/Sub    │
                    │  (Backplane)│
                    └─────────────┘

Message Flow:
Client A → Server 1 → Redis Channel → Server 2 → Client B
                    → Server 3 → Client C
```

```csharp
// Redis Backplane Configuration
builder.Services.AddSignalR()
    .AddStackExchangeRedis(redisConnectionString, options =>
    {
        options.Configuration.ChannelPrefix = "myapp:signalr";
        // Configure connection
        options.Configuration.AbortOnConnectFail = false;
        options.Configuration.ConnectRetry = 3;
    });

// For Azure: .AddAzureSignalR(connectionString)
```

#### 5. **Streaming (Server → Client)**

```csharp
// Server: Stream data
public async IAsyncEnumerable<int> StreamNumbers(int count, 
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    for (int i = 0; i < count; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Delay(500, cancellationToken);
        yield return i;
    }
}

// Client: Consume stream
const stream = connection.stream("StreamNumbers", 10);
stream.subscribe({
    next: (item) => console.log("Received:", item),
    complete: () => console.log("Stream completed"),
    error: (err) => console.error("Stream error:", err)
});

// Cancel: stream.cancel()
```

#### 6. **Authorization on Hub Methods**

```csharp
[Authorize]
public class SecureHub : Hub
{
    [Authorize(Policy = "AdminOnly")]
    public async Task AdminOnly(string message)
    {
        await Clients.All.SendAsync("AdminMessage", message);
    }
    
    [Authorize(Roles = "Moderator,Admin")]
    public async Task ModeratorAction(string action)
    {
        // ...
    }
    
    // Custom requirement
    public async Task CustomAction()
    {
        var canAccess = Context.User?.HasClaim("permission", "custom_action");
        if (!canAccess) throw new HubException("Unauthorized");
        // ...
    }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Typed Hubs** | Always | Dynamic `Clients.All.SendAsync()` | Compile-time safety, IntelliSense |
| **Groups for Rooms** | Chat rooms, dashboards | `Clients.All` for everything | Unnecessary traffic |
| **User Mapping** | 1:1 messaging, notifications | ConnectionId tracking | User can have multiple connections |
| **Streaming** | Large data, progress | Chunked SendAsync | Backpressure, cancellation |
| **Redis Backplane** | Multi-server | Sticky sessions | Single point of failure, no scale |
| **Reconnection Config** | Production | Default (no retry) | Poor UX on network blips |

---

### INTERVIEW QUESTIONS (Top 10)

#### 1. **Conceptual**: "How does SignalR choose transport? What if WebSockets fail?"

**Answer:**
```
Negotiation Process:
1. Client POST /negotiate → Server returns available transports + connectionId
2. Client tries WebSockets first (Upgrade header)
3. If fails (400/500/timeout) → Server-Sent Events (EventSource)
4. If fails → Long Polling (POST /poll)

Fallback Order: WebSockets → SSE → Long Polling

Client Config:
.withUrl("/hub", {
    transport: HttpTransportType.WebSockets | HttpTransportType.ServerSentEvents
    // Exclude LongPolling if not wanted
})

Server Config:
options.Transports = HttpTransportType.WebSockets | HttpTransportType.ServerSentEvents;
```

#### 2. **Code**: "Implement a real-time dashboard with server-side streaming"

```csharp
// Hub
public interface IDashboardClient
{
    Task MetricUpdate(string metric, double value);
    Task Alert(string message);
}

public class DashboardHub : Hub<IDashboardClient>
{
    private readonly IMetricsCollector _metrics;
    
    public async Task SubscribeToMetrics(string[] metricNames)
    {
        foreach (var metric in metricNames)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"metric:{metric}");
        }
    }
    
    public async Task UnsubscribeFromMetrics(string[] metricNames)
    {
        foreach (var metric in metricNames)
        {
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"metric:{metric}");
        }
    }
}

// Background Service pushing metrics
public class MetricsBroadcaster : BackgroundService
{
    private readonly IHubContext<DashboardHub, IDashboardClient> _hub;
    private readonly IMetricsCollector _metrics;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var metrics = await _metrics.GetCurrentMetricsAsync();
            
            foreach (var (name, value) in metrics)
            {
                await _hub.Clients.Group($"metric:{name}").MetricUpdate(name, value);
            }
            
            await Task.Delay(1000, stoppingToken); // 1 second updates
        }
    }
}

// Client
const connection = new HubConnectionBuilder()
    .withUrl("/hubs/dashboard")
    .build();

connection.on("MetricUpdate", (metric, value) => {
    updateChart(metric, value);
});

await connection.start();
await connection.invoke("SubscribeToMetrics", ["cpu", "memory", "requests"]);
```

#### 3. **Design**: "How to send notification to specific user across multiple devices?"

```csharp
// UserIdProvider maps JWT sub claim to userId
public class CustomUserIdProvider : IUserIdProvider
{
    public string GetUserId(HubConnectionContext connection)
    {
        return connection.User?.FindFirst("sub")?.Value 
            ?? connection.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    }
}

// Registration
builder.Services.AddSingleton<IUserIdProvider, CustomUserIdProvider>();

// Usage in Hub
public async Task NotifyUser(string userId, string message)
{
    await Clients.User(userId).ReceiveNotification(message);
}

// Sends to ALL connections for that user (web, mobile, desktop)
```

#### 4. **Scaling**: "SignalR with 100K concurrent connections - architecture?"

```
┌─────────────────────────────────────────────────────────────────┐
                        Load Balancer (L4/L7)
                    (Sticky Sessions NOT needed with backplane)
└─────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
   ┌─────────┐                 ┌─────────┐                 ┌─────────┐
   │ Pod 1   │                 │ Pod 2   │                 │ Pod 3   │
   │ SignalR │◄── Redis ──────►│ SignalR │◄── Redis ──────►│ SignalR │
   │  Hub    │    Pub/Sub      │  Hub    │    Pub/Sub      │  Hub    │
   └────┬────┘                 └────┬────┘                 └────┬────┘
        │                           │                           │
        └───────────────────────────┼───────────────────────────┘
                                    │
                           ┌────────▼────────┐
                           │  Redis Cluster  │
                           │  (Backplane +   │
                           │   Connection    │
                           │   State)        │
                           └─────────────────┘

Key Configurations:
- Connection Drain: graceful shutdown (SIGTERM → stop accepting, finish processing)
- Redis: Cluster mode, pipelining, connection pooling
- Metrics: Connections per pod, message throughput, latency p99
- Health Checks: /health/signalr (checks Redis connectivity)
- Circuit Breaker: On Redis failure, degrade to local-only (no cross-pod messages)
```

#### 5. **Code**: "Handle reconnection and recover missed messages"

```csharp
// Server: Store recent messages per group
public class MessageStore
{
    private readonly IDistributedCache _cache;
    private const int MaxMessagesPerGroup = 100;
    
    public async Task AddMessageAsync(string group, Message msg)
    {
        var key = $"messages:{group}";
        var messages = await GetMessagesAsync(group);
        messages.Add(msg);
        if (messages.Count > MaxMessagesPerGroup)
            messages.RemoveRange(0, messages.Count - MaxMessagesPerGroup);
        
        await _cache.SetStringAsync(key, JsonSerializer.Serialize(messages));
    }
    
    public async Task<List<Message>> GetMessagesSinceAsync(string group, DateTime since)
    {
        var all = await GetMessagesAsync(group);
        return all.Where(m => m.Timestamp > since).ToList();
    }
}

// Hub: On reconnect, send missed messages
public override async Task OnReconnectedAsync()
{
    var lastSeen = Context.GetHttpContext()?.Request.Query["lastSeen"];
    if (DateTime.TryParse(lastSeen, out var since))
    {
        var groups = await GetUserGroupsAsync(Context.UserIdentifier);
        foreach (var group in groups)
        {
            var missed = await _messageStore.GetMessagesSinceAsync(group, since);
            foreach (var msg in missed)
            {
                await Clients.Caller.ReceiveMessage(msg);
            }
        }
    }
    await base.OnReconnectedAsync();
}

// Client: Track last message timestamp
let lastMessageTime = new Date().toISOString();

connection.on("ReceiveMessage", (msg) => {
    lastMessageTime = msg.timestamp;
    displayMessage(msg);
});

connection.onreconnected(() => {
    // Re-send missed messages request
    connection.invoke("RequestMissedMessages", lastMessageTime);
});
```

#### 6. **Performance**: "Optimize SignalR for high-frequency updates (100 msg/sec)"

```csharp
// 1. Use MessagePack Protocol (Binary, smaller)
builder.Services.AddSignalR()
    .AddMessagePackProtocol();

// 2. Reduce Hub Method Calls - Use Streaming
public async IAsyncEnumerable<MarketData> StreamMarketData(
    string symbol, 
    [EnumeratorCancellation] CancellationToken ct)
{
    var channel = _marketDataService.Subscribe(symbol);
    await foreach (var data in channel.Reader.ReadAllAsync(ct))
    {
        yield return data;
    }
}

// 3. Batch Updates
public async Task SendBatchUpdates(List<Update> updates)
{
    // Single invocation with array
    await Clients.Group("realtime").BatchUpdate(updates);
}

// 4. Configure Buffer Sizes
builder.Services.AddSignalR(options =>
{
    options.StreamBufferCapacity = 100; // Buffer for slow clients
    options.MaximumReceiveMessageSize = 64 * 1024; // 64KB
    options.EnableDetailedErrors = false; // Disable in prod
});

// 5. Client: Throttle UI Updates
connection.on("BatchUpdate", (updates) => {
    // Batch DOM updates
    requestAnimationFrame(() => {
        updates.forEach(update => applyUpdate(update));
    });
});
```

#### 7. **Security**: "Prevent malicious clients from joining arbitrary groups"

```csharp
public class SecureChatHub : Hub<IChatClient>
{
    private readonly IAuthorizationService _auth;
    
    public async Task JoinGroup(string groupName)
    {
        // Validate group name format
        if (!Regex.IsMatch(groupName, @"^[a-zA-Z0-9-_]{1,50}$"))
            throw new HubException("Invalid group name");
        
        // Authorization check
        var authResult = await _auth.AuthorizeAsync(Context.User, groupName, "GroupAccess");
        if (!authResult.Succeeded)
            throw new HubException("Not authorized for this group");
        
        // Rate limiting
        var joinCount = await _rateLimiter.IncrementAsync($"join:{Context.ConnectionId}");
        if (joinCount > 10)
            throw new HubException("Too many join attempts");
        
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).UserJoined(Context.User?.Identity?.Name);
    }
}

// Policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("GroupAccess", policy =>
        policy.Requirements.Add(new GroupAccessRequirement()));
});

public class GroupAccessHandler : AuthorizationHandler<GroupAccessRequirement, string>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        GroupAccessRequirement requirement, 
        string groupName)
    {
        // Check if user is member of group (from DB)
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        var isMember = _groupService.IsMemberAsync(userId, groupName).Result;
        
        if (isMember) context.Succeed(requirement);
        return Task.CompletedTask;
    }
}
```

#### 8. **Debugging**: "Client receives duplicate messages - troubleshoot"

```
Common Causes:
☐ Multiple hub connections (open tabs, reconnect without stop)
☐ Multiple event registrations (on() called twice)
☐ Server sending to Groups + Users + All for same message
☐ Backplane misconfiguration (multiple Redis prefixes)
☐ Client not cleaning up on navigation (SPA)

Debug Steps:
1. Connection ID logging on server
   _logger.LogInformation("ConnId: {ConnId}, User: {User}", Context.ConnectionId, Context.UserIdentifier);

2. Client: Log every received message with timestamp
   connection.on("Message", (msg) => console.log(Date.now(), msg));

3. Check Groups: 
   await Groups.GetGroupAsync(connectionId); // Extension method

4. Verify single negotiation:
   Network tab → /negotiate should fire once per session

5. Redis MONITOR: Check PUBLISH frequency
```

#### 9. **Testing**: "Unit test Hub methods"

```csharp
public class ChatHubTests
{
    private readonly Mock<IChatClient> _mockClient;
    private readonly Mock<IClientProxy> _mockClients;
    private readonly Mock<HubCallerContext> _mockContext;
    private readonly Mock<IGroupManager> _mockGroups;
    private readonly ChatHub _hub;
    
    public ChatHubTests()
    {
        _mockClient = new Mock<IChatClient>();
        _mockClients = new Mock<IClientProxy>();
        _mockContext = new Mock<HubCallerContext>();
        _mockGroups = new Mock<IGroupManager>();
        
        _mockClients.Setup(c => c.User(It.IsAny<string>())).Returns(_mockClient.Object);
        _mockClients.Setup(c => c.Group(It.IsAny<string>())).Returns(_mockClient.Object);
        _mockClients.Setup(c => c.All).Returns(_mockClient.Object);
        
        var mockCaller = new Mock<HubCallerClients<IChatClient>>();
        mockCaller.Setup(c => c.All).Returns(_mockClient.Object);
        mockCaller.Setup(c => c.User(It.IsAny<string>())).Returns(_mockClient.Object);
        mockCaller.Setup(c => c.Group(It.IsAny<string>())).Returns(_mockClient.Object);
        mockCaller.Setup(c => c.Others).Returns(_mockClient.Object);
        mockCaller.Setup(c => c.Caller).Returns(_mockClient.Object);
        
        _mockContext.Setup(c => c.ConnectionId).Returns("test-connection");
        _mockContext.Setup(c => c.UserIdentifier).Returns("user-123");
        _mockContext.Setup(c => c.User).Returns(new ClaimsPrincipal(new ClaimsIdentity(
            new[] { new Claim(ClaimTypes.NameIdentifier, "user-123"), 
                    new Claim(ClaimTypes.Name, "TestUser") })));
        
        _hub = new ChatHub(Mock.Of<ILogger<ChatHub>>(), Mock.Of<IConnectionManager>())
        {
            Clients = mockCaller.Object,
            Context = _mockContext.Object,
            Groups = _mockGroups.Object
        };
    }
    
    [Fact]
    public async Task SendMessage_CallsReceiveMessageOnAllClients()
    {
        // Act
        await _hub.SendMessage("Hello World");
        
        // Assert
        _mockClient.Verify(c => c.ReceiveMessage("TestUser", "Hello World"), Times.Once);
    }
    
    [Fact]
    public async Task JoinGroup_AddsToGroupAndNotifies()
    {
        // Act
        await _hub.JoinGroup("general");
        
        // Assert
        _mockGroups.Verify(g => g.AddToGroupAsync("test-connection", "general", It.IsAny<CancellationToken>()), Times.Once);
        _mockClient.Verify(c => c.GroupMessage("general", "TestUser", "joined"), Times.Once);
    }
}
```

#### 10. **Architecture**: "SignalR vs gRPC Streaming vs WebSockets - when to use which?"

| Feature | SignalR | gRPC Streaming | Raw WebSockets |
|---------|---------|----------------|----------------|
| **Auto-reconnect** | ✅ Built-in | ❌ Manual | ❌ Manual |
| **Fallback Transports** | ✅ SSE, LongPoll | ❌ HTTP/2 only | ❌ None |
| **Type Safety** | ✅ TypeScript gen | ✅ Protobuf | ❌ Manual |
| **Groups/Users** | ✅ Built-in | ❌ Manual | ❌ Manual |
| **Scale-out** | ✅ Redis/Azure | ❌ Manual | ❌ Manual |
| **Binary** | ✅ MessagePack | ✅ Protobuf | ✅ Native |
| **Browser Support** | ✅ Universal | ⚠️ gRPC-Web needed | ✅ Universal |
| **Use Case** | Real-time UI, Chat, Notifications | Service-to-service, High perf | Custom protocols, Gaming |

---

### GOTCHAS & EDGE CASES

- [ ] **ConnectionId != UserId**: One user = multiple connections (tabs, devices)
- [ ] **Groups are per-server without backplane**: Use Redis for multi-server
- [ ] **OnDisconnected fires on reconnect**: Don't cleanup user state there
- [ ] **Hub lifetime**: Transient per connection, not singleton
- [ ] **Streaming cancellation**: Pass `CancellationToken` to `IAsyncEnumerable`
- [ ] **Message size limit**: Default 32KB, configure for large payloads
- [ ] **KeepAlive**: Default 15s, client timeout 30s - tune for mobile
- [ ] **Authorization**: Hub-level `[Authorize]` doesn't protect individual methods without policies
- [ ] **Client disconnect detection**: Server knows after KeepAlive timeout
- [ ] **Memory leaks**: Unsubscribe from streams, dispose HubConnection

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Hub with All Patterns**
```csharp
[Authorize]
public class NotificationHub : Hub<INotificationClient>
{
    private readonly IConnectionManager _connections;
    private readonly INotificationService _notifications;
    
    public override async Task OnConnectedAsync()
    {
        await _connections.AddAsync(Context.UserIdentifier, Context.ConnectionId);
        await Clients.Caller.Connected(Context.ConnectionId);
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception? ex)
    {
        await _connections.RemoveAsync(Context.UserIdentifier, Context.ConnectionId);
        await base.OnDisconnectedAsync(ex);
    }
    
    public async Task Subscribe(string topic)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"topic:{topic}");
    }
    
    public async Task Unsubscribe(string topic)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"topic:{topic}");
    }
    
    // Called from services
    public static async Task SendToUser(IHubContext<NotificationHub, INotificationClient> hub, 
        string userId, Notification notification)
    {
        await hub.Clients.User(userId).ReceiveNotification(notification);
    }
    
    public static async Task SendToTopic(IHubContext<NotificationHub, INotificationClient> hub,
        string topic, Notification notification)
    {
        await hub.Clients.Group($"topic:{topic}").ReceiveNotification(notification);
    }
}
```

#### 2. **Service Integration (Non-Hub Code)**
```csharp
public class OrderService
{
    private readonly IHubContext<OrderHub, IOrderClient> _hub;
    
    public OrderService(IHubContext<OrderHub, IOrderClient> hub) => _hub = hub;
    
    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        var order = await _repo.CreateAsync(dto);
        
        // Notify user
        await _hub.Clients.User(order.UserId).OrderCreated(order);
        
        // Notify kitchen group
        await _hub.Clients.Group($"restaurant:{order.RestaurantId}").NewOrder(order);
        
        // Notify delivery drivers in area
        await _hub.Clients.Group($"zone:{order.DeliveryZone}").OrderAvailable(order);
        
        return order;
    }
}
```

#### 3. **Client Reconnection with State Recovery**
```typescript
class ResilientConnection {
    private connection: HubConnection;
    private pendingInvocations = new Map<string, (value: any) => void>();
    private invocationId = 0;
    
    async start() {
        this.connection = new HubConnectionBuilder()
            .withUrl("/hubs/chat", { accessTokenFactory: () => getToken() })
            .withAutomaticReconnect([1000, 2000, 5000, 10000, 30000])
            .build();
        
        this.connection.onreconnecting(() => this.onReconnecting());
        this.connection.onreconnected(async (connId) => {
            await this.recoverState();
        });
        
        await this.connection.start();
    }
    
    async invoke<T>(method: string, ...args: any[]): Promise<T> {
        if (this.connection.state === HubConnectionState.Connected) {
            return this.connection.invoke(method, ...args);
        }
        
        // Queue during reconnect
        return new Promise((resolve, reject) => {
            const id = `${++this.invocationId}`;
            this.pendingInvocations.set(id, resolve);
            // Store for replay after reconnect
        });
    }
    
    private async recoverState() {
        // Rejoin groups
        for (const group of this.joinedGroups) {
            await this.connection.invoke("JoinGroup", group);
        }
        
        // Replay pending invocations
        for (const [id, resolve] of this.pendingInvocations) {
            // Re-invoke...
        }
        this.pendingInvocations.clear();
    }
}
```

---

### PRACTICE PROBLEMS

1. **Build**: Real-time collaborative editor (OT/CRDT + SignalR)
2. **Scale**: Design for 1M concurrent connections (sharding, connection draining)
3. **Debug**: Messages delayed by 30s - trace through transport negotiation
4. **Secure**: Implement end-to-end encryption for chat messages
5. **Test**: Load test 10K connections with message broadcasting

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| SignalR Transport Priority | WebSockets → SSE → Long Polling |
| Hub Lifetime | Transient (per connection) |
| Groups vs Users | Groups: logical rooms. Users: all connections for userId |
| Scale-out Mechanism | Redis Pub/Sub Backplane (or Azure SignalR) |
| Typed Hubs | `Hub<TClient>` + `Clients.All.Method()` - compile-time safety |
| Streaming | `IAsyncEnumerable<T>` with `CancellationToken` |
| Reconnection Config | `withAutomaticReconnect([1000, 5000, 30000])` |
| UserIdProvider | Maps ClaimsPrincipal to string userId for `Clients.User()` |
| OnDisconnectedAsync | Fires on graceful disconnect AND failed reconnect |
| KeepAlive/Timeout | Server ping 15s, client timeout 30s (configure in AddSignalR) |

---

## NEXT TOPIC: `08-ARCHITECTURE-PATTERNS.md`

> **Study Tip**: Build a notification system: Hub with typed client, JWT auth, Redis backplane, user/group targeting, background service broadcasting, and TypeScript client with auto-reconnect.