# Cloudflare Images Skill

**Cloudflare Images** is an end-to-end image management solution providing storage, transformation, optimization, and delivery at scale via Cloudflare's global network.

## Core Capabilities

### Two Primary Use Cases

1. **Stored Images**: Upload images to Cloudflare Images storage and deliver multiple variants
2. **Transformations**: Optimize images stored externally (R2, origin servers, etc.) without storage

## Authentication

All API requests require authentication via API Token or API Key:

```bash
# Using API Token (recommended)
Authorization: Bearer <API_TOKEN>

# Using API Key
X-Auth-Email: <EMAIL>
X-Auth-Key: <API_KEY>
```

Get credentials from: Cloudflare Dashboard > Images > API Keys

## Image Upload Patterns

### 1. Direct Upload (Server-side)

Upload images directly from your server:

```typescript
// Upload from file
const formData = new FormData();
formData.append('file', fileBuffer, 'image.jpg');

const response = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1`,
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiToken}`,
    },
    body: formData,
  }
);

const { result } = await response.json();
// result.id: unique image identifier
// result.variants: array of URLs for each variant
```

```bash
# Via cURL
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/images/v1 \
  -X POST \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'file=@./image.jpg' \
  -F 'requireSignedURLs=false' \
  -F 'metadata={"userId":"12345","category":"profile"}'
```

### 2. Upload from URL

Fetch and upload an image from a remote URL:

```typescript
const response = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1`,
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiToken}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      url: 'https://example.com/image.jpg',
      requireSignedURLs: false,
      metadata: { source: 'import' },
    }),
  }
);
```

### 3. Direct Creator Upload (Client-side)

Accept uploads directly from users without exposing API credentials:

```typescript
// Server: Generate one-time upload URL
async function generateUploadUrl(accountId: string, apiToken: string) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v2/direct_upload`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      },
      body: new URLSearchParams({
        requireSignedURLs: 'false',
        metadata: JSON.stringify({ userId: '12345' }),
        expiry: new Date(Date.now() + 30 * 60000).toISOString(), // 30 min
      }),
    }
  );
  
  const { result } = await response.json();
  return {
    uploadURL: result.uploadURL,
    imageId: result.id,
  };
}

// Client: Upload directly to Cloudflare
async function uploadImage(uploadURL: string, file: File) {
  const formData = new FormData();
  formData.append('file', file);
  
  await fetch(uploadURL, {
    method: 'POST',
    body: formData,
  });
}
```

### 4. Upload with Custom ID

Use your own identifiers instead of Cloudflare-generated IDs:

```bash
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/images/v1 \
  -X POST \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'file=@./image.jpg' \
  -F 'id=users/profile/user-12345.jpg'
```

**Note**: Images with custom IDs cannot use signed URLs (`requireSignedURLs` must be `false`)

## Image Management API

### List Images

```typescript
async function listImages(
  accountId: string,
  apiToken: string,
  page = 1,
  perPage = 100
) {
  const url = new URL(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1`
  );
  url.searchParams.set('page', String(page));
  url.searchParams.set('per_page', String(perPage));
  
  const response = await fetch(url.toString(), {
    headers: { 'Authorization': `Bearer ${apiToken}` },
  });
  
  const { result } = await response.json();
  return result.images; // Array of image objects
}
```

### Get Image Details

```typescript
async function getImage(accountId: string, apiToken: string, imageId: string) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1/${imageId}`,
    {
      headers: { 'Authorization': `Bearer ${apiToken}` },
    }
  );
  
  const { result } = await response.json();
  // result.id, result.filename, result.uploaded, result.variants, result.metadata
  return result;
}
```

### Update Image Metadata

```typescript
async function updateImage(
  accountId: string,
  apiToken: string,
  imageId: string,
  updates: { requireSignedURLs?: boolean; metadata?: Record<string, any> }
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1/${imageId}`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(updates),
    }
  );
  
  return await response.json();
}
```

### Delete Image

```typescript
async function deleteImage(accountId: string, apiToken: string, imageId: string) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1/${imageId}`,
    {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${apiToken}` },
    }
  );
  
  return await response.json();
}
```

### Get Base Image Blob

Fetch the original uploaded image (or near-lossless version for large images):

```typescript
async function getImageBlob(accountId: string, apiToken: string, imageId: string) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1/${imageId}/blob`,
    {
      headers: { 'Authorization': `Bearer ${apiToken}` },
    }
  );
  
  return await response.blob();
}
```

