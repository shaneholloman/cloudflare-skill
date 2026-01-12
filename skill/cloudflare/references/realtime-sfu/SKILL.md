# Cloudflare Realtime SFU Skill

Expert guidance for building real-time audio/video/data applications using Cloudflare Realtime SFU (Selective Forwarding Unit).

## Overview

Cloudflare Realtime SFU is infrastructure for real-time WebRTC applications that runs on Cloudflare's global network in 310+ cities. It's an unopinionated, low-level WebRTC media server that provides:

- **Anycast WebRTC**: Single IP address for all WebRTC protocols (DTLS, ICE, STUN, SRTP, SCTP)
- **No regional constraints**: Clients auto-connect to nearest data center via BGP
- **Pub/sub model**: Sessions and Tracks (no "rooms" abstraction)
- **Full control**: Raw WebRTC access, no SDK lock-in
- **Global scalability**: Automatic fan-out with cascading trees per track

## Core Concepts

### Sessions
A Session is an established PeerConnection to Cloudflare. Created via HTTPS API, it represents a single WebRTC peer connection.

### Tracks
Media or data channels within a Session. Each Track gets a unique ID and can be:
- **Published**: Sent from client to Cloudflare
- **Subscribed**: Received from Cloudflare by client
- **Types**: Audio, Video, or DataChannel

### No "Rooms" Concept
Unlike traditional SFUs, Cloudflare Realtime has no room abstraction. You build your own presence protocol by:
1. Publishing tracks to sessions
2. Sharing track IDs through your backend
3. Subscribing to tracks in other sessions

This enables flexible topologies: 1:1 calls, 1:many broadcast, many:many conferences, breakout rooms, etc.

## Architecture Patterns

### Basic Pattern
```
Client (WebRTC) <---> Cloudflare Edge <---> Your Backend (HTTP)
                           |
                    Cloudflare Backbone
                           |
                    Other Edge Locations <---> Other Clients
```

### Anycast Benefits
- **Last-mile optimization**: Clients connect to nearest datacenter (~95% of users within 50ms)
- **No region selection**: No need to choose SFU location - BGP routes optimally
- **NACK shield**: Packet retransmissions handled at edge, closer to users
- **Distributed state**: Sessions spread across network, consensus formed dynamically

### Scaling Pattern
Cloudflare uses cascading trees per track:
```
Publisher --> Edge A --> Edge B --> Subscriber 1
                    \--> Edge C --> Subscriber 2
                                \-> Subscriber 3
```
Each track has its own fan-out tree, automatically scaling to millions of participants.

## API Reference

### Authentication
Use App ID and App Secret (from Cloudflare Dashboard):
```bash
curl -X POST 'https://rtc.live/v1/apps/${CALLS_APP_ID}/sessions/new' \
  -H "Authorization: Bearer ${CALLS_APP_SECRET}"
```

### Endpoints

#### Create Session
```http
POST /v1/apps/{appId}/sessions/new
```
Response:
```json
{
  "sessionId": "session_abc123",
  "sessionDescription": {
    "sdp": "v=0\r\no=- ...",
    "type": "offer"
  }
}
```

#### Add Track (Publish)
```http
POST /v1/apps/{appId}/sessions/{sessionId}/tracks/new
Content-Type: application/json

{
  "sessionDescription": {
    "sdp": "v=0\r\n...",
    "type": "offer"
  },
  "tracks": [
    {
      "location": "local",
      "trackName": "my-video-track"
    }
  ]
}
```
Response includes SDP answer and track IDs.

#### Add Track (Subscribe)
```http
POST /v1/apps/{appId}/sessions/{sessionId}/tracks/new
Content-Type: application/json

{
  "tracks": [
    {
      "location": "remote",
      "trackName": "remote-track-id",
      "sessionId": "other-session-id"
    }
  ]
}
```
Server generates SDP offer for remote tracks.

#### Renegotiate Session
```http
PUT /v1/apps/{appId}/sessions/{sessionId}/renegotiate
Content-Type: application/json

{
  "sessionDescription": {
    "sdp": "v=0\r\n...",
    "type": "answer"
  }
}
```
Used after server sends new offer (e.g., adding remote tracks).

