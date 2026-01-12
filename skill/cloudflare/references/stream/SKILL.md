# Cloudflare Stream Expert

Expert skill for Cloudflare Stream - serverless live and on-demand video streaming platform.

## Core Capabilities

Cloudflare Stream provides video upload, storage, encoding, and delivery with one API, without managing infrastructure. Runs on Cloudflare's global network.

### Key Features
- **On-demand video**: Upload, encode, store, and deliver video files
- **Live streaming**: RTMPS/SRT ingestion with adaptive bitrate streaming
- **Direct creator uploads**: End users upload directly without API keys
- **Signed URLs**: Control access with tokens and access rules
- **Analytics**: Server-side metrics via dashboard or GraphQL API
- **Webhooks**: Notifications for video processing completion/errors
- **Watermarks**: Apply watermarks to videos
- **Captions**: Upload or AI-generate subtitles/captions
- **Downloads**: Enable MP4 downloads for offline viewing

## Video Upload Patterns

### 1. Basic Upload via API (TUS Protocol)

```bash
# Initiate upload
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/{account_id}/stream" \
  -H "Authorization: Bearer <API_TOKEN>" \
  -H "Tus-Resumable: 1.0.0" \
  -H "Upload-Length: <FILE_SIZE_BYTES>" \
  -H "Upload-Metadata: name <BASE64_FILENAME>"

# Response includes Location header with upload URL
# Upload chunks to that URL using TUS protocol
```

### 2. Upload from URL

```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/{account_id}/stream/copy" \
  -H "Authorization: Bearer <API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/video.mp4",
    "meta": {
      "name": "My Video"
    },
    "requireSignedURLs": false,
    "allowedOrigins": ["example.com"]
  }'
```

### 3. Direct Creator Uploads (Recommended for User-Generated Content)

**Backend: Create upload URL**
```typescript
// Workers/Node.js example
async function createUploadURL(accountId: string, apiToken: string) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/direct_upload`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        maxDurationSeconds: 3600, // 1 hour max
        expiry: new Date(Date.now() + 3600000).toISOString(), // 1 hour validity
        requireSignedURLs: true,
        allowedOrigins: ['https://yourdomain.com'],
        meta: {
          creator: 'user-id-123'
        }
      })
    }
  );
  
  const data = await response.json();
  return {
    uploadURL: data.result.uploadURL,
    uid: data.result.uid
  };
}
```

**Frontend: Upload directly**
```typescript
// Client-side upload to Stream (no API key exposed)
async function uploadVideo(file: File, uploadURL: string) {
  const formData = new FormData();
  formData.append('file', file);
  
  const response = await fetch(uploadURL, {
    method: 'POST',
    body: formData,
  });
  
  return response.json();
}
```

### 4. Upload with Metadata

```typescript
interface VideoMetadata {
  name?: string;
  creator?: string;
  // Custom fields - any key-value pairs
  [key: string]: string | undefined;
}

const metadata: VideoMetadata = {
  name: "Product Demo",
  creator: "user-123",
  category: "tutorials",
  visibility: "public"
};

// Metadata is searchable and returned with video info
```

## Video Playback

### 1. Using Stream Player (Recommended)

**Basic embed**
```html
<iframe
  src="https://customer-<CODE>.cloudflarestream.com/<VIDEO_ID>/iframe"
  style="border: none;"
  height="720"
  width="1280"
  allow="accelerometer; gyroscope; autoplay; encrypted-media; picture-in-picture;"
  allowfullscreen="true"
></iframe>
```

**React component**
```tsx
interface StreamPlayerProps {
  videoId: string;
  autoplay?: boolean;
  muted?: boolean;
  preload?: 'auto' | 'metadata' | 'none';
  loop?: boolean;
  controls?: boolean;
  defaultTextTrack?: string; // Language code
}

