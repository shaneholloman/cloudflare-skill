# Cloudflare R2 Object Storage Skill

## Overview

Cloudflare R2 is S3-compatible object storage with zero egress fees. This skill provides comprehensive guidance for working with R2, including the Workers API, S3 API compatibility, Wrangler CLI operations, and common patterns.

## Core Concepts

### Buckets and Objects
- **Buckets**: Containers for objects, fundamental unit of performance/scaling/access
- **Objects**: Unstructured data stored in buckets with keys (paths)
- **Storage Classes**: Standard (frequent access) vs Infrequent Access (30-day minimum, retrieval fees)
- **Region**: Always `auto` (or empty/`us-east-1` as aliases for compatibility)

### API Options
1. **Workers API**: Native R2 API for Cloudflare Workers (recommended for Workers)
2. **S3 API**: S3-compatible REST API at `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`
3. **Public Buckets**: Direct internet access to bucket contents

## Workers API (Recommended for Workers)

### Binding Configuration

```jsonc
// wrangler.jsonc
{
  "r2_buckets": [
    {
      "binding": "MY_BUCKET",
      "bucket_name": "my-bucket-name"
    }
  ]
}
```

```toml
# wrangler.toml
[[r2_buckets]]
binding = 'MY_BUCKET'
bucket_name = 'my-bucket-name'
```

### Core Operations

#### PUT (Upload Object)

```typescript
// Basic upload
await env.MY_BUCKET.put(key, value);

// With metadata and options
await env.MY_BUCKET.put(key, value, {
  httpMetadata: {
    contentType: 'image/jpeg',
    contentLanguage: 'en-US',
    contentDisposition: 'attachment; filename="photo.jpg"',
    contentEncoding: 'gzip',
    cacheControl: 'max-age=3600',
    cacheExpiry: new Date(Date.now() + 86400000)
  },
  customMetadata: {
    userId: '12345',
    uploadedBy: 'worker'
  },
  storageClass: 'Standard', // or 'InfrequentAccess'
  md5: arrayBufferOrHexString, // integrity check (only one hash algorithm allowed)
  sha256: arrayBufferOrHexString,
  ssecKey: arrayBufferOrHexString // 32 bytes for SSE-C encryption
});

// Value types: ReadableStream | ArrayBuffer | ArrayBufferView | string | null | Blob
```

**Key Points:**
- Returns `R2Object` with metadata on success, `null` if preconditions fail
- Writes are strongly consistent globally
- Only one checksum algorithm can be specified (md5/sha1/sha256/sha384/sha512)
- Use `httpMetadata` for standard HTTP headers, `customMetadata` for custom key-value pairs

#### GET (Retrieve Object)

```typescript
// Get object with body
const object = await env.MY_BUCKET.get(key);
if (object === null) {
  return new Response('Not found', { status: 404 });
}

// Access body as different formats
const arrayBuffer = await object.arrayBuffer();
const text = await object.text();
const json = await object.json();
const blob = await object.blob();
const stream = object.body; // ReadableStream

// With conditional operations
const object = await env.MY_BUCKET.get(key, {
  onlyIf: {
    etagMatches: '"abc123"',
    etagDoesNotMatch: '"def456"',
    uploadedBefore: new Date('2024-01-01'),
    uploadedAfter: new Date('2023-01-01')
  },
  range: { 
    offset: 0, 
    length: 1024 
  }, // or { suffix: 1024 }
  ssecKey: arrayBufferOrHexString // if object encrypted with SSE-C
});

// Alternative: use Headers for conditionals
const object = await env.MY_BUCKET.get(key, {
  onlyIf: new Headers({
    'If-Match': '"abc123"',
    'If-None-Match': '"def456"',
    'If-Modified-Since': 'Wed, 21 Oct 2023 07:28:00 GMT',
    'If-Unmodified-Since': 'Wed, 21 Oct 2024 07:28:00 GMT'
  })
});
```