#### Close Tracks
```http
PUT /v1/apps/{appId}/sessions/{sessionId}/tracks/close
Content-Type: application/json

{
  "tracks": [
    {"trackName": "track-to-close"}
  ]
}
```

#### Get Session Info
```http
GET /v1/apps/{appId}/sessions/{sessionId}
```
Returns session details and active tracks.

## WebRTC Client Patterns

### Standard Flow
```typescript
// 1. Create RTCPeerConnection
const pc = new RTCPeerConnection({
  iceServers: [{ urls: 'stun:stun.cloudflare.com:3478' }]
});

// 2. Add local tracks
const stream = await navigator.mediaDevices.getUserMedia({ 
  video: true, 
  audio: true 
});
stream.getTracks().forEach(track => pc.addTrack(track, stream));

// 3. Create offer
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);

// 4. Send to backend, backend calls Cloudflare API
const response = await fetch('/api/new-session', {
  method: 'POST',
  body: JSON.stringify({ sdp: offer.sdp })
});

// 5. Set remote answer
const { sessionDescription } = await response.json();
await pc.setRemoteDescription(sessionDescription);

// 6. Handle ICE candidates (usually automatic with SDP)
pc.onicecandidate = (event) => {
  // Cloudflare uses trickle ICE, often complete in SDP
};
```

### Publishing Pattern
```typescript
// Publish video track
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);

const res = await fetch(`/api/sessions/${sessionId}/tracks`, {
  method: 'POST',
  body: JSON.stringify({
    sdp: offer.sdp,
    tracks: [{ location: 'local', trackName: 'my-video' }]
  })
});

const { sessionDescription, tracks } = await res.json();
await pc.setRemoteDescription(sessionDescription);

// Save tracks[0].trackName to share with others
const publishedTrackId = tracks[0].trackName;
```

### Subscribing Pattern
```typescript
// Subscribe to remote track
const res = await fetch(`/api/sessions/${sessionId}/tracks`, {
  method: 'POST',
  body: JSON.stringify({
    tracks: [{ 
      location: 'remote', 
      trackName: remoteTrackId,
      sessionId: remoteSessionId 
    }]
  })
});

const { sessionDescription } = await res.json();
await pc.setRemoteDescription(sessionDescription);

const answer = await pc.createAnswer();
await pc.setLocalDescription(answer);

await fetch(`/api/sessions/${sessionId}/renegotiate`, {
  method: 'PUT',
  body: JSON.stringify({ sdp: answer.sdp })
});

// Handle incoming track
pc.ontrack = (event) => {
  const [remoteStream] = event.streams;
  videoElement.srcObject = remoteStream;
};
```

## Common Use Cases

### 1:1 Video Call
```
User A: Creates session → Publishes video/audio
User B: Creates session → Subscribes to A's tracks → Publishes own tracks
User A: Subscribes to B's tracks
```

### Group Conference (N:N)
```
Each user:
1. Creates session
2. Publishes own tracks
3. Backend broadcasts track IDs to all participants
4. Each user subscribes to all other tracks
```

### Broadcast (1:N)
```
Publisher: Creates session → Publishes tracks
Viewers: Each creates session → Each subscribes to publisher's tracks
No fan-out limit (Cloudflare scales automatically)
```

### Breakout Rooms
```
Same PeerConnection throughout!
- User starts in room A, subscribed to tracks A1, A2
- Backend closes tracks A1, A2
- Backend adds tracks B1, B2 (breakout room)
- User moves back: close B1, B2, add A1, A2
No need to recreate PeerConnection
```

## Backend Implementation Patterns

### Express/Node.js Example
```javascript
import express from 'express';

const app = express();
const CALLS_API = 'https://rtc.live/v1';

// Create new session
app.post('/api/new-session', async (req, res) => {
  const response = await fetch(
    `${CALLS_API}/apps/${process.env.CALLS_APP_ID}/sessions/new`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.CALLS_APP_SECRET}`
      }
    }
  );
  const data = await response.json();
  res.json(data);
});