export function StreamPlayer({
  videoId,
  autoplay = false,
  muted = false,
  preload = 'auto',
  loop = false,
  controls = true,
  defaultTextTrack
}: StreamPlayerProps) {
  const params = new URLSearchParams({
    ...(autoplay && { autoplay: 'true' }),
    ...(muted && { muted: 'true' }),
    ...(preload && { preload }),
    ...(loop && { loop: 'true' }),
    ...(controls === false && { controls: 'false' }),
    ...(defaultTextTrack && { defaultTextTrack })
  });
  
  return (
    <iframe
      src={`https://customer-<CODE>.cloudflarestream.com/${videoId}/iframe?${params}`}
      style={{ border: 'none' }}
      height={720}
      width={1280}
      allow="accelerometer; gyroscope; autoplay; encrypted-media; picture-in-picture"
      allowFullScreen
    />
  );
}
```

### 2. Using Custom Player (HLS/DASH)

```typescript
// Video.js example
import videojs from 'video.js';

const player = videojs('my-video', {
  controls: true,
  sources: [{
    src: `https://customer-<CODE>.cloudflarestream.com/${videoId}/manifest/video.m3u8`,
    type: 'application/x-mpegURL' // HLS
  }]
});

// Or DASH
const dashPlayer = videojs('my-video', {
  controls: true,
  sources: [{
    src: `https://customer-<CODE>.cloudflarestream.com/${videoId}/manifest/video.mpd`,
    type: 'application/dash+xml'
  }]
});
```

**HLS.js example**
```typescript
import Hls from 'hls.js';

if (Hls.isSupported()) {
  const video = document.getElementById('video') as HTMLVideoElement;
  const hls = new Hls();
  hls.loadSource(
    `https://customer-<CODE>.cloudflarestream.com/${videoId}/manifest/video.m3u8`
  );
  hls.attachMedia(video);
}
```

### 3. Thumbnails

```typescript
// Get thumbnail at specific time (percentage)
const thumbnailURL = (videoId: string, time: number = 0) => 
  `https://customer-<CODE>.cloudflarestream.com/${videoId}/thumbnails/thumbnail.jpg?time=${time}s`;

// Or by percentage
const thumbnailByPct = (videoId: string, pct: number = 0) => 
  `https://customer-<CODE>.cloudflarestream.com/${videoId}/thumbnails/thumbnail.jpg?time=${pct}%`;

// Animated GIF
const animatedThumbnail = (videoId: string) =>
  `https://customer-<CODE>.cloudflarestream.com/${videoId}/thumbnails/thumbnail.gif`;
```

## Signed URLs / Access Control

### Pattern 1: Short-lived Token (Low Volume)

**Use when**: Generating < 1,000 tokens/day

```typescript
async function getSignedToken(
  accountId: string,
  videoId: string,
  apiToken: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}/token`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        exp: Math.floor(Date.now() / 1000) + 3600, // 1 hour
        downloadable: true, // Allow MP4 download
        accessRules: [
          {
            type: 'ip.geoip.country',
            action: 'allow',
            country: ['US', 'CA', 'GB']
          },
          {
            type: 'any',
            action: 'block'
          }
        ]
      })
    }
  );
  
  const { result } = await response.json();
  return result.token;
}

// Use token in place of video ID
const videoURL = `https://customer-<CODE>.cloudflarestream.com/${token}/iframe`;
```

### Pattern 2: Self-Signing with Keys (High Volume)

**Use when**: Thousands of daily users or WebRTC live streaming

**Step 1: Create signing key (once)**
```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/{account_id}/stream/keys" \
  -H "Authorization: Bearer <API_TOKEN>"

# Save the `id` and `jwk` (base64-encoded) from response
```

**Step 2: Generate tokens in Workers**
```typescript
// Cloudflare Worker
interface SigningKey {
  id: string;
  jwk: string; // base64-encoded JWK
}

interface TokenPayload {
  sub: string; // video ID
  kid: string; // key ID
  exp: number; // expiration timestamp
  accessRules?: AccessRule[];
  downloadable?: boolean;
}

interface AccessRule {
  type: 'ip.src' | 'ip.geoip.country' | 'any';
  action: 'allow' | 'block';
  country?: string[];
  ip?: string[];
}