**Key Points:**
- Returns `R2ObjectBody` (object + body) on success, `R2Object` (no body) if preconditions fail, `null` if not found
- If preconditions fail, body is `undefined` but metadata still returned
- Use ranged reads for large files to reduce bandwidth
- Conditional operations reduce latency by avoiding body transfer on condition failure

#### HEAD (Get Metadata Only)

```typescript
const object = await env.MY_BUCKET.head(key);
if (object === null) {
  return new Response('Not found', { status: 404 });
}

// Access metadata
console.log(object.key, object.version, object.size, object.etag);
console.log(object.uploaded, object.httpMetadata, object.customMetadata);
console.log(object.storageClass); // 'Standard' or 'InfrequentAccess'
console.log(object.checksums); // { md5?, sha1?, sha256?, sha384?, sha512? }
```

**Key Points:**
- Faster than GET when only metadata needed
- Returns `R2Object | null`

#### DELETE (Remove Objects)

```typescript
// Delete single object
await env.MY_BUCKET.delete(key);

// Delete multiple objects (up to 1000)
await env.MY_BUCKET.delete([key1, key2, key3]);
```

**Key Points:**
- Returns `Promise<void>`
- Deletes are strongly consistent globally
- Max 1000 keys per batch delete

#### LIST (List Objects)

```typescript
// Basic list
const listed = await env.MY_BUCKET.list();

// With options
const listed = await env.MY_BUCKET.list({
  limit: 1000, // max 1000, default 1000
  prefix: 'photos/', // only keys starting with prefix
  cursor: 'cursor-from-previous-list', // pagination token
  delimiter: '/', // group by delimiter for directory-like structure
  include: ['httpMetadata', 'customMetadata'] // include metadata (may reduce returned count)
});

// Pagination with include
let truncated = listed.truncated;
let cursor = listed.cursor;

// WRONG: Don't compare object count to limit with include
while (listed.objects.length < options.limit) { // ❌
  // ...
}

// CORRECT: Use truncated property
while (truncated) { // ✅
  const next = await env.MY_BUCKET.list({
    limit: 1000,
    cursor: cursor,
    include: ['customMetadata']
  });
  listed.objects.push(...next.objects);
  truncated = next.truncated;
  cursor = next.cursor;
}

// Response structure
interface R2Objects {
  objects: R2Object[]; // listed objects (sorted lexicographically)
  truncated: boolean; // more results available?
  cursor?: string; // token for next page
  delimitedPrefixes: string[]; // prefixes between prefix and next delimiter
}
```

**Key Points:**
- Lexicographic (alphabetical) ordering
- `include` with metadata may return fewer than `limit` objects to accommodate metadata size
- Must set compatibility date >= `2022-08-04` or use `r2_list_honor_include` flag, otherwise `include` is ignored
- Use `truncated` property, NOT object count comparison
- `delimiter` enables directory-like hierarchical listing

### Multipart Uploads (Large Objects)

```typescript
// Create multipart upload
const multipart = await env.MY_BUCKET.createMultipartUpload(key, {
  httpMetadata: { contentType: 'video/mp4' },
  customMetadata: { videoId: '123' },
  storageClass: 'Standard',
  ssecKey: arrayBufferOrHexString
});

// Upload parts (all parts must be uniform size except last)
const uploadedParts: R2UploadedPart[] = [];
for (let i = 0; i < partCount; i++) {
  const part = await multipart.uploadPart(i + 1, partData, {
    ssecKey: arrayBufferOrHexString // must match createMultipartUpload key
  });
  uploadedParts.push(part);
}

// Complete upload
const object = await multipart.complete(uploadedParts);

// Or abort if needed
await multipart.abort();

// Resume existing upload
const multipart = env.MY_BUCKET.resumeMultipartUpload(key, uploadId);
// Note: resumeMultipartUpload doesn't validate uploadId/existence (low latency)
```

**Key Points:**
- Uncompleted uploads auto-abort after 7 days
- All parts must be uniform size (except final part can be smaller)
- `resumeMultipartUpload` doesn't validate existence (can fail on subsequent operations)
- Part numbers start at 1
- Add error handling for parallel completion/abort

