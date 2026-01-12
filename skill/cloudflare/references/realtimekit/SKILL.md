# Cloudflare RealtimeKit Development Skill

Expert guidance for building real-time video and audio applications using **Cloudflare RealtimeKit** - a comprehensive SDK suite for adding customizable live video and voice to web or mobile applications.

---

## Product Overview

**RealtimeKit** is Cloudflare's high-level SDK suite built on top of the Realtime SFU (Selective Forwarding Unit). It abstracts WebRTC complexities while providing:

- **Fast integration**: Add video/voice in minutes with minimal code
- **Full customization**: Pre-built UI components or build from scratch
- **Global performance**: Runs on Cloudflare's network in 300+ cities
- **Production-ready features**: Recording, transcription, chat, polls, breakout rooms, virtual backgrounds

### Product Suite Hierarchy

```
RealtimeKit (High-level SDK)
    ├── UI Kit (Pre-built UI components)
    ├── Core SDK (Programmatic APIs)
    └── Realtime SFU (Media server - underlying infrastructure)
```

### Key Use Cases

- Team meetings & virtual classrooms
- Webinars & livestreaming
- Social video chat
- Audio-only calls
- Interactive experiences with plugins

---

## Architecture & Core Concepts

### App
Workspace/environment container that groups meetings, participants, presets, recordings. Create separate Apps for staging/production.

### Meeting
Re-usable virtual room. Each join creates a new **Session**. Only one active session per meeting at a time.

### Session
Live instance of a meeting. Created when first participant joins, ends after last participant leaves. Each session has independent participants, chat, recordings.

### Participant
User added to meeting via REST API. API returns `authToken` for client SDK authentication. **Do not reuse tokens.**

### Preset
Reusable permission & UI configuration template. Defines:
- User permissions (can share audio/video, can record, can livestream, etc.)
- Meeting type (group call, webinar, audio-only)
- UI customization (colors, theme, layout)
- Applied at participant creation, can differ per user in same meeting

**Example**: Teacher (webinar-host preset) + Students (webinar-participant preset) in same meeting

### Peer ID vs Participant ID
- **Peer ID** (`id`): Unique per session connection, changes on rejoin
- **Participant ID** (`userId`): Persistent across sessions
- Use `userId` for cross-session tracking, `id` for current session

---

## Development Workflow

### 1. Setup & Configuration

#### Create App & Get Credentials
```bash
# Via Dashboard
https://dash.cloudflare.com/?to=/:account/realtime/kit

# Via API
curl -X POST 'https://api.cloudflare.com/client/v4/accounts/<account_id>/realtime/kit/apps' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <api_token>' \
  -d '{"name": "My RealtimeKit App"}'
```

**Required Permissions**: Create API token with **Realtime / Realtime Admin** permissions

#### Create Presets (Optional - Dashboard creates defaults)
```bash
curl -X POST 'https://api.cloudflare.com/client/v4/accounts/<account_id>/realtime/kit/<app_id>/presets' \
  -H 'Authorization: Bearer <api_token>' \
  -d '{
    "name": "host",
    "permissions": {
      "canShareAudio": true,
      "canShareVideo": true,
      "canRecord": true
    }
  }'
```

### 2. Create Meeting
```bash
curl -X POST 'https://api.cloudflare.com/client/v4/accounts/<account_id>/realtime/kit/<app_id>/meetings' \
  -H 'Authorization: Bearer <api_token>' \
  -d '{"title": "Team Standup"}'
```

**Response**: Returns `meeting_id` for adding participants

### 3. Add Participants
```bash
curl -X POST 'https://api.cloudflare.com/client/v4/accounts/<account_id>/realtime/kit/<app_id>/meetings/<meeting_id>/participants' \
  -H 'Authorization: Bearer <api_token>' \
  -d '{
    "name": "Alice",
    "preset_name": "host",
    "custom_participant_id": "<your_user_id>"
  }'
```

**Response**: Returns `authToken` - pass to client SDK for authentication

---

## Client SDK Integration

### Installation

**React**:
```bash
npm install @cloudflare/realtimekit @cloudflare/realtimekit-react-ui
```

**Angular**:
```bash
npm install @cloudflare/realtimekit @cloudflare/realtimekit-angular-ui
```