async function generateSignedToken(
  videoId: string,
  keyId: string,
  jwkBase64: string,
  expiresInSeconds: number = 3600,
  accessRules?: AccessRule[]
): Promise<string> {
  const encoder = new TextEncoder();
  const expiresAt = Math.floor(Date.now() / 1000) + expiresInSeconds;
  
  const headers = {
    alg: 'RS256',
    kid: keyId
  };
  
  const payload: TokenPayload = {
    sub: videoId,
    kid: keyId,
    exp: expiresAt,
    ...(accessRules && { accessRules }),
  };
  
  const token = `${objectToBase64url(headers)}.${objectToBase64url(payload)}`;
  
  // Decode base64 JWK
  const jwk = JSON.parse(atob(jwkBase64));
  
  const key = await crypto.subtle.importKey(
    'jwk',
    jwk,
    { name: 'RSASSA-PKCS1-v1_5', hash: 'SHA-256' },
    false,
    ['sign']
  );
  
  const signature = await crypto.subtle.sign(
    { name: 'RSASSA-PKCS1-v1_5' },
    key,
    encoder.encode(token)
  );
  
  return `${token}.${arrayBufferToBase64Url(signature)}`;
}

function objectToBase64url(obj: unknown): string {
  return arrayBufferToBase64Url(
    new TextEncoder().encode(JSON.stringify(obj))
  );
}

function arrayBufferToBase64Url(buffer: ArrayBuffer): string {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/=/g, '')
    .replace(/\+/g, '-')
    .replace(/\//g, '_');
}

// Example usage in Worker
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { videoId } = await request.json();
    
    // Restrict to US/CA, valid for 2 hours
    const token = await generateSignedToken(
      videoId,
      env.STREAM_KEY_ID,
      env.STREAM_JWK,
      7200,
      [
        { type: 'ip.geoip.country', action: 'allow', country: ['US', 'CA'] },
        { type: 'any', action: 'block' }
      ]
    );
    
    return Response.json({ token });
  }
};
```

### Access Rule Examples

```typescript
// Allow specific countries, block rest
const geoRestriction: AccessRule[] = [
  { type: 'ip.geoip.country', action: 'allow', country: ['US', 'GB', 'CA'] },
  { type: 'any', action: 'block' }
];

// Block specific countries
const geoBlock: AccessRule[] = [
  { type: 'ip.geoip.country', action: 'block', country: ['CN', 'RU'] },
  { type: 'any', action: 'allow' }
];

// IP allowlist (internal use)
const ipAllowlist: AccessRule[] = [
  { 
    type: 'ip.src',
    action: 'allow',
    ip: ['10.0.0.0/8', '2001:db8::/32']
  },
  { type: 'any', action: 'block' }
];

// Time-limited + geo-restricted + downloadable
const premiumAccess = {
  exp: Math.floor(Date.now() / 1000) + 86400, // 24 hours
  downloadable: true,
  accessRules: [
    { type: 'ip.geoip.country', action: 'allow', country: ['US'] }
  ]
};
```

## Live Streaming

### 1. Create Live Input

```typescript
interface LiveInputConfig {
  meta?: Record<string, string>;
  recording?: {
    mode: 'automatic' | 'off';
    requireSignedURLs?: boolean;
    allowedOrigins?: string[];
    timeoutSeconds?: number;
  };
  deleteRecordingAfterDays?: number;
}