### R2Object Interface

```typescript
interface R2Object {
  key: string;
  version: string; // unique per upload
  size: number; // bytes
  etag: string; // unquoted etag
  httpEtag: string; // quoted etag (RFC 9110 compliant, use for headers)
  uploaded: Date;
  httpMetadata: R2HTTPMetadata;
  customMetadata: Record<string, string>;
  range?: R2Range;
  checksums: R2Checksums; // { md5?, sha1?, sha256?, sha384?, sha512? }
  storageClass: 'Standard' | 'InfrequentAccess';
  ssecKeyMd5?: string; // hex MD5 of SSE-C key (identifies which key needed)
  
  writeHttpMetadata(headers: Headers): void; // applies httpMetadata to Headers object
}

interface R2ObjectBody extends R2Object {
  body: ReadableStream;
  bodyUsed: boolean;
  arrayBuffer(): Promise<ArrayBuffer>;
  text(): Promise<string>;
  json<T>(): Promise<T>;
  blob(): Promise<Blob>;
}
```

**Key Points:**
- Use `httpEtag` for response headers (quoted), `etag` for comparisons
- `writeHttpMetadata()` convenience method applies HTTP headers
- `checksums` populated if provided during PUT (MD5 auto-included for non-multipart)

## S3 API Compatibility

### Endpoint and Authentication

```
https://<ACCOUNT_ID>.r2.cloudflarestorage.com
```

**Region:** `auto` (aliases: empty string, `us-east-1`)

### S3 API Implementation Status

#### Bucket Operations (Implemented)
- ✅ ListBuckets, HeadBucket, CreateBucket, DeleteBucket
- ✅ CORS: GetBucketCors, PutBucketCors, DeleteBucketCors
- ✅ Lifecycle: GetBucketLifecycleConfiguration, PutBucketLifecycleConfiguration
- ✅ GetBucketLocation, GetBucketEncryption

#### Object Operations (Implemented)
- ✅ HeadObject, GetObject, PutObject, DeleteObject, DeleteObjects
- ✅ CopyObject (with storage class transitions)
- ✅ Multipart: CreateMultipartUpload, UploadPart, UploadPartCopy, CompleteMultipartUpload, AbortMultipartUpload, ListMultipartUploads, ListParts
- ✅ ListObjects, ListObjectsV2 (use V2 for new applications)

#### Not Implemented
- ❌ ACLs, versioning, object locking, tagging
- ❌ Bucket policies, notifications, analytics
- ❌ Server-side encryption (except SSE-C ✅)

#### SSE-C (Server-Side Encryption with Customer Keys)
```typescript
// Encrypt with SSE-C (32-byte key)
await env.MY_BUCKET.put(key, data, {
  ssecKey: hexStringOrArrayBuffer // 32 bytes
});

// Retrieve encrypted object
const object = await env.MY_BUCKET.get(key, {
  ssecKey: sameKeyUsedForEncryption
});

// Copy encrypted object
const headers = new Headers({
  'x-amz-copy-source': `/bucket/${sourceKey}`,
  'x-amz-copy-source-server-side-encryption-customer-algorithm': 'AES256',
  'x-amz-copy-source-server-side-encryption-customer-key': base64Key,
  'x-amz-copy-source-server-side-encryption-customer-key-MD5': base64Md5,
  'x-amz-server-side-encryption-customer-algorithm': 'AES256',
  'x-amz-server-side-encryption-customer-key': base64NewKey,
  'x-amz-server-side-encryption-customer-key-MD5': base64NewMd5
});
```

### S3 SDK Example (AWS SDK)

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${accountId}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: env.R2_ACCESS_KEY_ID,
    secretAccessKey: env.R2_SECRET_ACCESS_KEY
  }
});

// Upload
await s3.send(new PutObjectCommand({
  Bucket: 'my-bucket',
  Key: 'path/to/file.txt',
  Body: fileData,
  ContentType: 'text/plain',
  StorageClass: 'STANDARD' // or 'STANDARD_IA'
}));