**Web Components/HTML**:
```bash
npm install @cloudflare/realtimekit @cloudflare/realtimekit-ui
```

### Quick Start - Default UI

**React**:
```tsx
import { RtkMeeting } from '@cloudflare/realtimekit-react-ui';

function App() {
  return (
    <RtkMeeting
      authToken="<participant_auth_token>"
      onLeave={() => console.log('User left meeting')}
    />
  );
}
```

**Angular**:
```typescript
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <rtk-meeting
      [authToken]="authToken"
      (rtkLeave)="onLeave($event)">
    </rtk-meeting>
  `
})
export class AppComponent {
  authToken = '<participant_auth_token>';
  
  onLeave(event: any) {
    console.log('User left meeting');
  }
}
```

**HTML/Web Components**:
```html
<!DOCTYPE html>
<html>
<head>
  <script type="module" src="https://cdn.jsdelivr.net/npm/@cloudflare/realtimekit-ui/dist/realtimekit-ui/realtimekit-ui.esm.js"></script>
</head>
<body>
  <rtk-meeting id="meeting"></rtk-meeting>
  
  <script>
    const meeting = document.getElementById('meeting');
    meeting.authToken = '<participant_auth_token>';
    meeting.addEventListener('rtkLeave', () => {
      console.log('User left meeting');
    });
  </script>
</body>
</html>
```

### Core SDK - Programmatic Control

**Initialize SDK**:
```typescript
import RealtimeKitClient from '@cloudflare/realtimekit';

const meeting = new RealtimeKitClient({
  authToken: '<participant_auth_token>',
  video: true,    // Enable video by default
  audio: true,    // Enable audio by default
  mediaConfiguration: {
    video: {
      width: { ideal: 1280 },
      height: { ideal: 720 },
      frameRate: { ideal: 30 }
    },
    audio: {
      echoCancellation: true,
      noiseSuppression: true,
      autoGainControl: true
    }
  },
  autoSwitchAudioDevice: true  // Auto-switch to new audio devices
});

// Join the session
await meeting.join();
```

**Advanced Configuration Options**:
```typescript
interface MediaConfiguration {
  video?: {
    width: { ideal: number };
    height: { ideal: number };
    frameRate?: { ideal: number };
  };
  audio?: {
    echoCancellation?: boolean;
    noiseSuppression?: boolean;
    autoGainControl?: boolean;
    enableStereo?: boolean;
    enableHighBitrate?: boolean;
  };
  screenshare?: {
    width?: { max: number };
    height?: { max: number };
    frameRate?: { ideal: number; max: number };
    displaySurface?: 'window' | 'monitor' | 'browser';
    selfBrowserSurface?: 'include' | 'exclude';
  };
}

interface RecordingConfig {
  fileNamePrefix?: string;
  videoConfig?: {
    height?: number;
    width?: number;
    codec?: string;
  };
}
```

---

## Meeting Object API

The `meeting` object returned by SDK initialization provides access to all functionality:

### `meeting.self` - Local Participant Control

**Properties**:
```typescript
meeting.self.id                 // Peer ID (unique per session)
meeting.self.userId             // Participant ID (persistent)
meeting.self.name               // Display name
meeting.self.audioEnabled       // Boolean: audio state
meeting.self.videoEnabled       // Boolean: video state
meeting.self.screenShareEnabled // Boolean: screen share state
meeting.self.audioTrack         // MediaStreamTrack | null
meeting.self.videoTrack         // MediaStreamTrack | null
meeting.self.screenShareTracks  // { audio: MediaStreamTrack, video: MediaStreamTrack } | null
meeting.self.roomJoined         // Boolean: joined state
meeting.self.roomState          // Current room state
```

**Methods**:
```typescript
// Media controls
await meeting.self.enableAudio()
await meeting.self.disableAudio()
await meeting.self.enableVideo()
await meeting.self.disableVideo()
await meeting.self.enableScreenShare()
await meeting.self.disableScreenShare()