## Variants

Variants define how images should be resized. Every account has a default `public` variant. You can create up to 100 variants.

### URL Pattern for Stored Images

```
https://imagedelivery.net/<ACCOUNT_HASH>/<IMAGE_ID>/<VARIANT_NAME>
```

Example:
```
https://imagedelivery.net/abc123/my-image-id/thumbnail
https://imagedelivery.net/abc123/my-image-id/hero
```

### Create Named Variant (API)

```bash
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/images/v1/variants \
  -X POST \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "thumbnail",
    "options": {
      "fit": "cover",
      "width": 200,
      "height": 200,
      "metadata": "none"
    },
    "neverRequireSignedURLs": false
  }'
```

### Fit Modes

| Fit Mode | Behavior |
|----------|----------|
| `scale-down` | Shrink to fit within dimensions, never enlarge |
| `contain` | Resize to largest size within dimensions, preserve aspect ratio |
| `cover` | Resize to fill dimensions exactly, crop if needed |
| `crop` | Same as `cover` for large images, `scale-down` for small |
| `pad` | Resize to fit, fill extra area with background color |
| `squeeze` | Resize to exact dimensions, may distort aspect ratio |

### Flexible Variants

Enable dynamic resizing with inline parameters:

```bash
# Enable flexible variants
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/images/v1/config \
  -X PATCH \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"flexible_variants": true}'
```

Then use transformation parameters directly in URLs:

```
https://imagedelivery.net/<ACCOUNT_HASH>/<IMAGE_ID>/w=400,h=300,fit=cover
https://imagedelivery.net/<ACCOUNT_HASH>/<IMAGE_ID>/w=800,sharpen=3,quality=85
```

**Limitation**: Flexible variants cannot be used with signed URLs (private images)

## Image Transformations (External Images)

Transform images stored outside Cloudflare Images (on your origin, R2, etc.)

### Transform via URL

#### Enable Transformations

1. Go to Cloudflare Dashboard > [Zone] > Speed > Optimization
2. Enable "Image Transformations"

#### URL Format

```
https://yourdomain.com/cdn-cgi/image/<OPTIONS>/<SOURCE_URL>
```

Examples:
```
https://example.com/cdn-cgi/image/width=800,quality=85/uploads/photo.jpg
https://example.com/cdn-cgi/image/fit=cover,width=400,height=300/https://external.com/image.png
```

#### Common Options

```
width=<PIXELS>         # w=<PIXELS> (alias)
height=<PIXELS>        # h=<PIXELS> (alias)
fit=scale-down         # scale-down|contain|cover|crop|pad|squeeze
quality=85             # q=85 (1-100)
format=auto            # f=auto (auto|webp|avif|jpeg|png)
dpr=2                  # Device Pixel Ratio (1-3)
gravity=auto           # auto|left|right|top|bottom|face|0.5x0.5
sharpen=2              # 0-10
blur=10                # 1-250
rotate=90              # 90|180|270
background=white       # CSS color for pad fit mode
metadata=none          # none|copyright|keep
```

### Transform via Workers