// Download
const response = await s3.send(new GetObjectCommand({
  Bucket: 'my-bucket',
  Key: 'path/to/file.txt'
}));
const body = await response.Body.transformToString();
```

## Wrangler CLI Commands

### Bucket Management

```bash
# Create bucket
wrangler r2 bucket create my-bucket \
  --location=enam \
  --storage-class=Standard

# List buckets
wrangler r2 bucket list

# Get bucket info
wrangler r2 bucket info my-bucket

# Delete bucket (must be empty)
wrangler r2 bucket delete my-bucket

# Update default storage class
wrangler r2 bucket update-storage-class my-bucket --storage-class=InfrequentAccess
```

**Location Hints:**
- `weur`: Western Europe
- `eeur`: Eastern Europe
- `apac`: Asia Pacific
- `oc`: Oceania
- `wnam`: Western North America
- `enam`: Eastern North America

**Jurisdictions (overrides location):**
- `eu`: European Union
- `fedramp`: FedRAMP-compliant data centers

### Object Management

```bash
# Upload object
wrangler r2 object put my-bucket/path/to/file.txt \
  --file=./local/file.txt \
  --content-type=text/plain \
  --storage-class=Standard

# Download object
wrangler r2 object get my-bucket/path/to/file.txt \
  --file=./download/file.txt

# Delete object
wrangler r2 object delete my-bucket/path/to/file.txt

# List objects
wrangler r2 object list my-bucket \
  --prefix=photos/ \
  --delimiter=/

# Copy object (change storage class)
aws s3api copy-object \
  --endpoint-url https://${ACCOUNT_ID}.r2.cloudflarestorage.com \
  --bucket my-bucket \
  --key path/to/file.txt \
  --copy-source /my-bucket/path/to/file.txt \
  --storage-class STANDARD_IA
```

## Common Patterns and Best Practices

### 1. File Upload with Validation

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method !== 'PUT') {
      return new Response('Method not allowed', { status: 405 });
    }

    const url = new URL(request.url);
    const key = url.pathname.slice(1);

    // Validate key (prevent path traversal)
    if (!key || key.includes('..') || key.startsWith('/')) {
      return new Response('Invalid key', { status: 400 });
    }

    const contentType = request.headers.get('content-type') || 'application/octet-stream';

    try {
      const object = await env.MY_BUCKET.put(key, request.body, {
        httpMetadata: {
          contentType
        },
        customMetadata: {
          uploadedAt: new Date().toISOString(),
          uploadedFrom: request.headers.get('cf-connecting-ip') || 'unknown'
        }
      });

      return Response.json({
        success: true,
        key: object.key,
        size: object.size,
        etag: object.httpEtag
      });
    } catch (error) {
      return new Response('Upload failed', { status: 500 });
    }
  }
};
```

### 2. Streaming Large Files

```typescript
async function streamLargeFile(key: string, env: Env): Promise<Response> {
  const object = await env.MY_BUCKET.get(key);
  
  if (object === null) {
    return new Response('Not found', { status: 404 });
  }

  const headers = new Headers();
  object.writeHttpMetadata(headers);
  headers.set('etag', object.httpEtag);

  return new Response(object.body, { headers });
}
```

### 3. Conditional GET (304 Not Modified)

```typescript
async function conditionalGet(
  request: Request, 
  key: string, 
  env: Env
): Promise<Response> {
  const ifNoneMatch = request.headers.get('if-none-match');
  
  const object = await env.MY_BUCKET.get(key, {
    onlyIf: {
      etagDoesNotMatch: ifNoneMatch?.replace(/"/g, '') || ''
    }
  });

  if (object === null) {
    return new Response('Not found', { status: 404 });
  }

  // If precondition failed, body is undefined (etag matches)
  if (object.body === undefined) {
    return new Response(null, { 
      status: 304,
      headers: {
        'etag': object.httpEtag,
        'cache-control': 'public, max-age=3600'
      }
    });
  }

  const headers = new Headers();
  object.writeHttpMetadata(headers);
  headers.set('etag', object.httpEtag);
  headers.set('cache-control', 'public, max-age=3600');

  return new Response(object.body, { headers });
}
```