// Device management
await meeting.self.setName("New Name")  // Before join only
const devices = await meeting.self.getAllDevices()
const audioDevices = await meeting.self.getAudioDevices()
const videoDevices = await meeting.self.getVideoDevices()
const speakerDevices = await meeting.self.getSpeakerDevices()
const current = await meeting.self.getCurrentDevices()
await meeting.self.setDevice(device)
```

**Events**:
```typescript
meeting.self.on('roomJoined', () => {
  console.log('Successfully joined');
});

meeting.self.on('audioUpdate', ({ audioEnabled, audioTrack }) => {
  console.log('Audio state:', audioEnabled);
});

meeting.self.on('videoUpdate', ({ videoEnabled, videoTrack }) => {
  console.log('Video state:', videoEnabled);
});

meeting.self.on('screenShareUpdate', ({ screenShareEnabled, screenShareTracks }) => {
  console.log('Screen share state:', screenShareEnabled);
});

meeting.self.on('deviceUpdate', ({ device }) => {
  console.log('Device changed:', device);
});

meeting.self.on('deviceListUpdate', ({ added, removed, devices }) => {
  console.log('Device list updated:', devices);
});
```

### `meeting.participants` - Remote Participants

**Participant Maps**:
```typescript
meeting.participants.joined      // Map of all joined participants
meeting.participants.active      // Map of participants with active media
meeting.participants.waitlisted  // Map of waitlisted participants
meeting.participants.pinned      // Map of pinned participants

// Convert to arrays
const joined = meeting.participants.joined.toArray()
const count = meeting.participants.joined.size()

// Access by ID
const participant = meeting.participants.joined.get('peer-id')
```

**Participant Properties**:
```typescript
participant.id                 // Peer ID
participant.userId             // Participant ID
participant.name               // Display name
participant.audioEnabled       // Boolean
participant.videoEnabled       // Boolean
participant.screenShareEnabled // Boolean
participant.audioTrack         // MediaStreamTrack | null
participant.videoTrack         // MediaStreamTrack | null
participant.screenShareTracks  // { audio, video } | null
```

**Events**:
```typescript
meeting.participants.joined.on('participantJoined', (participant) => {
  console.log(`${participant.name} joined`);
});

meeting.participants.joined.on('participantLeft', (participant) => {
  console.log(`${participant.name} left`);
});
```

### `meeting.meta` - Meeting Metadata

```typescript
meeting.meta.meetingId                // Meeting identifier
meeting.meta.meetingTitle             // Meeting title
meeting.meta.meetingStartedTimestamp  // Start time
```

### `meeting.chat` - Chat Messaging

```typescript
// Properties
meeting.chat.messages  // Array of all messages

// Methods
await meeting.chat.sendTextMessage("Hello everyone!")
await meeting.chat.sendImageMessage(imageFile)

// Events
meeting.chat.on('chatUpdate', ({ message, messages }) => {
  console.log(`New message: ${message}`);
  console.log(`All messages:`, messages);
});
```

### `meeting.polls` - Polling

```typescript
// Properties
meeting.polls.items  // Array of all polls

// Methods
await meeting.polls.create(
  "What time works?",           // question
  ["9 AM", "2 PM", "5 PM"],    // options
  false,                        // anonymous
  false                         // hideVotes
)
await meeting.polls.vote(pollId, optionIndex)
```

### `meeting.plugins` - Collaborative Apps

```typescript
// Properties
meeting.plugins.all  // Array of available plugins

// Methods
await meeting.plugins.activate(pluginId)
await meeting.plugins.deactivate()
```

### `meeting.ai` - AI Features

```typescript
// Access live transcriptions (when enabled in Preset)
meeting.ai.transcripts
```

### Core Meeting Methods

```typescript
await meeting.join()   // Join session (emits 'roomJoined' on meeting.self)
await meeting.leave()  // Leave session
```

---

## Session Lifecycle & States

### Peer States Flow

```
Initialization (Setup Screen)
    ↓
Join Intent
    ↓
    ├─→ Waitlisted? → Waitlist Screen
    │       ↓
    │       ├─→ Approved → Meeting Screen (Stage)
    │       └─→ Rejected → Rejected Screen → Ended
    │
    └─→ Direct Join → Meeting Screen (Stage)
            ↓
            ├─→ Left Voluntarily → Ended Screen
            ├─→ Kicked → Ended Screen
            └─→ Meeting Ended → Ended Screen