// Publish tracks
app.post('/api/sessions/:sessionId/tracks', async (req, res) => {
  const { sessionId } = req.params;
  const { sdp, tracks } = req.body;
  
  const response = await fetch(
    `${CALLS_API}/apps/${process.env.CALLS_APP_ID}/sessions/${sessionId}/tracks/new`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.CALLS_APP_SECRET}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        sessionDescription: { sdp, type: 'offer' },
        tracks
      })
    }
  );
  const data = await response.json();
  res.json(data);
});
```

### Cloudflare Workers Example
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    if (url.pathname === '/api/new-session') {
      const response = await fetch(
        `https://rtc.live/v1/apps/${env.CALLS_APP_ID}/sessions/new`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${env.CALLS_APP_SECRET}`
          }
        }
      );
      return response;
    }
    
    // Handle other endpoints...
  }
};
```

### Presence Protocol with Durable Objects
```typescript
export class Room {
  private sessions = new Map<string, {
    userId: string;
    tracks: { trackName: string; kind: string }[];
  }>();

  async fetch(request: Request) {
    const { method, url } = request;
    const path = new URL(url).pathname;

    if (path === '/join') {
      const { sessionId, userId } = await request.json();
      
      // Store session
      this.sessions.set(sessionId, { userId, tracks: [] });
      
      // Return existing participants' tracks
      const existingTracks = Array.from(this.sessions.entries())
        .filter(([id]) => id !== sessionId)
        .flatMap(([id, data]) => 
          data.tracks.map(t => ({ ...t, sessionId: id }))
        );
      
      return new Response(JSON.stringify({ existingTracks }));
    }

    if (path === '/publish') {
      const { sessionId, tracks } = await request.json();
      
      const session = this.sessions.get(sessionId);
      if (session) {
        session.tracks.push(...tracks);
        
        // Notify other participants
        // (implement WebSocket broadcast here)
      }
      
      return new Response('OK');
    }
  }
}
```

## Configuration & Deployment

### Wrangler Setup
```toml
# wrangler.toml
name = "my-calls-app"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
CALLS_APP_ID = "your-app-id"
MAX_WEBCAM_BITRATE = "1200000"
MAX_WEBCAM_FRAMERATE = "24"
MAX_WEBCAM_QUALITY_LEVEL = "1080"

# Set secret separately:
# wrangler secret put CALLS_APP_SECRET

[[durable_objects.bindings]]
name = "ROOM"
class_name = "Room"
```

### Deploy
```bash
# Login
wrangler login

# Set secrets
wrangler secret put CALLS_APP_SECRET
# Or programmatically:
echo "your-secret" | wrangler secret put CALLS_APP_SECRET

# Deploy
wrangler deploy
```

### Environment Variables
Required:
- `CALLS_APP_ID`: From Cloudflare Dashboard
- `CALLS_APP_SECRET`: From Cloudflare Dashboard (secret)

Optional:
- `MAX_WEBCAM_BITRATE` (default: 1200000)
- `MAX_WEBCAM_FRAMERATE` (default: 24)
- `MAX_WEBCAM_QUALITY_LEVEL` (default: 1080)
- `TURN_SERVICE_ID`: Optional TURN service integration
- `TURN_SERVICE_TOKEN`: TURN authentication (secret)

## Advanced Features

### TURN Support
Cloudflare provides TURN servers for restrictive networks:
```javascript
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: 'stun:stun.cloudflare.com:3478' },
    {
      urls: [
        'turn:turn.cloudflare.com:3478?transport=udp',
        'turn:turn.cloudflare.com:3478?transport=tcp',
        'turns:turn.cloudflare.com:5349?transport=tcp'
      ],
      username: turnUsername,
      credential: turnCredential
    }
  ]
});
```
Ports available: 3478 (UDP/TCP), 53 (UDP), 80 (TCP), 443 (TLS), 5349 (TLS)

### Bandwidth Management
```typescript
// Client-side bitrate control
const sender = pc.getSenders().find(s => s.track?.kind === 'video');
const params = sender.getParameters();

if (!params.encodings) {
  params.encodings = [{}];
}

params.encodings[0].maxBitrate = 1200000; // 1.2 Mbps
params.encodings[0].maxFramerate = 24;