Use Cloudflare Workers for programmatic control:

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    
    // Parse Accept header for format negotiation
    const accept = request.headers.get('Accept') || '';
    let format: 'avif' | 'webp' | undefined;
    if (/image\/avif/.test(accept)) {
      format = 'avif';
    } else if (/image\/webp/.test(accept)) {
      format = 'webp';
    }
    
    // Define transformation options
    const options: RequestInitCfPropertiesImage = {
      cf: {
        image: {
          width: 800,
          height: 600,
          fit: 'cover',
          quality: 85,
          format,
          gravity: 'auto',
          sharpen: 1,
        },
      },
    };
    
    // Fetch and transform
    const imageURL = 'https://example.com/uploads/photo.jpg';
    return fetch(imageURL, options);
  },
};
```

#### Advanced Worker Pattern: Custom URL Scheme

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    
    // Custom URL: /img/<PRESET>/<IMAGE_PATH>
    const [, , preset, ...pathParts] = url.pathname.split('/');
    const imagePath = pathParts.join('/');
    
    // Define presets
    const presets: Record<string, any> = {
      thumbnail: { width: 200, height: 200, fit: 'cover' },
      large: { width: 1200, quality: 90 },
      avatar: { width: 128, height: 128, fit: 'cover', gravity: 'face' },
    };
    
    const imageOptions = presets[preset] || {};
    
    // Prevent request loops
    if (/image-resizing/.test(request.headers.get('via') || '')) {
      return fetch(`https://example.com/${imagePath}`);
    }
    
    // Format negotiation
    const accept = request.headers.get('Accept') || '';
    if (/image\/avif/.test(accept)) {
      imageOptions.format = 'avif';
    } else if (/image\/webp/.test(accept)) {
      imageOptions.format = 'webp';
    }
    
    return fetch(`https://example.com/${imagePath}`, {
      cf: { image: imageOptions },
    });
  },
};
```

#### Error Handling in Workers

```typescript
const response = await fetch(imageURL, options);

if (response.ok || response.redirected) {
  return response;
}

// Option 1: Redirect to original
return Response.redirect(imageURL, 307);

// Option 2: Return placeholder
return fetch('https://example.com/placeholder.png');
```

#### Prevent Request Loops

When Workers handle paths that overlap with source images:

```typescript
// Check for image-resizing header to avoid loops
if (/image-resizing/.test(request.headers.get('via') || '')) {
  return fetch(request); // Pass through to origin
}
```

## Advanced Transformation Options

### Responsive Images with srcset

```html
<img
  src="https://example.com/cdn-cgi/image/width=400/photo.jpg"
  srcset="
    https://example.com/cdn-cgi/image/width=400/photo.jpg 400w,
    https://example.com/cdn-cgi/image/width=800/photo.jpg 800w,
    https://example.com/cdn-cgi/image/width=1200/photo.jpg 1200w
  "
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  alt="Responsive image"
/>
```

### Background Removal (Segmentation)

Automatically remove image backgrounds:

```
https://example.com/cdn-cgi/image/segment=foreground/photo.jpg
```

```typescript
cf: {
  image: {
    segment: 'foreground', // Remove background, keep subject
  },
}
```

### Face Detection with Gravity

Automatically crop around detected faces:

```
https://example.com/cdn-cgi/image/fit=cover,width=400,height=400,gravity=face,zoom=0.5/portrait.jpg
```

```typescript
cf: {
  image: {
    fit: 'cover',
    width: 400,
    height: 400,
    gravity: 'face', // Auto-detect faces
    zoom: 0.5,       // 0=include background, 1=tight crop
  },
}
```

### Trim Borders

Remove borders automatically or by color:

```typescript
// Auto-detect border color
cf: { image: { trim: 'border' } }

// Specific color with tolerance
cf: {
  image: {
    trim: {
      border: {
        color: '#FFFFFF',
        tolerance: 10,
        keep: 5, // Keep 5px of original border
      },
    },
  },
}
```

### Adjustments

```typescript
cf: {
  image: {
    brightness: 1.2,   // 0.5=darker, 2.0=brighter
    contrast: 1.1,     // 0.5=low, 2.0=high
    gamma: 1.0,        // Exposure adjustment
    saturation: 1.2,   // 0=grayscale, 2.0=twice saturated
    blur: 5,           // 1-250
    sharpen: 2,        // 0-10
  },
}
```

### Output Formats

```typescript
// Auto-select best format based on browser support
format: 'auto' // AVIF > WebP > JPEG

// Specific formats
format: 'avif'   // Modern, best compression
format: 'webp'   // Wide support, good compression
format: 'jpeg'   // Progressive JPEG
format: 'baseline-jpeg' // For old devices
format: 'json'   // Returns metadata instead of image
```

## Private Images with Signed URLs

Restrict image access by requiring signed URL tokens:

### Enable Signed URLs

```typescript
// On upload
const formData = new FormData();
formData.append('file', fileBuffer);
formData.append('requireSignedURLs', 'true');