```

**UI Kit automatically handles these transitions** - no manual state management needed.

---

## REST API Operations

### Meetings

**Fetch All**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/meetings
```

**Get Meeting**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}
```

**Update Meeting**:
```bash
PATCH /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}
Body: {"title": "New Title", "record_on_start": true}
```

### Participants

**List Participants**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/participants
```

**Get Participant**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/participants/{participant_id}
```

**Update Participant**:
```bash
PATCH /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/participants/{participant_id}
Body: {"name": "New Name", "preset_name": "viewer"}
```

**Delete Participant**:
```bash
DELETE /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/participants/{participant_id}
```

**Refresh Auth Token**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/participants/{participant_id}/token
```

### Active Session Management

**Fetch Active Session**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/active-session
```

**Kick Participants**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/active-session/kick
Body: {"user_ids": ["user1", "user2"]}  # or custom_participant_ids
```

**Kick All**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/active-session/kick-all
```

**Create Poll**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/active-session/poll
Body: {
  "question": "What time?",
  "options": ["9 AM", "2 PM"],
  "anonymous": false
}
```

### Recording

**Start Recording**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/recordings
Body: {
  "meeting_id": "<meeting_id>",
  "type": "composite"  # or "track"
}
```

**List Recordings**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/recordings?meeting_id=<meeting_id>
```

**Get Active Recording**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/recordings/active-recording/{meeting_id}
```

**Control Recording (Pause/Resume/Stop)**:
```bash
PUT /accounts/{account_id}/realtime/kit/{app_id}/recordings/{recording_id}
Body: {"action": "pause"}  # or "resume", "stop"
```

**Track Recording** (Advanced):
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/recordings/track
Body: {
  "meeting_id": "<meeting_id>",
  "layers": [
    {"peer_id": "<peer_id>", "track_type": "video", "destination": "file1.mp4"}
  ]
}
```

### Livestreaming

**Start Livestream**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/livestreams
```

**Stop Livestream**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/active-livestream/stop
```

**List Livestreams**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/livestreams?exclude_meetings=false
```

**Get Livestream Details**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/livestreams/{livestream_id}
```

**Create Independent Livestream**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/livestreams
Returns: {ingest_server, stream_key, playback_url}
```

### Sessions & Analytics

**List Sessions**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions
```

**Get Session Details**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}
```

**Get Session Participants**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}/participants
```

**Get Participant Call Stats**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}/participants/{participant_id}
```

**Download Chat Messages (CSV)**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}/chat
```

**Download Transcript (CSV)**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}/transcript
```

**Get Transcript Summary**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}/summary
```

**Generate Summary**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/sessions/{session_id}/summary
```

**Analytics (Day-wise)**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/analytics/daywise?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD
```

**Livestream Analytics**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/analytics/livestreams/overall
```

### Webhooks

**List Webhooks**:
```bash
GET /accounts/{account_id}/realtime/kit/{app_id}/webhooks
```

**Create Webhook**:
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/webhooks
Body: {
  "url": "https://example.com/webhook",
  "events": ["session.started", "session.ended"]
}
```

**Update Webhook**:
```bash
PATCH /accounts/{account_id}/realtime/kit/{app_id}/webhooks/{webhook_id}
```

**Delete Webhook**:
```bash
DELETE /accounts/{account_id}/realtime/kit/{app_id}/webhooks/{webhook_id}
```

---

## Common Code Patterns

### Pattern: Basic Meeting Setup

```typescript
import RealtimeKitClient from '@cloudflare/realtimekit';

async function setupMeeting(authToken: string) {
  const meeting = new RealtimeKitClient({
    authToken,
    video: true,
    audio: true
  });

  // Listen for join event
  meeting.self.on('roomJoined', () => {
    console.log('Joined meeting:', meeting.meta.meetingTitle);
  });

  // Listen for new participants
  meeting.participants.joined.on('participantJoined', (participant) => {
    console.log(`${participant.name} joined`);
  });

  // Join the session
  await meeting.join();
  
  return meeting;
}
```

### Pattern: Participant Video Grid