await sender.setParameters(params);
```

### Quality Monitoring
```typescript
// Monitor connection quality
setInterval(async () => {
  const stats = await pc.getStats();
  
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'video') {
      console.log('Packet loss:', report.packetsLost);
      console.log('Jitter:', report.jitter);
      console.log('Bitrate:', report.bytesReceived);
    }
  });
}, 1000);
```

### Simulcast
```typescript
// Publish multiple quality levels
const pc = new RTCPeerConnection();
const sender = pc.addTransceiver('video', {
  direction: 'sendonly',
  sendEncodings: [
    { rid: 'high', maxBitrate: 1200000 },
    { rid: 'medium', maxBitrate: 600000, scaleResolutionDownBy: 2 },
    { rid: 'low', maxBitrate: 200000, scaleResolutionDownBy: 4 }
  ]
}).sender;

// Cloudflare automatically forwards appropriate layer to subscribers
```

### DataChannel Usage
```typescript
// Create DataChannel for signaling/chat
const dataChannel = pc.createDataChannel('chat', {
  ordered: true,
  maxRetransmits: 3
});

dataChannel.onopen = () => {
  dataChannel.send(JSON.stringify({ type: 'chat', text: 'Hello!' }));
};

dataChannel.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};
```

## Performance Optimization

### Connection Speed
Typical PeerConnection establishment: 100-250ms globally
- Anycast routing to nearest datacenter
- Parallel STUN/ICE/DTLS
- Pre-warmed connections

### Latency Characteristics
- ~50ms for 95% of Internet users
- ~1ms possible in metro areas with good connectivity
- Glass-to-glass latency: typically 200-400ms (includes encoding/decoding)

### Scaling Considerations
- No participant limit per track (Cloudflare handles fan-out)
- Limit based on client bandwidth and CPU
- Each client typically supports 10-50 concurrent tracks
- Use bandwidth management for large conferences

## Security Best Practices

### Secret Management
❌ Never expose App Secret client-side
✅ Keep in backend environment variables
✅ Use Wrangler secrets for Workers

### Track ID Security
- Track IDs should be treated as capabilities
- Implement authorization in your backend:
```typescript
app.post('/api/sessions/:sessionId/tracks', async (req, res) => {
  const userId = req.user.id; // From your auth
  const { tracks } = req.body;
  
  // Verify user can subscribe to these tracks
  for (const track of tracks) {
    if (!await canAccessTrack(userId, track.trackName)) {
      return res.status(403).json({ error: 'Unauthorized' });
    }
  }
  
  // Proceed with Cloudflare API call
});
```

### Session Validation
- Validate session ownership before operations
- Implement session timeouts
- Clean up abandoned sessions

## Debugging & Troubleshooting

### Chrome WebRTC Internals
Navigate to `chrome://webrtc-internals` to see:
- ICE candidate pairs
- DTLS handshake status
- Media flow statistics
- Bandwidth utilization

### Common Issues

#### Slow Connection (~1.8s)
**Symptom**: First STUN response delayed  
**Cause**: Consensus formation across Cloudflare network  
**Solution**: Normal behavior, subsequent connections faster

#### USE-CANDIDATE Delay (Chrome)
**Symptom**: Chrome waits before sending USE-CANDIDATE  
**Cause**: Initial latency interpreted as high-latency network  
**Solution**: Cloudflare detects DTLS ClientHello earlier to compensate

#### No Media Flow
**Checklist**:
1. Verify SDP exchange completed
2. Check `pc.connectionState === 'connected'`
3. Verify tracks added before offer creation
4. Check browser permissions for camera/mic
5. Inspect `chrome://webrtc-internals` for errors

#### Track Not Received
**Checklist**:
1. Verify track was successfully published (check response)
2. Confirm track ID shared correctly with subscriber
3. Check session IDs match in subscribe request
4. Verify `pc.ontrack` handler attached before answer
5. Ensure renegotiation completed after server offer

### Logging
```typescript
// Enable verbose WebRTC logging
pc.addEventListener('icecandidateerror', (event) => {
  console.error('ICE candidate error:', event);
});

pc.addEventListener('connectionstatechange', () => {
  console.log('Connection state:', pc.connectionState);
});

pc.addEventListener('iceconnectionstatechange', () => {
  console.log('ICE state:', pc.iceConnectionState);
});
```

## Pricing & Limits

### Pricing (as of 2024)
- **Free tier**: 1 TB/month egress
- **Paid usage**: $0.05/GB after free tier
- **Inbound traffic**: Free
- **TURN**: Free with Realtime SFU, $0.05/GB standalone