async function createLiveInput(
  accountId: string,
  apiToken: string,
  config: LiveInputConfig
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/live_inputs`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        meta: config.meta || {},
        recording: config.recording || { mode: 'automatic' },
        deleteRecordingAfterDays: config.deleteRecordingAfterDays
      })
    }
  );
  
  const { result } = await response.json();
  
  return {
    uid: result.uid,
    rtmps: {
      url: result.rtmps.url,
      streamKey: result.rtmps.streamKey
    },
    srt: {
      url: result.srt.url,
      streamId: result.srt.streamId,
      passphrase: result.srt.passphrase
    },
    webRTC: result.webRTC
  };
}
```

### 2. Stream to Live Input

**OBS Configuration:**
```
Server: rtmps://live.cloudflare.com:443/live/
Stream Key: <STREAM_KEY from API>
```

**FFmpeg:**
```bash
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 3000k \
  -maxrate 3000k -bufsize 6000k -g 50 \
  -c:a aac -b:a 128k -ar 44100 \
  -f flv rtmps://live.cloudflare.com:443/live/<STREAM_KEY>
```

### 3. Watch Live Stream

```tsx
// Same as on-demand - use live input UID
<iframe
  src={`https://customer-<CODE>.cloudflarestream.com/${liveInputUID}/iframe`}
  style={{ border: 'none' }}
  height={720}
  width={1280}
  allow="accelerometer; gyroscope; autoplay; encrypted-media; picture-in-picture"
  allowFullScreen
/>
```

### 4. Live Output (Simulcast/Restream)

```typescript
// Create output for simulcasting to YouTube, Twitch, etc.
async function createLiveOutput(
  accountId: string,
  liveInputId: string,
  apiToken: string,
  outputUrl: string,
  streamKey: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/live_inputs/${liveInputId}/outputs`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        url: `${outputUrl}/${streamKey}`,
        enabled: true
      })
    }
  );
  
  return response.json();
}

// Example: Simulcast to YouTube
await createLiveOutput(
  accountId,
  liveInputId,
  apiToken,
  'rtmps://a.rtmps.youtube.com/live2',
  'your-youtube-stream-key'
);
```

### 5. Check Live Status

```typescript
async function getLiveStatus(
  accountId: string,
  liveInputId: string,
  apiToken: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/live_inputs/${liveInputId}`,
    {
      headers: { 'Authorization': `Bearer ${apiToken}` }
    }
  );
  
  const { result } = await response.json();
  
  return {
    isLive: result.status?.current?.state === 'connected',
    recording: result.recording,
    lastSeen: result.status?.current?.lastSeen
  };
}
```

## Video Management

### 1. List Videos with Filtering

```typescript
interface ListVideosOptions {
  search?: string; // Search in name, metadata
  creator?: string; // Filter by creator ID
  limit?: number; // Max 1000
  before?: string; // Pagination cursor
  after?: string; // Pagination cursor
  status?: 'ready' | 'inprogress' | 'error';
}

async function listVideos(
  accountId: string,
  apiToken: string,
  options: ListVideosOptions = {}
) {
  const params = new URLSearchParams();
  if (options.search) params.set('search', options.search);
  if (options.creator) params.set('creator', options.creator);
  if (options.limit) params.set('limit', options.limit.toString());
  if (options.before) params.set('before', options.before);
  if (options.after) params.set('after', options.after);
  
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream?${params}`,
    {
      headers: { 'Authorization': `Bearer ${apiToken}` }
    }
  );
  
  return response.json();
}
```

### 2. Update Video Metadata

```typescript
async function updateVideo(
  accountId: string,
  videoId: string,
  apiToken: string,
  updates: {
    meta?: Record<string, string>;
    requireSignedURLs?: boolean;
    allowedOrigins?: string[];
    thumbnailTimestampPct?: number; // 0-1
  }
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(updates)
    }
  );
  
  return response.json();
}
```

### 3. Delete Video

```typescript
async function deleteVideo(
  accountId: string,
  videoId: string,
  apiToken: string
) {
  await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}`,
    {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${apiToken}` }
    }
  );
}
```

### 4. Clip Video

```typescript
// Create clip from existing video
async function createClip(
  accountId: string,
  videoId: string,
  apiToken: string,
  startTime: number, // seconds
  endTime: number    // seconds
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/clip`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        clippedFromVideoUID: videoId,
        startTime,
        endTime,
        requireSignedURLs: false
      })
    }
  );
  
  const { result } = await response.json();
  return result.uid; // New video ID for clip
}
```

## Captions & Subtitles

### 1. Upload Captions

```typescript
async function uploadCaption(
  accountId: string,
  videoId: string,
  apiToken: string,
  language: string, // BCP47 code (e.g., 'en', 'es', 'fr')
  vttContent: string,
  label?: string
) {
  const formData = new FormData();
  formData.append('file', new Blob([vttContent], { type: 'text/vtt' }));
  if (label) formData.append('label', label);
  
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}/captions/${language}`,
    {
      method: 'PUT',
      headers: { 'Authorization': `Bearer ${apiToken}` },
      body: formData
    }
  );
  
  return response.json();
}
```

### 2. AI-Generated Captions

```typescript
async function generateCaptions(
  accountId: string,
  videoId: string,
  apiToken: string,
  language: string = 'en'
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}/captions/${language}/generate`,
    {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiToken}` }
    }
  );
  
  return response.json();
}
```

### 3. List Captions