### 4. Batch Operations

```typescript
async function batchDelete(keys: string[], env: Env): Promise<void> {
  const BATCH_SIZE = 1000;
  
  for (let i = 0; i < keys.length; i += BATCH_SIZE) {
    const batch = keys.slice(i, i + BATCH_SIZE);
    await env.MY_BUCKET.delete(batch);
  }
}

async function listAllObjects(env: Env, prefix?: string): Promise<R2Object[]> {
  const allObjects: R2Object[] = [];
  let cursor: string | undefined;
  let truncated = true;

  while (truncated) {
    const listed = await env.MY_BUCKET.list({
      limit: 1000,
      prefix,
      cursor
    });

    allObjects.push(...listed.objects);
    truncated = listed.truncated;
    cursor = listed.cursor;
  }

  return allObjects;
}
```

### 5. Multipart Upload with Progress

```typescript
async function uploadLargeFile(
  key: string,
  file: File,
  env: Env,
  onProgress?: (percent: number) => void
): Promise<R2Object> {
  const PART_SIZE = 5 * 1024 * 1024; // 5MB parts
  const partCount = Math.ceil(file.size / PART_SIZE);

  const multipart = await env.MY_BUCKET.createMultipartUpload(key, {
    httpMetadata: {
      contentType: file.type
    }
  });

  const uploadedParts: R2UploadedPart[] = [];

  try {
    for (let i = 0; i < partCount; i++) {
      const start = i * PART_SIZE;
      const end = Math.min(start + PART_SIZE, file.size);
      const partData = file.slice(start, end);

      const part = await multipart.uploadPart(i + 1, partData);
      uploadedParts.push(part);

      if (onProgress) {
        onProgress(Math.round(((i + 1) / partCount) * 100));
      }
    }

    return await multipart.complete(uploadedParts);
  } catch (error) {
    await multipart.abort();
    throw error;
  }
}
```

### 6. Storage Class Lifecycle

```typescript
// Transition object to Infrequent Access
async function transitionToIA(key: string, env: Env): Promise<void> {
  const object = await env.MY_BUCKET.head(key);
  if (!object) throw new Error('Object not found');

  // Copy to self with new storage class
  await env.MY_BUCKET.put(key, object.body, {
    storageClass: 'InfrequentAccess',
    httpMetadata: object.httpMetadata,
    customMetadata: object.customMetadata
  });
}
```

### 7. Public Bucket Patterns

```typescript
// Serve public bucket with custom domain
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const key = url.pathname.slice(1);

    const object = await env.MY_BUCKET.get(key);
    
    if (object === null) {
      return new Response('Not found', { status: 404 });
    }

    const headers = new Headers();
    object.writeHttpMetadata(headers);
    headers.set('etag', object.httpEtag);
    headers.set('access-control-allow-origin', '*');
    headers.set('cache-control', 'public, max-age=31536000, immutable');

    return new Response(object.body, { headers });
  }
};
```

### 8. Checksums and Integrity

```typescript
// Upload with checksum validation
async function uploadWithChecksum(
  key: string,
  data: ArrayBuffer,
  env: Env
): Promise<R2Object> {
  const hash = await crypto.subtle.digest('SHA-256', data);

  return await env.MY_BUCKET.put(key, data, {
    sha256: hash
  });
}

// Verify checksum on retrieval
async function verifyChecksum(key: string, env: Env): Promise<boolean> {
  const object = await env.MY_BUCKET.get(key);
  if (!object) return false;

  const data = await object.arrayBuffer();
  const hash = await crypto.subtle.digest('SHA-256', data);
  
  const hashArray = Array.from(new Uint8Array(hash));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

  const storedHash = object.checksums.sha256;
  if (!storedHash) return false;

  const storedHashHex = Array.from(new Uint8Array(storedHash))
    .map(b => b.toString(16).padStart(2, '0')).join('');

  return hashHex === storedHashHex;
}
```