### Rate Limits
- No enforced participant limits
- No enforced track limits
- Limited by client bandwidth/CPU
- Enterprise: Custom limits available

### Best Practices for Cost
- Optimize bitrates per use case
- Use simulcast for large conferences
- Implement adaptive bitrate on poor connections
- Clean up unused sessions/tracks promptly

## Example Applications

### Orange Meets Reference
Full-featured video conferencing app: https://github.com/cloudflare/orange
- Group video calls
- Screen sharing
- Background blur
- Noise cancellation
- Durable Objects for room state
- End-to-end encryption support

### Minimal Examples
Simple implementations: https://github.com/cloudflare/calls-examples
- 1:1 video call
- Broadcast streaming
- Screen sharing
- DataChannel messaging

## Key Differences from Traditional SFUs

| Aspect | Traditional SFU | Cloudflare Realtime |
|--------|----------------|---------------------|
| **Deployment** | Single server/region | 310+ global datacenters |
| **Routing** | Manual region selection | Automatic anycast routing |
| **SDK** | Often required | Pure WebRTC, no SDK |
| **Architecture** | Rooms with participants | Sessions with tracks (pub/sub) |
| **Scaling** | Vertical/horizontal within region | Automatic global fan-out |
| **TURN** | Separate service | Integrated, anycast |
| **State** | Centralized per room | Distributed, consensus-based |

## Integration Patterns

### With Durable Objects
Use DO for:
- Room presence/participant list
- Track registry
- Chat history
- Recording coordination

```typescript
export class Room {
  async handleJoin(userId: string, sessionId: string) {
    // Store participant
    await this.state.storage.put(`session:${sessionId}`, {
      userId,
      joinedAt: Date.now()
    });
    
    // Return existing participants
    const sessions = await this.state.storage.list({ prefix: 'session:' });
    return Array.from(sessions.entries());
  }
}
```

### With R2 for Recording
```typescript
// Save recording to R2
async function saveRecording(roomId: string, blob: Blob) {
  const key = `recordings/${roomId}/${Date.now()}.webm`;
  await env.R2_BUCKET.put(key, blob);
}
```

### With Queues for Analytics
```typescript
// Track events
await env.ANALYTICS_QUEUE.send({
  event: 'track_published',
  roomId,
  userId,
  trackId,
  timestamp: Date.now()
});
```

## Resources

- **Documentation**: https://developers.cloudflare.com/realtime/sfu/
- **Dashboard**: https://dash.cloudflare.com/?to=/:account/realtime/sfu
- **API Reference**: https://developers.cloudflare.com/api/resources/calls/
- **Orange Meets Demo**: https://demo.orange.cloudflare.dev/
- **Orange Meets Source**: https://github.com/cloudflare/orange
- **Simple Examples**: https://github.com/cloudflare/calls-examples
- **STUN Server**: stun.cloudflare.com:3478
- **Blog Post**: https://blog.cloudflare.com/cloudflare-calls-anycast-webrtc/

## Quick Reference

### Essential API Calls
```bash
# Create session
POST /v1/apps/{appId}/sessions/new

# Publish track
POST /v1/apps/{appId}/sessions/{sessionId}/tracks/new
Body: { sessionDescription, tracks: [{ location: 'local', trackName }] }

# Subscribe to track
POST /v1/apps/{appId}/sessions/{sessionId}/tracks/new
Body: { tracks: [{ location: 'remote', trackName, sessionId }] }

# Renegotiate
PUT /v1/apps/{appId}/sessions/{sessionId}/renegotiate
Body: { sessionDescription }

# Close track
PUT /v1/apps/{appId}/sessions/{sessionId}/tracks/close
Body: { tracks: [{ trackName }] }
```

### Typical Flow
1. Client: Create `RTCPeerConnection`
2. Client: Add local media → Create offer
3. Backend: Call `/sessions/new` or `/tracks/new`
4. Client: Set remote answer
5. WebRTC: ICE + DTLS handshake (automatic)
6. Client: Media flowing
7. Repeat steps 2-6 for subscribing (reverse offer/answer)

---

**Remember**: Cloudflare Realtime SFU is unopinionated infrastructure. You control the entire WebRTC flow and presence protocol. This gives maximum flexibility but requires understanding WebRTC fundamentals.