```typescript
async function listCaptions(
  accountId: string,
  videoId: string,
  apiToken: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}/captions`,
    {
      headers: { 'Authorization': `Bearer ${apiToken}` }
    }
  );
  
  return response.json();
}
```

## Webhooks

### 1. Configure Webhook

```typescript
async function setupWebhook(
  accountId: string,
  apiToken: string,
  notificationUrl: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/webhook`,
    {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ notificationUrl })
    }
  );
  
  const { result } = await response.json();
  return result.secret; // Save this for signature verification
}
```

### 2. Verify Webhook Signature

```typescript
// Cloudflare Worker webhook handler
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }
    
    const signature = request.headers.get('Webhook-Signature');
    if (!signature) {
      return new Response('No signature', { status: 401 });
    }
    
    const body = await request.text();
    
    // Verify signature
    const isValid = await verifyWebhookSignature(
      signature,
      body,
      env.WEBHOOK_SECRET
    );
    
    if (!isValid) {
      return new Response('Invalid signature', { status: 401 });
    }
    
    // Process webhook
    const payload = JSON.parse(body);
    
    if (payload.readyToStream && payload.status.state === 'ready') {
      // Video is ready
      console.log(`Video ${payload.uid} is ready`);
    } else if (payload.status.state === 'error') {
      // Video failed
      console.error(`Video ${payload.uid} failed:`, payload.status.errorReasonCode);
    }
    
    return new Response('OK');
  }
};

async function verifyWebhookSignature(
  signatureHeader: string,
  body: string,
  secret: string
): Promise<boolean> {
  // Parse signature header: "time=1234567890,sig1=abc123..."
  const parts = signatureHeader.split(',');
  const timePart = parts.find(p => p.startsWith('time='));
  const sigPart = parts.find(p => p.startsWith('sig1='));
  
  if (!timePart || !sigPart) return false;
  
  const timestamp = timePart.split('=')[1];
  const receivedSig = sigPart.split('=')[1];
  
  // Check timestamp is recent (within 5 minutes)
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - parseInt(timestamp)) > 300) {
    return false;
  }
  
  // Create signature source: timestamp + "." + body
  const message = `${timestamp}.${body}`;
  
  // Compute expected signature
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  
  const signature = await crypto.subtle.sign(
    'HMAC',
    key,
    encoder.encode(message)
  );
  
  const expectedSig = Array.from(new Uint8Array(signature))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
  
  return expectedSig === receivedSig;
}
```

## Watermarks

### 1. Upload Watermark Profile

```typescript
async function createWatermark(
  accountId: string,
  apiToken: string,
  imageFile: File,
  options: {
    name: string;
    opacity?: number; // 0-1
    padding?: number; // pixels
    position?: 'upperRight' | 'upperLeft' | 'lowerRight' | 'lowerLeft' | 'center';
    scale?: number; // 0-1 (percentage of video width)
  }
) {
  const formData = new FormData();
  formData.append('file', imageFile);
  formData.append('name', options.name);
  if (options.opacity) formData.append('opacity', options.opacity.toString());
  if (options.padding) formData.append('padding', options.padding.toString());
  if (options.position) formData.append('position', options.position);
  if (options.scale) formData.append('scale', options.scale.toString());
  
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/watermarks`,
    {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiToken}` },
      body: formData
    }
  );
  
  const { result } = await response.json();
  return result.uid;
}
```

### 2. Apply Watermark to Video

```typescript
// Set watermark when uploading
const uploadURL = await createDirectUploadURL(accountId, apiToken, {
  watermark: watermarkUID
});

// Or update existing video
await updateVideo(accountId, videoId, apiToken, {
  watermark: { uid: watermarkUID }
});
```

## Analytics

### GraphQL Query Examples

```typescript
interface AnalyticsQuery {
  accountId: string;
  startDate: string; // ISO 8601
  endDate: string;
  filters?: {
    videoId?: string;
    creator?: string;
    country?: string;
  };
}

// Get video views
const videoViewsQuery = `
  query GetVideoViews($accountId: String!, $startDate: String!, $endDate: String!) {
    viewer {
      accounts(filter: { accountTag: $accountId }) {
        videoPlaybackEventsAdaptiveGroups(
          filter: {
            date_geq: $startDate
            date_leq: $endDate
          }
          orderBy: [date_ASC]
        ) {
          count
          sum {
            timeViewedMinutes
          }
          dimensions {
            date
            videoId
          }
        }
      }
    }
  }
`;

async function getAnalytics(query: AnalyticsQuery, apiToken: string) {
  const response = await fetch(
    'https://api.cloudflare.com/client/v4/graphql',
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        query: videoViewsQuery,
        variables: {
          accountId: query.accountId,
          startDate: query.startDate,
          endDate: query.endDate
        }
      })
    }
  );
  
  return response.json();
}
```

## Downloads (MP4)

### 1. Enable Downloads

```typescript
// Enable default quality MP4 download
async function enableDownloads(
  accountId: string,
  videoId: string,
  apiToken: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}/downloads`,
    {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiToken}` }
    }
  );
  
  return response.json();
}

// Download URL (requires signed token if requireSignedURLs is true)
const downloadUrl = `https://customer-<CODE>.cloudflarestream.com/${videoId}/downloads/default.mp4`;
```

## Common Patterns

### Full-Stack Video Upload Flow

```typescript
// Backend API endpoint (Next.js API route example)
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { userId, videoName } = req.body;
  
  // Create direct upload URL
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${process.env.CF_ACCOUNT_ID}/stream/direct_upload`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.CF_API_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        maxDurationSeconds: 3600,
        requireSignedURLs: true,
        meta: {
          creator: userId,
          name: videoName
        }
      })
    }
  );
  
  const data = await response.json();
  
  return res.json({
    uploadURL: data.result.uploadURL,
    uid: data.result.uid
  });
}
```

```tsx
// Frontend component
'use client';

import { useState } from 'react';

export function VideoUploader() {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  
  async function handleUpload(file: File) {
    setUploading(true);
    
    // Get upload URL from backend
    const { uploadURL, uid } = await fetch('/api/upload-url', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ videoName: file.name })
    }).then(r => r.json());
    
    // Upload directly to Stream
    const formData = new FormData();
    formData.append('file', file);
    
    const xhr = new XMLHttpRequest();
    
    xhr.upload.addEventListener('progress', (e) => {
      if (e.lengthComputable) {
        setProgress((e.loaded / e.total) * 100);
      }
    });
    
    xhr.addEventListener('load', () => {
      setUploading(false);
      // Redirect to video page or show success
      window.location.href = `/videos/${uid}`;
    });
    
    xhr.open('POST', uploadURL);
    xhr.send(formData);
  }
  
  return (
    <div>
      <input
        type="file"
        accept="video/*"
        onChange={(e) => {
          const file = e.target.files?.[0];
          if (file) handleUpload(file);
        }}
        disabled={uploading}
      />
      {uploading && <progress value={progress} max={100} />}
    </div>
  );
}
```

### Video Processing State Management

```typescript
interface VideoState {
  uid: string;
  readyToStream: boolean;
  status: {
    state: 'queued' | 'inprogress' | 'ready' | 'error';
    pctComplete?: string;
    errorReasonCode?: string;
    errorReasonText?: string;
  };
}

async function waitForVideoReady(
  accountId: string,
  videoId: string,
  apiToken: string,
  maxAttempts = 60,
  intervalMs = 5000
): Promise<VideoState> {
  for (let i = 0; i < maxAttempts; i++) {
    const response = await fetch(
      `https://api.cloudflare.com/client/v4/accounts/${accountId}/stream/${videoId}`,
      {
        headers: { 'Authorization': `Bearer ${apiToken}` }
      }
    );
    
    const { result } = await response.json();
    
    if (result.readyToStream || result.status.state === 'error') {
      return result;
    }
    
    await new Promise(resolve => setTimeout(resolve, intervalMs));
  }
  
  throw new Error('Video processing timeout');
}
```

## Configuration Files

### Wrangler Configuration (Workers)

```toml
name = "stream-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
CF_ACCOUNT_ID = "your-account-id"