await fetch(`https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1`, {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${apiToken}` },
  body: formData,
});

// Or update existing
await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1/${imageId}`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${apiToken}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ requireSignedURLs: true }),
  }
);
```

### Generate Signed URL

Get signing key from Dashboard > Images > Keys

```typescript
const SIGNING_KEY = 'your-signing-key';
const EXPIRATION_SECONDS = 60 * 60; // 1 hour

async function generateSignedUrl(
  accountHash: string,
  imageId: string,
  variant: string,
  signingKey: string,
  expirationSeconds: number
): Promise<string> {
  const encoder = new TextEncoder();
  const secretKeyData = encoder.encode(signingKey);
  
  const key = await crypto.subtle.importKey(
    'raw',
    secretKeyData,
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  
  const url = new URL(
    `https://imagedelivery.net/${accountHash}/${imageId}/${variant}`
  );
  
  const expiry = Math.floor(Date.now() / 1000) + expirationSeconds;
  url.searchParams.set('exp', String(expiry));
  
  const stringToSign = url.pathname + '?' + url.searchParams.toString();
  const mac = await crypto.subtle.sign('HMAC', key, encoder.encode(stringToSign));
  const sig = Array.from(new Uint8Array(mac))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
  
  url.searchParams.set('sig', sig);
  
  return url.toString();
}

// Usage
const signedUrl = await generateSignedUrl(
  'abc123',
  'image-id',
  'public',
  SIGNING_KEY,
  3600
);
```

### Public Access for Specific Variants

Allow certain variants to remain public even when `requireSignedURLs` is true:

```bash
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/images/v1/variants \
  -X POST \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "public-thumbnail",
    "options": {
      "width": 200,
      "height": 200,
      "fit": "cover"
    },
    "neverRequireSignedURLs": true
  }'
```

## Webhooks

Receive notifications when images are uploaded or deleted:

### Configure Webhook

```bash
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/images/v1/webhooks \
  -X PUT \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-api.com/webhooks/images",
    "secret": "your-webhook-secret"
  }'
```

### Webhook Payload

```json
{
  "event": "image.uploaded",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "id": "image-id",
    "filename": "photo.jpg",
    "metadata": { "userId": "12345" },
    "uploaded": "2024-01-15T10:30:00Z",
    "requireSignedURLs": false,
    "variants": [
      "https://imagedelivery.net/abc123/image-id/public",
      "https://imagedelivery.net/abc123/image-id/thumbnail"
    ]
  }
}
```

Events: `image.uploaded`, `image.deleted`

## Wrangler Integration

### Upload via Wrangler

Wrangler doesn't have built-in Cloudflare Images commands, but you can use the API:

```typescript
// scripts/upload-image.ts
import fs from 'fs';
import FormData from 'form-data';
import fetch from 'node-fetch';

async function uploadImage(filePath: string) {
  const accountId = process.env.CLOUDFLARE_ACCOUNT_ID;
  const apiToken = process.env.CLOUDFLARE_API_TOKEN;
  
  const formData = new FormData();
  formData.append('file', fs.createReadStream(filePath));
  
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      },
      body: formData,
    }
  );
  
  const result = await response.json();
  console.log('Uploaded:', result);
}

uploadImage('./photo.jpg');
```

### Access Images from Workers

Store account hash as an environment variable:

```toml
# wrangler.toml
[vars]
IMAGES_ACCOUNT_HASH = "abc123def456"
```

```typescript
// worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const imageId = 'my-image-id';
    const variant = 'thumbnail';
    
    const imageUrl = `https://imagedelivery.net/${env.IMAGES_ACCOUNT_HASH}/${imageId}/${variant}`;
    
    return Response.redirect(imageUrl, 302);
  },
};
```

### Transform R2 Images

Combine R2 storage with Images transformations:

```typescript
// wrangler.toml
[[r2_buckets]]
binding = "IMAGES_BUCKET"
bucket_name = "my-images"

// worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const key = url.pathname.slice(1);
    
    // Fetch from R2
    const object = await env.IMAGES_BUCKET.get(key);
    if (!object) {
      return new Response('Not Found', { status: 404 });
    }
    
    // Transform image
    const response = new Response(object.body);
    return fetch(url.toString(), {
      cf: {
        image: {
          width: 800,
          quality: 85,
          format: 'auto',
        },
      },
    });
  },
};
```

## Supported Formats & Limitations

### Input Formats
- JPEG, PNG, GIF (including animations)
- WebP (including animations)
- SVG (sanitized, not resized)
- HEIC (decoded but served as web-safe formats)

### Output Formats
- JPEG (progressive or baseline)
- PNG
- GIF, WebP (animations supported)
- AVIF (best effort, may fallback to WebP)
- SVG (passed through)

### Size Limits

| Limit | Value |
|-------|-------|
| Maximum file size (upload to storage) | 10 MB |
| Maximum dimension (single side) | 12,000 px |
| Maximum image area | 100 megapixels |
| Metadata size | 1024 bytes |
| Animated GIF/WebP total size | 50 megapixels |

### Format-Specific Limits

| Format | Max Dimension (Hard) | Max Dimension (Soft, when overloaded) |
|--------|---------------------|--------------------------------------|
| AVIF | 1,600 px | 640 px |
| WebP | N/A | 2,560 px (lossy), 1920 px (lossless) |
| Other | 12,000 px | N/A |

**Note**: AVIF encoding may fallback to WebP for large images due to processing time

## Pricing (as of 2024)

### Free Plan
- 5,000 unique transformations/month
- Transformations only (no storage)
- After limit: existing cached images served, new transformations return error

### Paid Plan

| Metric | Price |
|--------|-------|
| Images Transformed | 5,000 included + $0.50/1,000/month |
| Images Stored | $5/100,000 images/month |
| Images Delivered | $1/100,000 images/month |

**Unique Transformation**: Same image + parameters counted once per 30 days (sliding window)

## Common Patterns

### Pattern: User Avatar Upload

```typescript
// 1. Generate upload URL (server)
const { uploadURL, imageId } = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v2/direct_upload`,
  {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${apiToken}` },
    body: new URLSearchParams({
      requireSignedURLs: 'false',
      metadata: JSON.stringify({ userId, type: 'avatar' }),
    }),
  }
).then(r => r.json()).then(d => d.result);

// 2. Client uploads
await fetch(uploadURL, {
  method: 'POST',
  body: formDataWithFile,
});

// 3. Store imageId in user record
await db.users.update(userId, { avatarImageId: imageId });

// 4. Display avatar
const avatarUrl = `https://imagedelivery.net/${accountHash}/${imageId}/avatar`;
```

### Pattern: Product Image Gallery

```typescript
// Upload multiple images with metadata
const productId = 'prod-123';
const images = [];

for (const [index, file] of files.entries()) {
  const formData = new FormData();
  formData.append('file', file);
  formData.append('metadata', JSON.stringify({
    productId,
    position: index,
    altText: file.name,
  }));
  
  const result = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1`,
    {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiToken}` },
      body: formData,
    }
  ).then(r => r.json());
  
  images.push(result.result.id);
}

// Store in database
await db.products.update(productId, { imageIds: images });
```

### Pattern: Optimized Blog Images (R2 + Transformations)

```typescript
// Worker serving blog
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Match: /images/blog/2024/photo.jpg
    const match = url.pathname.match(/^\/images\/(.+)$/);
    if (!match) return new Response('Not Found', { status: 404 });
    
    const key = match[1];
    const object = await env.BLOG_IMAGES.get(key);
    if (!object) return new Response('Not Found', { status: 404 });
    
    // Transform for web
    const accept = request.headers.get('Accept') || '';
    const format = /image\/avif/.test(accept) ? 'avif' : 
                   /image\/webp/.test(accept) ? 'webp' : undefined;
    
    return new Response(object.body, {
      headers: {
        'Content-Type': object.httpMetadata?.contentType || 'image/jpeg',
        'Cache-Control': 'public, max-age=31536000, immutable',
      },
      cf: {
        image: {
          width: 1200,
          quality: 85,
          format,
          sharpen: 1,
        },
      } as any,
    });
  },
};
```

### Pattern: Social Media Sharing Images

```typescript
// Generate optimized share images
const shareVariants = [
  { name: 'og-image', width: 1200, height: 630, fit: 'cover' },
  { name: 'twitter-card', width: 1200, height: 600, fit: 'cover' },
  { name: 'linkedin-share', width: 1200, height: 627, fit: 'cover' },
];