```typescript
import { useEffect, useState } from 'react';

function VideoGrid({ meeting }) {
  const [participants, setParticipants] = useState([]);

  useEffect(() => {
    const updateParticipants = () => {
      setParticipants(meeting.participants.joined.toArray());
    };

    meeting.participants.joined.on('participantJoined', updateParticipants);
    meeting.participants.joined.on('participantLeft', updateParticipants);
    
    updateParticipants();

    return () => {
      meeting.participants.joined.off('participantJoined', updateParticipants);
      meeting.participants.joined.off('participantLeft', updateParticipants);
    };
  }, [meeting]);

  return (
    <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(300px, 1fr))' }}>
      {participants.map(participant => (
        <VideoTile key={participant.id} participant={participant} />
      ))}
    </div>
  );
}

function VideoTile({ participant }) {
  const videoRef = useRef(null);

  useEffect(() => {
    if (videoRef.current && participant.videoTrack) {
      const stream = new MediaStream([participant.videoTrack]);
      videoRef.current.srcObject = stream;
    }
  }, [participant.videoTrack]);

  return (
    <div>
      <video ref={videoRef} autoPlay playsInline muted />
      <div>{participant.name}</div>
    </div>
  );
}
```

### Pattern: Device Selection

```typescript
async function setupDeviceSelection(meeting: RealtimeKitClient) {
  // Get available devices
  const devices = await meeting.self.getAllDevices();
  
  const audioInputs = devices.filter(d => d.kind === 'audioinput');
  const videoInputs = devices.filter(d => d.kind === 'videoinput');
  const audioOutputs = devices.filter(d => d.kind === 'audiooutput');

  // Listen for device changes
  meeting.self.on('deviceListUpdate', ({ added, removed, devices }) => {
    console.log('Device list updated');
    console.log('Added:', added);
    console.log('Removed:', removed);
  });

  // Switch device
  async function switchCamera(deviceId: string) {
    const device = devices.find(d => d.deviceId === deviceId);
    if (device) {
      await meeting.self.setDevice(device);
    }
  }

  return { audioInputs, videoInputs, audioOutputs, switchCamera };
}
```

### Pattern: Chat Integration

```typescript
function ChatComponent({ meeting }) {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');

  useEffect(() => {
    setMessages(meeting.chat.messages);

    const handleChatUpdate = ({ messages }) => {
      setMessages(messages);
    };

    meeting.chat.on('chatUpdate', handleChatUpdate);

    return () => {
      meeting.chat.off('chatUpdate', handleChatUpdate);
    };
  }, [meeting]);

  const sendMessage = async () => {
    if (input.trim()) {
      await meeting.chat.sendTextMessage(input);
      setInput('');
    }
  };

  return (
    <div>
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i}>
            <strong>{msg.senderName}:</strong> {msg.text}
          </div>
        ))}
      </div>
      <input
        value={input}
        onChange={e => setInput(e.target.value)}
        onKeyPress={e => e.key === 'Enter' && sendMessage()}
      />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
}
```

### Pattern: React Hook for Meeting

```typescript
import { useEffect, useState } from 'react';
import RealtimeKitClient from '@cloudflare/realtimekit';

export function useMeeting(authToken: string) {
  const [meeting, setMeeting] = useState<RealtimeKitClient | null>(null);
  const [joined, setJoined] = useState(false);
  const [participants, setParticipants] = useState([]);

  useEffect(() => {
    const client = new RealtimeKitClient({ authToken });

    client.self.on('roomJoined', () => {
      setJoined(true);
    });

    const updateParticipants = () => {
      setParticipants(client.participants.joined.toArray());
    };

    client.participants.joined.on('participantJoined', updateParticipants);
    client.participants.joined.on('participantLeft', updateParticipants);

    setMeeting(client);

    return () => {
      client.leave();
    };
  }, [authToken]);

  const join = async () => {
    if (meeting) await meeting.join();
  };

  const leave = async () => {
    if (meeting) await meeting.leave();
  };

  return { meeting, joined, participants, join, leave };
}
```

### Pattern: Backend Token Generation