# Store API token and signing keys in secrets
# wrangler secret put CF_API_TOKEN
# wrangler secret put STREAM_KEY_ID
# wrangler secret put STREAM_JWK
# wrangler secret put WEBHOOK_SECRET
```

### Environment Variables

```bash
# Required
CF_ACCOUNT_ID=your-account-id
CF_API_TOKEN=your-api-token

# For signed URLs (high volume)
STREAM_KEY_ID=your-key-id
STREAM_JWK=base64-encoded-jwk

# For webhooks
WEBHOOK_SECRET=your-webhook-secret

# Customer subdomain code (from dashboard)
STREAM_CUSTOMER_CODE=your-customer-code
```

## Error Handling

### Common Error Codes

```typescript
enum StreamErrorCode {
  ERR_NON_VIDEO = 'ERR_NON_VIDEO',
  ERR_DURATION_EXCEED_CONSTRAINT = 'ERR_DURATION_EXCEED_CONSTRAINT',
  ERR_FETCH_ORIGIN_ERROR = 'ERR_FETCH_ORIGIN_ERROR',
  ERR_MALFORMED_VIDEO = 'ERR_MALFORMED_VIDEO',
  ERR_DURATION_TOO_SHORT = 'ERR_DURATION_TOO_SHORT',
  ERR_UNKNOWN = 'ERR_UNKNOWN'
}

