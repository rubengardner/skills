# Specialist E — Real-Time & Async

## THE CORE CONFLICT

The client wants live updates. The backend wants to avoid maintaining long-lived connections at scale.

---

## MEDIATION

**⚙ BACKEND SAYS**

> Polling is fine. The client calls `GET /notifications?since=<timestamp>` every 30 seconds. Cheap, stateless, works behind any load balancer, no special infrastructure. I don't want to maintain a WebSocket connection per user — that's connection state I have to manage, persist across deploys, and route correctly. Polling scales horizontally without thinking.

**📱 CLIENT SAYS**

> Polling every 30 seconds means a message sits unread for up to 30 seconds before the user sees it. I want live. And every poll request — even when there's nothing new — wastes bandwidth and keeps the radio awake on mobile (battery drain). If you're worried about connections, use SSE — it's one-directional and way simpler than WebSocket. You push when you have something; otherwise silence.

**⚡ THE REAL TENSION**

Real-time UX requires a push model. Push models require server-side connection management. The right tool depends on whether the communication is one-way or two-way.

---

## THE THREE OPTIONS

| Mechanism | Direction | Infrastructure | Best for |
|-----------|-----------|---------------|----------|
| **Polling** | Pull | None | Low-frequency updates (> 30s OK), admin dashboards |
| **SSE (Server-Sent Events)** | Server → Client | HTTP/1.1 | Notifications, live feeds, progress updates |
| **WebSocket** | Bidirectional | WS server | Chat, collaborative editing, live gaming |

**Default decision:** Use SSE unless the client also needs to push data continuously (chat, live collaboration). WebSocket for bidirectional; SSE for everything else.

---

## SSE PATTERN (Django)

```python
# views.py — SSE endpoint
import json
import time
from django.http import StreamingHttpResponse

def event_stream(user_id: str):
    """Generator that yields SSE-formatted events."""
    last_id = 0
    while True:
        events = Notification.objects.filter(
            user_id=user_id,
            id__gt=last_id,
            seen=False,
        ).order_by("id")[:10]

        for event in events:
            last_id = event.id
            yield f"id: {event.id}\n"
            yield f"event: notification\n"
            yield f"data: {json.dumps(event.to_dict())}\n\n"

        if not events:
            yield ": heartbeat\n\n"  # keep connection alive, prevents proxy timeout

        time.sleep(2)  # check every 2 seconds

@router.get("/events/stream", auth=JWTAuth())
def stream_events(request):
    response = StreamingHttpResponse(
        event_stream(str(request.user.id)),
        content_type="text/event-stream",
    )
    response["Cache-Control"] = "no-cache"
    response["X-Accel-Buffering"] = "no"  # disable Nginx buffering
    return response
```

```typescript
// Client — EventSource API
const es = new EventSource("/events/stream", { withCredentials: true });

es.addEventListener("notification", (event) => {
  const notification = JSON.parse(event.data);
  addToNotificationBell(notification);
});

es.onerror = () => {
  // Browser auto-reconnects after error — nothing to do here
  console.log("SSE reconnecting...");
};

// Cleanup on unmount
return () => es.close();
```

**SSE auto-reconnects.** The browser natively handles reconnection. The `Last-Event-ID` header is sent on reconnect so the server can resume from where it left off.

---

## WEBSOCKET PATTERN (Django Channels)

Use only for bidirectional real-time (chat, collaborative editing).

```python
# consumers.py (Django Channels)
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class ChatConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.room_id = self.scope["url_route"]["kwargs"]["room_id"]
        self.group_name = f"chat_{self.room_id}"

        # Verify auth
        user = self.scope["user"]
        if not user.is_authenticated:
            await self.close(code=4001)
            return

        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def receive_json(self, content):
        # Client sent a message
        message = await save_message(self.scope["user"], self.room_id, content["text"])
        await self.channel_layer.group_send(
            self.group_name,
            {"type": "chat.message", "message": message.to_dict()},
        )

    async def chat_message(self, event):
        # Broadcast to this connection
        await self.send_json(event["message"])
```

---

## POLLING PATTERN (when real-time is not required)

If updates older than 30 seconds are acceptable, long polling is the simplest option:

```python
@router.get("/notifications")
def get_notifications(request, since: str | None = None):
    since_dt = datetime.fromisoformat(since) if since else datetime.utcnow() - timedelta(hours=24)
    notifications = Notification.objects.filter(
        user=request.user,
        created_at__gt=since_dt,
    ).order_by("-created_at")[:50]

    return {
        "items": [n.to_dict() for n in notifications],
        "server_time": datetime.utcnow().isoformat(),  # client uses this as next `since`
    }
```

```typescript
// Client polling with exponential backoff when idle
function startPolling() {
  let interval = 10_000;  // start at 10s
  const MAX_INTERVAL = 60_000;

  async function poll() {
    const data = await fetchNotifications(lastServerTime);
    if (data.items.length > 0) {
      handleNotifications(data.items);
      interval = 10_000;  // reset to fast polling when active
    } else {
      interval = Math.min(interval * 1.5, MAX_INTERVAL);  // slow down when idle
    }
    lastServerTime = data.server_time;
    setTimeout(poll, interval);
  }

  poll();
}
```

---

## DECISION RULES

- **Chat / collaborative editing** → WebSocket.
- **Notifications, live feed, progress bars, order status** → SSE.
- **Admin dashboards, infrequent status checks** → polling with backoff.
- **Never use polling at < 5s intervals.** That's just a slow SSE with more overhead.
- **Never use WebSocket for one-directional server push.** SSE is simpler, works over HTTP/1.1, passes through proxies more easily.