```typescript
// Express.js example
import express from 'express';

const app = express();
const CLOUDFLARE_API_TOKEN = process.env.CLOUDFLARE_API_TOKEN;
const ACCOUNT_ID = process.env.CLOUDFLARE_ACCOUNT_ID;
const APP_ID = process.env.REALTIMEKIT_APP_ID;

app.post('/api/join-meeting', async (req, res) => {
  const { meetingId, userName, presetName } = req.body;

  // Add participant to meeting
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/realtime/kit/${APP_ID}/meetings/${meetingId}/participants`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${CLOUDFLARE_API_TOKEN}`
      },
      body: JSON.stringify({
        name: userName,
        preset_name: presetName,
        custom_participant_id: req.user.id  // Your user ID
      })
    }
  );

  const data = await response.json();
  
  // Return authToken to client
  res.json({ authToken: data.result.authToken });
});
```

### Pattern: Wrangler/Workers Integration

```typescript
// worker.ts
export interface Env {
  CLOUDFLARE_API_TOKEN: string;
  CLOUDFLARE_ACCOUNT_ID: string;
  REALTIMEKIT_APP_ID: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/api/create-meeting') {
      const response = await fetch(
        `https://api.cloudflare.com/client/v4/accounts/${env.CLOUDFLARE_ACCOUNT_ID}/realtime/kit/${env.REALTIMEKIT_APP_ID}/meetings`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${env.CLOUDFLARE_API_TOKEN}`
          },
          body: JSON.stringify({
            title: 'Team Meeting'
          })
        }
      );

      return response;
    }

    return new Response('Not found', { status: 404 });
  }
};
```

**wrangler.toml**:
```toml
name = "realtimekit-app"
main = "src/worker.ts"
compatibility_date = "2024-01-01"

[vars]
CLOUDFLARE_ACCOUNT_ID = "your_account_id"
REALTIMEKIT_APP_ID = "your_app_id"

# Add secret: wrangler secret put CLOUDFLARE_API_TOKEN
```

### Pattern: TURN Service Integration

RealtimeKit can use Cloudflare's TURN service for better connectivity through restrictive networks:

```typescript
// When creating participant with TURN credentials
const meeting = new RealtimeKitClient({
  authToken,
  // TURN automatically configured if enabled in account
});
```

**Environment Variable Setup**:
```bash
# wrangler.toml
[vars]
TURN_SERVICE_ID = "your_turn_service_id"

# Set secret
wrangler secret put TURN_SERVICE_TOKEN
```

---

## Wrangler Configuration Examples

### Basic Configuration

```toml
name = "realtimekit-app"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
CLOUDFLARE_ACCOUNT_ID = "abc123"
REALTIMEKIT_APP_ID = "xyz789"

# Secrets (set with: wrangler secret put SECRET_NAME)
# - CLOUDFLARE_API_TOKEN
```

### With Database & Storage

```toml
name = "realtimekit-app"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
CLOUDFLARE_ACCOUNT_ID = "abc123"
REALTIMEKIT_APP_ID = "xyz789"

# D1 Database for storing meeting metadata
[[d1_databases]]
binding = "DB"
database_name = "realtimekit-meetings"
database_id = "d1-id-here"

# R2 for storing recordings
[[r2_buckets]]
binding = "RECORDINGS"
bucket_name = "meeting-recordings"

# KV for session management
[[kv_namespaces]]
binding = "SESSIONS"
id = "kv-id-here"
```

### Multi-Environment Setup

```toml
# wrangler.toml (base config)
name = "realtimekit-app"
main = "src/index.ts"
compatibility_date = "2024-01-01"

# wrangler.staging.toml
[env.staging]
vars = { 
  CLOUDFLARE_ACCOUNT_ID = "staging-account-id",
  REALTIMEKIT_APP_ID = "staging-app-id"
}