function handleStreamError(errorCode: string): string {
  const messages: Record<string, string> = {
    ERR_NON_VIDEO: 'The uploaded file is not a valid video.',
    ERR_DURATION_EXCEED_CONSTRAINT: 'Video duration exceeds maximum allowed.',
    ERR_FETCH_ORIGIN_ERROR: 'Failed to download video from URL.',
    ERR_MALFORMED_VIDEO: 'Video file is corrupted or malformed.',
    ERR_DURATION_TOO_SHORT: 'Video must be at least 0.1 seconds long.',
    ERR_UNKNOWN: 'An unknown error occurred during video processing.'
  };
  
  return messages[errorCode] || 'Video processing failed.';
}
```

## Performance Best Practices

1. **Use Direct Creator Uploads** for user-generated content to avoid proxying video through your servers
2. **Enable requireSignedURLs** for private content to prevent unauthorized access
3. **Use signing keys** for high-volume token generation instead of API endpoint
4. **Set appropriate allowedOrigins** to prevent hotlinking
5. **Use webhooks** instead of polling for video processing status
6. **Cache video metadata** in your database to reduce API calls
7. **Use HLS/DASH with ABR** for best streaming quality and performance
8. **Set maxDurationSeconds** on direct uploads to prevent abuse
9. **Use creator metadata** to enable per-user analytics and filtering
10. **Enable recordings** for live streams with appropriate retention policies

## Type Definitions

```typescript
interface StreamVideo {
  uid: string;
  thumbnail: string;
  thumbnailTimestampPct: number;
  readyToStream: boolean;
  status: {
    state: 'queued' | 'inprogress' | 'ready' | 'error';
    pctComplete?: string;
    errorReasonCode?: string;
    errorReasonText?: string;
  };
  meta: Record<string, string>;
  created: string;
  modified: string;
  size?: number;
  preview: string;
  allowedOrigins: string[];
  requireSignedURLs: boolean;
  uploaded?: string;
  uploadExpiry?: string;
  maxSizeBytes?: number;
  maxDurationSeconds?: number;
  duration?: number;
  input?: {
    width: number;
    height: number;
  };
  playback?: {
    hls: string;
    dash: string;
  };
  watermark?: {
    uid: string;
  };
}

interface LiveInput {
  uid: string;
  rtmps: {
    url: string;
    streamKey: string;
  };
  srt: {
    url: string;
    streamId: string;
    passphrase: string;
  };
  webRTC?: {
    url: string;
  };
  created: string;
  modified: string;
  meta: Record<string, string>;
  status?: {
    current?: {
      state: 'connected' | 'disconnected';
      lastSeen?: string;
    };
  };
  recording?: {
    mode: 'automatic' | 'off';
    requireSignedURLs: boolean;
    allowedOrigins: string[];
    timeoutSeconds: number;
  };
  deleteRecordingAfterDays?: number;
}
```

## Resources

- API Documentation: https://developers.cloudflare.com/api/resources/stream/
- Stream Docs: https://developers.cloudflare.com/stream/
- Dashboard: https://dash.cloudflare.com/?to=/:account/stream
- Pricing: $5/1000 min stored, $1/1000 min delivered
- Supported formats: MP4, MKV, MOV, AVI, FLV, MPEG-2 TS/PS, MXF, LXF, GXF, 3GP, WebM, MPG, QuickTime
- Max file size: 30 GB
- Max frame rate: 60 fps (recommended)