for (const variant of shareVariants) {
  await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/images/v1/variants`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        id: variant.name,
        options: variant,
      }),
    }
  );
}

// Use in meta tags
const ogImageUrl = `https://imagedelivery.net/${accountHash}/${imageId}/og-image`;
```

## Best Practices

### 1. Use Appropriate Fit Modes
- `cover`: Hero images, thumbnails, avatars (fills space, crops)
- `contain`: Product images, artwork (preserves full image)
- `scale-down`: Ensure images aren't unnecessarily enlarged

### 2. Format Selection
- Use `format=auto` for automatic AVIF/WebP/JPEG selection
- For Workers, parse `Accept` header for format negotiation
- AVIF: Best compression, use for modern browsers
- WebP: Wide support, good compression
- JPEG: Fallback for older browsers

### 3. Quality Settings
- `85`: Good default balance
- `90-95`: High-quality images (portfolios, product photos)
- `75-80`: Acceptable quality for faster loading
- WebP lossless: `quality=100`

### 4. Responsive Images
- Use `srcset` with multiple widths (400w, 800w, 1200w)
- Set appropriate `sizes` attribute
- Consider `dpr=2` for Retina displays

### 5. Metadata Management
- Use metadata for internal tracking (userId, category, etc.)
- Metadata is never exposed to end users
- Limit: 1024 bytes per image

### 6. Caching Strategy
- Set long cache headers for immutable images
- Use versioned URLs or query params for cache busting
- Transformations cached for 30 days

### 7. Private Images
- Enable `requireSignedURLs` for sensitive content
- Set appropriate expiration times
- Use `neverRequireSignedURLs` on variants for public thumbnails

### 8. Direct Creator Upload
- Set reasonable expiration (default 30 min)
- Validate file types on client
- Use webhooks to track upload completion

### 9. SVG Handling
- SVGs are sanitized automatically (scripts, links removed)
- Use as regular images, resizing parameters ignored
- Good for logos, icons served via Images

### 10. Error Handling
- Check response status codes
- Implement fallbacks for transformation failures
- Use `onerror=redirect` for transformations (redirects to original)

## Troubleshooting

### Issue: Transformation returns 9422 error
**Cause**: Exceeded 5,000 unique transformations on Free plan  
**Solution**: Upgrade to Paid plan or optimize variant usage

### Issue: AVIF not being served
**Cause**: Image too large or processing timeout  
**Solution**: Automatic fallback to WebP; reduce source image size

### Issue: Signed URL not working
**Cause**: Incorrect signature or expired token  
**Solution**: Verify signing key, check expiration time, ensure correct pathname format

### Issue: Request loop in Worker
**Cause**: Worker transforming images at same path  
**Solution**: Check `via` header for `image-resizing` string

### Issue: Custom ID upload fails
**Cause**: Trying to use `requireSignedURLs=true` with custom ID  
**Solution**: Custom IDs cannot use signed URLs; set to `false`

### Issue: Flexible variants not working
**Cause**: Feature not enabled  
**Solution**: Enable via API or Dashboard > Images > Delivery > Flexible variants

### Issue: Large GIF not transforming
**Cause**: Animation exceeds 50 megapixels  
**Solution**: Reduce frame count or dimensions; set `anim=false` to convert to still

## Additional Resources

- [Cloudflare Images Docs](https://developers.cloudflare.com/images/)
- [API Reference](https://developers.cloudflare.com/api/resources/images/)
- [Workers Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/request/#the-cf-property-requestinitcfproperties)
- [Community Forum](https://community.cloudflare.com/c/developers/images/63)