# wrangler.production.toml
[env.production]
vars = { 
  CLOUDFLARE_ACCOUNT_ID = "prod-account-id",
  REALTIMEKIT_APP_ID = "prod-app-id"
}
```

**Deploy**:
```bash
wrangler deploy --env staging
wrangler deploy --env production
```

---

## Best Practices

### Security

1. **Never expose API tokens client-side**
   - Always generate participant tokens server-side
   - Use API tokens only in backend/Workers

2. **Don't reuse participant tokens**
   - Generate fresh token per session
   - Use refresh endpoint if token expires

3. **Use custom participant IDs**
   - Map to your user system
   - Enables tracking across sessions

### Performance

1. **Event-driven updates**
   - Listen to events, don't poll
   - Use `toArray()` only when needed

2. **Media quality constraints**
   - Set appropriate resolution/bitrate limits
   - Consider network conditions

3. **Device management**
   - Enable `autoSwitchAudioDevice` for better UX
   - Handle device list updates

### Architecture

1. **Separate Apps for environments**
   - staging vs production
   - Prevents data mixing

2. **Preset strategy**
   - Create presets at App level
   - Reuse across meetings

3. **Token management**
   - Backend generates tokens
   - Frontend receives via authenticated endpoint

### Development Workflow

1. **Local development**
   ```bash
   wrangler dev --env development
   ```

2. **Testing**
   - Use staging App
   - Test different presets/permissions

3. **Production deployment**
   ```bash
   wrangler deploy --env production
   ```

---

## Common Issues & Solutions

### Issue: Cannot connect to meeting
- **Check**: Auth token is valid and not expired
- **Check**: API credentials have correct permissions
- **Check**: Network allows WebRTC traffic (not blocked by firewall)
- **Solution**: Enable TURN service for restrictive networks

### Issue: No video/audio tracks
- **Check**: Browser permissions granted
- **Check**: `video: true, audio: true` in initialization
- **Check**: Device is available and not in use by another app
- **Solution**: Use `meeting.self.getAllDevices()` to debug

### Issue: Participant count mismatched
- **Remember**: `meeting.participants` doesn't include `meeting.self`
- **Total count**: `meeting.participants.joined.size() + 1`

### Issue: Events not firing
- **Check**: Event listeners registered before actions occur
- **Check**: Correct event name spelling
- **Check**: Using correct namespace (e.g., `meeting.self` vs `meeting.participants`)

### Issue: CORS errors in API calls
- **Cause**: Making REST API calls from client-side
- **Solution**: All REST API calls must be server-side (Workers, backend)

### Issue: Preset not applying
- **Check**: Preset exists in App
- **Check**: `preset_name` matches exactly (case-sensitive)
- **Check**: Participant created after preset

---

## Reference Links

- **Official Docs**: https://developers.cloudflare.com/realtime/realtimekit/
- **API Reference**: https://developers.cloudflare.com/api/resources/realtime_kit/
- **Examples Repo**: https://github.com/cloudflare/realtimekit-web-examples
- **Orange Meets Demo**: https://github.com/cloudflare/orange (full-featured example)
- **Dashboard**: https://dash.cloudflare.com/?to=/:account/realtime/kit
- **Community Discord**: https://discord.cloudflare.com

---

## Quick Reference

### SDK Initialization
```typescript
import RealtimeKitClient from '@cloudflare/realtimekit';
const meeting = new RealtimeKitClient({ authToken, video: true, audio: true });
await meeting.join();
```

### UI Kit (React)
```tsx
import { RtkMeeting } from '@cloudflare/realtimekit-react-ui';
<RtkMeeting authToken={token} onLeave={() => {}} />
```

### Create Meeting (Backend)
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings
```

### Add Participant (Backend)
```bash
POST /accounts/{account_id}/realtime/kit/{app_id}/meetings/{meeting_id}/participants
Returns: { authToken }
```

### Media Controls
```typescript
await meeting.self.enableAudio()
await meeting.self.enableVideo()
await meeting.self.enableScreenShare()
```

### Access Participants
```typescript
const participants = meeting.participants.joined.toArray()
const count = meeting.participants.joined.size()
```

### Chat
```typescript
await meeting.chat.sendTextMessage("Hello!")
meeting.chat.on('chatUpdate', ({ messages }) => {})
```

---

## Pricing

- **RealtimeKit**: Per-minute pricing ([view details](https://workers.cloudflare.com/pricing#media))
- **Realtime SFU**: $0.05/GB egress, first 1,000 GB/month free
- **TURN Service**: Free when used with Realtime SFU, otherwise $0.05/GB egress

---

*This skill focuses exclusively on Cloudflare RealtimeKit. For the underlying Realtime SFU or other Cloudflare products, consult separate documentation.*