## TypeScript Types

```typescript
// Environment with R2 binding
interface Env {
  MY_BUCKET: R2Bucket;
}

// R2Bucket interface
interface R2Bucket {
  head(key: string): Promise<R2Object | null>;
  get(key: string, options?: R2GetOptions): Promise<R2ObjectBody | R2Object | null>;
  put(key: string, value: ReadableStream | ArrayBuffer | ArrayBufferView | string | null | Blob, options?: R2PutOptions): Promise<R2Object | null>;
  delete(keys: string | string[]): Promise<void>;
  list(options?: R2ListOptions): Promise<R2Objects>;
  createMultipartUpload(key: string, options?: R2MultipartOptions): Promise<R2MultipartUpload>;
  resumeMultipartUpload(key: string, uploadId: string): R2MultipartUpload;
}

// For Wrangler types generation
// Add to wrangler.toml or wrangler.jsonc, then run: wrangler types
```

## Performance Considerations

1. **Ranged Reads**: Use for large files to reduce bandwidth/latency
2. **Conditional Operations**: Reduce latency on cache hits (304 responses)
3. **Multipart Uploads**: Required for objects >5GB, recommended for >100MB
4. **Batch Deletes**: Use array form (up to 1000 keys) instead of individual deletes
5. **Listing with Include**: May return fewer objects per request to fit metadata
6. **Strong Consistency**: All operations globally consistent immediately

## Pricing Notes

- **Storage**: Standard vs Infrequent Access pricing
- **Class A Operations**: Writes, lists (PUT, POST, LIST, COPY)
- **Class B Operations**: Reads (GET, HEAD)
- **Data Retrieval**: Only for Infrequent Access storage class
- **Egress**: Zero cost (key R2 benefit)
- **Minimum Storage Duration**: 30 days for Infrequent Access (charged even if deleted early)

## Security Best Practices

1. **Key Validation**: Always sanitize/validate object keys (prevent path traversal)
2. **Access Control**: Use R2 bucket tokens or Workers auth
3. **SSE-C**: For encryption at rest with customer keys
4. **Presigned URLs**: Use S3 SDK to generate time-limited URLs
5. **CORS**: Configure appropriately for browser access
6. **Metadata**: Don't store sensitive data in custom metadata (visible in listings)

## Troubleshooting

### Common Errors

1. **"oldString not found"**: Object key doesn't exist → check HEAD first
2. **Precondition failures**: `put()` returns `null`, `get()` returns object without body
3. **Multipart failures**: Ensure uniform part sizes, handle abort on error
4. **List truncation**: Always check `truncated` property, not object count
5. **Storage class**: Can't transition from IA back to Standard via lifecycle (must use CopyObject)

### Debugging

```typescript
// Log object metadata
const object = await env.MY_BUCKET.head(key);
console.log(JSON.stringify({
  key: object?.key,
  version: object?.version,
  size: object?.size,
  storageClass: object?.storageClass,
  uploaded: object?.uploaded,
  checksums: object?.checksums
}, null, 2));

// Check bucket binding
console.log('Bucket available:', !!env.MY_BUCKET);
```

## Migration from S3

1. **Region**: Change to `auto`
2. **Unsupported Features**: Remove ACLs, versioning, object locking, tagging
3. **Storage Classes**: Map STANDARD → Standard, STANDARD_IA → InfrequentAccess
4. **Endpoint**: Update to `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`
5. **Credentials**: Generate R2 API tokens
6. **Testing**: Verify ListObjectsV2, multipart uploads, conditional requests

## References

- **Docs**: https://developers.cloudflare.com/r2/
- **Workers API**: https://developers.cloudflare.com/r2/api/workers/
- **S3 API**: https://developers.cloudflare.com/r2/api/s3/
- **Wrangler**: https://developers.cloudflare.com/workers/wrangler/commands/#r2-bucket
- **Examples**: https://developers.cloudflare.com/r2/examples/
