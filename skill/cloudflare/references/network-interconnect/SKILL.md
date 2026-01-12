# Cloudflare Network Interconnect (CNI) Skill

Expert guidance for implementing and managing Cloudflare Network Interconnect - private, high-performance connectivity to Cloudflare's network.

## Overview

Cloudflare Network Interconnect (CNI) enables direct private connectivity between your network infrastructure and Cloudflare, bypassing the public Internet for enhanced performance and security. CNI is an **Enterprise-only** feature.

## Connection Types

### 1. Direct Interconnect
- **Physical fiber connection** between your equipment and Cloudflare hardware in shared data center
- **You manage**: Cross-connect procurement and management
- **Best for**: Customers collocated with Cloudflare requiring maximum control
- **Speeds**: 10 Gbps or 100 Gbps

### 2. Partner Interconnect
- **Virtual connection** via Cloudflare connectivity partners
- **Partners manage**: Connection logistics via SDN portal
- **Best for**: Customers not collocated or prefer managed solution
- **Partners include**: Console Connect, CoreSite, Digital Realty, Equinix Fabric, Megaport, PacketFabric, Zayo

### 3. Cloud Interconnect
- **Private connection** between cloud environments (AWS, GCP) and Cloudflare
- **Supports**: AWS Direct Connect (Dedicated), Google Cloud Interconnect
- **Best for**: Workloads in public clouds needing secure Cloudflare connectivity

## Dataplane Versions

### Dataplane v1 (Classic)
- Peering connection to Cloudflare edge data center
- **GRE tunnel support** for Magic Networking overlay
- **MTU**: 1,500 bytes ingress, 1,476 bytes egress
- **VLAN**: Single 802.1Q VLAN tag supported
- **BFD**: Supported for fast failover
- **LACP**: Supported for link aggregation
- **Optics**: 10GBASE-LR, 100GBASE-LR (single-mode fiber)

### Dataplane v2 (Beta)
- Based on Customer Connectivity Router (CCR)
- **No GRE tunneling** required - simplified routing
- **MTU**: 1,500 bytes bidirectional
- **VLAN**: Not yet supported (no 802.1Q or QinQ)
- **BFD**: Not yet supported
- **LACP**: Not yet supported (use ECMP instead)
- **Optics**: 10GBASE-LR, 100GBASE-LR4 (single-mode fiber)

## Use Cases by Dataplane

| Use Case | Dataplane v1 | Dataplane v2 (Beta) |
|----------|--------------|---------------------|
| **Magic Transit DSR** (DDoS protection, egress via ISP) | ✅ With/without GRE | ✅ Supported |
| **Magic Transit with Egress** (DDoS + egress via CF) | ✅ With/without GRE | ✅ Supported |
| **Magic WAN + Zero Trust** (Private network backbone) | ✅ Requires GRE tunnel | ✅ Supported |
| **Peering** (Exchange public routes at PoP) | ✅ AS13335 peering | ❌ Not supported |
| **App Security/Performance** (WAF, Cache, LB) | ✅ Via peering or Magic Transit | ✅ Over Magic Transit CNI v2 |

## Prerequisites

### Port Availability
- CNI offered **no charge** to Enterprise customers
- Available at select Cloudflare data centers
- Coordinate with Cloudflare account team for eligibility
- Check [available locations PDF](https://developers.cloudflare.com/network-interconnect/static/cni-locations-30-10-2025.pdf)

### Prefix Requirements
- **IPv4**: `/24` or shorter prefix length
- **IPv6**: `/48` or shorter prefix length

### BGP Requirements
- **Dataplane v1**: BGP session required for operation
- Provide BGP ASN during setup
- Optional BGP password supported

## Technical Specifications

### IP Addressing
- All CNI connections use `/31` subnet for point-to-point connectivity

### Distance Limitations
- **Maximum**: 10 km optical link distance
- For longer distances: Use intermediate hardware or third-party provider

### Performance Characteristics

| Direction | 10G Circuit | 100G Circuit |
|-----------|-------------|--------------|
| **CF → Customer** (all use cases) | Up to 10 Gbps | Up to 100 Gbps |
| **Customer → CF** (peering) | Up to 10 Gbps | Up to 100 Gbps |
| **Customer → CF** (Magic Transit/WAN) | v1: 1 Gbps/GRE tunnel<br>v2: 1 Gbps/CNI | v1: 1 Gbps/GRE tunnel<br>v2: 1 Gbps/CNI |

### Service Expectations
- **No formal SLA** (offered at no charge)
- **No dashboard visibility** of interconnect config/status
- **Recovery time**: Could be several days in some locations
- **Required**: Maintain backup Internet connectivity
- **Your responsibility**: Capacity planning across available links

## API Reference

### Base Endpoints

```
Base URL: https://api.cloudflare.com/client/v4
Authentication: Bearer token in X-Auth-Email and X-Auth-Key headers
                OR Authorization: Bearer <api_token>
```

### Interconnects API

#### List Interconnects
```http
GET /accounts/{account_id}/cni/interconnects
```

**Query Parameters:**
- `page` - Page number for pagination
- `per_page` - Items per page

**Response:**
```json
{
  "items": [
    {
      "account": "account_id",
      "facility": "EWR1",
      "name": "my-interconnect",
      "speed": "10G",
      "type": "direct",
      "status": "active"
    }
  ],
  "next": "next_page_token"
}
```

#### Create Interconnect
```http
POST /accounts/{account_id}/cni/interconnects
```

**Request Body:**
```json
{
  "account": "account_id",
  "slot_id": "slot_identifier",
  "type": "direct",
  "facility": "EWR1",
  "speed": "10G",
  "name": "my-interconnect",
  "description": "Production interconnect"
}
```

#### Get Interconnect Details
```http
GET /accounts/{account_id}/cni/interconnects/{icon}
```

#### Get Interconnect Status
```http
GET /accounts/{account_id}/cni/interconnects/{icon}/status
```

**Response:**
```json
{
  "state": "active"
}
```

**Possible States:**
- `active` - Link operational state up, port sees sufficient light
- `unhealthy` - Link down, connectivity issues
- `pending` - Not yet active, awaiting cross-connect or setup

#### Generate LOA (Letter of Authorization)
```http
GET /accounts/{account_id}/cni/interconnects/{icon}/loa
```

Returns PDF document for ordering physical cross-connect.

#### Delete Interconnect
```http
DELETE /accounts/{account_id}/cni/interconnects/{icon}
```

### CNI Objects API

CNI objects represent the BGP configuration for your interconnect.

#### List CNI Objects
```http
GET /accounts/{account_id}/cni/cnis
```

#### Create CNI Object
```http
POST /accounts/{account_id}/cni/cnis
```

**Request Body:**
```json
{
  "account": "account_id",
  "cust_ip": "192.0.2.1/31",
  "cf_ip": "192.0.2.0/31",
  "bgp_asn": 65000,
  "bgp_password": "optional_password",
  "vlan": 100
}
```

#### Get CNI Object
```http
GET /accounts/{account_id}/cni/cnis/{cni}
```

#### Update CNI Object
```http
PUT /accounts/{account_id}/cni/cnis/{cni}
```

#### Delete CNI Object
```http
DELETE /accounts/{account_id}/cni/cnis/{cni}
```

### Slots API

Query available physical ports/slots at Cloudflare locations.

#### List Available Slots
```http
GET /accounts/{account_id}/cni/slots
```

**Query Parameters:**
- `facility` - Filter by facility code
- `occupied` - Filter by availability (true/false)

#### Get Slot Details
```http
GET /accounts/{account_id}/cni/slots/{slot}
```

**Response:**
```json
{
  "id": "slot_id",
  "facility": "EWR1",
  "occupied": false,
  "speed": "10G",
  "port_type": "10GBASE-LR",
  "dataplane_version": "v2"
}
```

### Settings API

#### Get Account Settings
```http
GET /accounts/{account_id}/cni/settings
```

**Response:**
```json
{
  "default_asn": 65000
}
```

#### Update Account Settings
```http
PUT /accounts/{account_id}/cni/settings
```

**Request Body:**
```json
{
  "default_asn": 65001
}
```

## Code Examples

### TypeScript (Official SDK)

```typescript
import Cloudflare from 'cloudflare';

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

// List all interconnects
async function listInterconnects(accountId: string) {
  const response = await client.networkInterconnects.interconnects.list({
    account_id: accountId,
  });
  
  console.log('Interconnects:', response.items);
  return response;
}

// Create a new interconnect
async function createInterconnect(accountId: string) {
  const interconnect = await client.networkInterconnects.interconnects.create({
    account_id: accountId,
    account: accountId,
    slot_id: 'slot_abc123',
    type: 'direct',
    facility: 'EWR1',
    speed: '10G',
    name: 'prod-interconnect',
    description: 'Production datacenter connection',
  });
  
  console.log('Created:', interconnect);
  return interconnect;
}

// Get interconnect status
async function getStatus(accountId: string, iconId: string) {
  const status = await client.networkInterconnects.interconnects.get(
    accountId,
    iconId
  );
  
  console.log('Status:', status);
  return status;
}

// Download LOA for cross-connect
async function downloadLOA(accountId: string, iconId: string) {
  // Note: SDK method may return PDF as buffer
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/cni/interconnects/${iconId}/loa`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.CLOUDFLARE_API_TOKEN}`,
      },
    }
  );
  
  const buffer = await response.arrayBuffer();
  // Save buffer to file
  await fs.writeFile('loa.pdf', Buffer.from(buffer));
  console.log('LOA saved to loa.pdf');
}

// Create CNI object (BGP configuration)
async function createCNI(accountId: string) {
  const cni = await client.networkInterconnects.cnis.create({
    account_id: accountId,
    account: accountId,
    cust_ip: '192.0.2.1/31',
    cf_ip: '192.0.2.0/31',
    bgp_asn: 65000,
    bgp_password: 'secure_password',
    vlan: 100,
  });
  
  console.log('CNI created:', cni);
  return cni;
}

// List available slots
async function listSlots(accountId: string, facility?: string) {
  const slots = await client.networkInterconnects.slots.list({
    account_id: accountId,
    facility,
    occupied: false, // Only show available slots
  });
  
  console.log('Available slots:', slots.items);
  return slots;
}
```

### Python (Official SDK)

```python
from cloudflare import Cloudflare
import os

client = Cloudflare(
    api_token=os.environ.get("CLOUDFLARE_API_TOKEN"),
)

# List all interconnects
def list_interconnects(account_id: str):
    response = client.network_interconnects.interconnects.list(
        account_id=account_id
    )
    
    print(f"Interconnects: {response.items}")
    return response

# Create a new interconnect
def create_interconnect(account_id: str):
    interconnect = client.network_interconnects.interconnects.create(
        account_id=account_id,
        account=account_id,
        slot_id="slot_abc123",
        type="direct",
        facility="EWR1",
        speed="10G",
        name="prod-interconnect",
        description="Production datacenter connection",
    )
    
    print(f"Created: {interconnect}")
    return interconnect

# Get interconnect status
def get_status(account_id: str, icon_id: str):
    status = client.network_interconnects.interconnects.get(
        account_id=account_id,
        icon=icon_id
    )
    
    print(f"Status: {status}")
    return status

# Download LOA
def download_loa(account_id: str, icon_id: str):
    import requests
    
    response = requests.get(
        f"https://api.cloudflare.com/client/v4/accounts/{account_id}/cni/interconnects/{icon_id}/loa",
        headers={
            "Authorization": f"Bearer {os.environ.get('CLOUDFLARE_API_TOKEN')}",
        }
    )
    
    with open("loa.pdf", "wb") as f:
        f.write(response.content)
    print("LOA saved to loa.pdf")

# Create CNI object
def create_cni(account_id: str):
    cni = client.network_interconnects.cnis.create(
        account_id=account_id,
        account=account_id,
        cust_ip="192.0.2.1/31",
        cf_ip="192.0.2.0/31",
        bgp_asn=65000,
        bgp_password="secure_password",
        vlan=100,
    )
    
    print(f"CNI created: {cni}")
    return cni

# List available slots
def list_slots(account_id: str, facility: str = None):
    slots = client.network_interconnects.slots.list(
        account_id=account_id,
        facility=facility,
        occupied=False,  # Only available slots
    )
    
    print(f"Available slots: {slots.items}")
    return slots
```

### cURL Examples

```bash
# Set environment variables
export CF_API_TOKEN="your_api_token"
export ACCOUNT_ID="your_account_id"

# List interconnects
curl -X GET "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/cni/interconnects" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json"

# Create interconnect
curl -X POST "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/cni/interconnects" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "'${ACCOUNT_ID}'",
    "slot_id": "slot_abc123",
    "type": "direct",
    "facility": "EWR1",
    "speed": "10G",
    "name": "prod-interconnect"
  }'

# Get interconnect status
curl -X GET "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/cni/interconnects/${ICON_ID}/status" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json"

# Download LOA
curl -X GET "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/cni/interconnects/${ICON_ID}/loa" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  --output loa.pdf

# Create CNI object
curl -X POST "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/cni/cnis" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "'${ACCOUNT_ID}'",
    "cust_ip": "192.0.2.1/31",
    "cf_ip": "192.0.2.0/31",
    "bgp_asn": 65000,
    "vlan": 100
  }'

# List available slots
curl -X GET "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/cni/slots?occupied=false" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json"
```

## Implementation Workflow

### Timeline: 2-4 Weeks
Most common delays occur during physical connection phase (outside Cloudflare control).

### Steps

#### 1. Submit Request (Week 1)
- Contact account team with CNI request ticket
- Provide: CNI type, location, use case, technical details
- Implementation Manager assigned

#### 2. Review Configuration (Week 1-2, v1 only)
- Implementation Manager provides config document (IP addressing, VLANs, specs)
- Review and approve document
- **v2 dataplane**: This step not necessary

#### 3. Order Connection (Week 2-3)
**Direct Interconnect:**
- Receive Letter of Authorization (LOA) from Cloudflare
- Order physical cross-connect from data center facility operator
- Provide LOA to facility

**Partner Interconnect:**
- Use provided details to order virtual circuit from partner portal

**Cloud Interconnect:**
- Order Direct Connect (AWS) or Cloud Interconnect (GCP)
- Provide LOA and VLAN ID to Cloudflare account team

#### 4. Configure Network (Week 3)
- Cloudflare configures their network devices
- Your team configures your network devices
- Follow approved configuration document

#### 5. Test and Verify (Week 3-4)
- Perform basic connectivity tests (ping)
- Verify BGP session establishment (if applicable)
- Confirm routing table entries

#### 6. Enable Health Checks (Week 4)
Configure tunnel health checks for:
- [Magic Transit](https://developers.cloudflare.com/magic-transit/how-to/configure-tunnel-endpoints/#add-tunnels)
- [Magic WAN](https://developers.cloudflare.com/magic-wan/configuration/manually/how-to/configure-tunnel-endpoints/#add-tunnels)

#### 7. Activate Services (Week 4)
- Configure Cloudflare products (Magic Transit, Magic WAN, etc.)
- Route traffic over CNI
- Implementation Manager verifies end-to-end traffic flow
- Mark deployment complete

#### 8. Setup Monitoring
- [Enable maintenance notifications](https://developers.cloudflare.com/network-interconnect/monitoring-and-alerts/#enable-cloudflare-status-maintenance-notification)
- Subscribe to CNI Connection Maintenance Alert (beta)
- Subscribe to Cloudflare Status Maintenance Notification

## Cloud Interconnect Setup

### AWS Direct Connect (Beta)

**Requirements:**
- Magic WAN customer
- AWS Dedicated Direct Connect (1 Gbps or 10 Gbps)
- Hosted Direct Connect not yet supported

**Setup Process:**
1. Contact account team to start provisioning
2. Choose interconnect location from available options
3. Order Direct Connect in AWS portal
4. AWS provides LOA and VLAN ID
5. Send LOA and VLAN ID to account team
6. Wait ~4 weeks for provisioning

### Google Cloud Interconnect (Beta)

**Setup in Cloudflare Dashboard:**
```
1. Dashboard → Interconnects → Create new
2. Cloud Interconnect → Create new
3. Google Integration → Select integration
4. Provide:
   - Name and description
   - MTU (must match GCP VLAN attachment)
5. Select interface speed (affects GCP billing)
6. Enter VLAN attachment pairing key
7. Review and confirm order
```

**Post-Setup Routing:**

**Route traffic to GCP:**
- Add [static routes](https://developers.cloudflare.com/magic-wan/configuration/manually/how-to/configure-routes/#configure-static-routes) to Magic WAN routing table
- Enable [legacy bidirectional tunnel health checks](https://developers.cloudflare.com/magic-wan/configuration/manually/how-to/configure-tunnel-endpoints/#legacy-bidirectional-health-checks)
- **Note**: BGP routes advertised from GCP Cloud Router are ignored

**Route traffic to Cloudflare:**
- Add [custom learned routes to Cloud Router](https://cloud.google.com/network-connectivity/docs/router/how-to/configure-custom-learned-routes)
- Use BGP session
- Contact account team to request prefixes to advertise
- Specify interconnect ID for advertisement

## Monitoring and Alerts

### Dashboard Status Indicators

**Active:**
- Link operational state UP at CCR port
- Port sees sufficient light levels
- Ethernet link negotiated

**Unhealthy:**
- Link operational state DOWN
- Possible causes:
  - No light detected
  - Cannot negotiate Ethernet signal
  - Light levels below -20 dBm
- **Action**: Check cables, status lights, contact account team if unresolved

**Pending:**
- Link not yet active
- Expected during initial setup
- Possible reasons:
  - Cross-connect not received
  - Device unresponsive
  - RX/TX fibers need swapping
- **Action**: Complete cross-connect installation

### Alert Types

#### CNI Connection Maintenance Alert (Beta)
**Scope**: Magic Networking overlay CNI circuits only

**Features:**
- Warnings up to 2 weeks advance
- Notifications for scheduled, updated, or canceled events
- 6-hour delay for recent additions (prevents alert fatigue)

**Setup:**
```
Dashboard → Notifications → Add
Product: Cloudflare Network Interconnect
Type: Connection Maintenance Alert
Configure delivery method (email, webhook, etc.)
```

#### Cloudflare Status Maintenance Notification
**Scope**: Entire Cloudflare PoP (affects all CNI services)

**Features:**
- Warnings for potentially disruptive PoP maintenance
- Filter by event type (Scheduled, Changed, Canceled)
- Filter by specific PoP locations

**Setup:**
```
Dashboard → Notifications → Add
Product: Cloudflare Status
Type: Maintenance Notification
Filter on PoPs: Enter 3-letter codes (e.g., "gru,fra,lhr")
```

**Finding Your PoP Code:**
```
Dashboard → Magic Transit/WAN → Configuration → Interconnects
Select CNI → Note Data Center code (e.g., "gru-b")
Use first 3 letters for PoP code (e.g., "gru")
```

## High Availability Design

### Critical Considerations

**Single Point of Failure Risk:**
- Design for resilience from day one
- Seek locations with device-level diversity
- Ensures connections terminate on physically separate hardware

**Backup Connectivity:**
- **Required**: Alternative Internet connectivity for all CNI implementations
- CNI has no formal SLA
- Recovery could take several days in some locations

**Network-Resilient Locations:**
- Maintain connectivity during maintenance
- Single-homed locations can experience full disruption

### Recommended Architecture

```
┌─────────────────┐     ┌─────────────────┐
│   Your Network  │     │   Your Network  │
│   Location A    │     │   Location B    │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │ CNI v2                │ CNI v2
         │ 10G/100G              │ 10G/100G
         │                       │
┌────────▼────────┐     ┌────────▼────────┐
│  Cloudflare     │     │  Cloudflare     │
│  CCR Device 1   │     │  CCR Device 2   │
│  (Different     │     │  (Different     │
│   Hardware)     │     │   Hardware)     │
└─────────────────┘     └─────────────────┘
         │                       │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   Cloudflare Global   │
         │   Network (AS13335)   │
         └───────────────────────┘
```

### Capacity Planning
- **Your responsibility**: Plan capacity across all links
- Consider topology and link sizes
- Account for failover scenarios
- Test failover procedures regularly

## Common Patterns

### Pattern 1: Magic Transit with CNI v2 (Recommended)

**Use Case**: DDoS protection with direct private connectivity

**Architecture:**
```typescript
// 1. Create interconnect
const interconnect = await client.networkInterconnects.interconnects.create({
  account_id: accountId,
  type: 'direct',
  facility: 'EWR1',
  speed: '10G',
  name: 'magic-transit-primary',
});

// 2. Wait for active status
const status = await pollUntilActive(accountId, interconnect.id);

// 3. Configure Magic Transit tunnel
// Use Magic Transit API or Dashboard to configure tunnels
// pointing to CNI connection
```

**Benefits:**
- No GRE overhead (1,500 MTU bidirectional)
- Simplified routing
- Direct path to Cloudflare

### Pattern 2: Multi-Cloud with Cloud Interconnect

**Use Case**: Hybrid cloud architecture with AWS/GCP workloads

**Architecture:**
```typescript
// AWS Direct Connect
async function setupAWSInterconnect(accountId: string) {
  // 1. Order AWS Direct Connect in AWS Console
  // 2. Get LOA and VLAN ID from AWS
  
  // 3. Provide to Cloudflare (via account team)
  // No direct API call - coordinate with account team
  
  // 4. After provisioning, configure routing in Magic WAN
  await configureStaticRoutes(accountId, {
    prefix: '10.0.0.0/8',
    nexthop: 'aws-direct-connect',
  });
}

// GCP Cloud Interconnect
async function setupGCPInterconnect(accountId: string) {
  // 1. Create in Cloudflare Dashboard
  // 2. Get VLAN attachment pairing key from GCP
  
  const interconnect = await client.networkInterconnects.interconnects.create({
    account_id: accountId,
    type: 'cloud',
    cloud_provider: 'gcp',
    pairing_key: 'gcp_pairing_key_from_console',
    name: 'gcp-interconnect',
  });
  
  // 3. Configure routes in Magic WAN
  // 4. Configure BGP in GCP Cloud Router
}
```

### Pattern 3: High-Availability Multi-Location

**Use Case**: Enterprise requiring 99.99%+ uptime

**Architecture:**
```typescript
async function setupHAInterconnect(accountId: string) {
  // Primary location
  const primary = await client.networkInterconnects.interconnects.create({
    account_id: accountId,
    type: 'direct',
    facility: 'EWR1', // New York
    speed: '10G',
    name: 'primary-ewr1',
  });
  
  // Secondary location (different device)
  const secondary = await client.networkInterconnects.interconnects.create({
    account_id: accountId,
    type: 'direct',
    facility: 'EWR2', // Different hardware in same metro
    speed: '10G',
    name: 'secondary-ewr2',
  });
  
  // Tertiary location (different geography)
  const tertiary = await client.networkInterconnects.interconnects.create({
    account_id: accountId,
    type: 'partner',
    facility: 'LAX1', // Los Angeles
    speed: '10G',
    name: 'tertiary-lax1',
  });
  
  // Configure BGP with route priorities
  // Primary: Local pref 200
  // Secondary: Local pref 150
  // Tertiary: Local pref 100
  
  // Internet backup remains as last resort
}
```

### Pattern 4: Partner Interconnect with Equinix

**Use Case**: Quick deployment without physical colocation

**Setup:**
```
1. Order virtual circuit in Equinix Fabric Portal
2. Select Cloudflare as destination
3. Choose Cloudflare facility/location
4. Provide connection details to Cloudflare account team
5. Cloudflare accepts connection in portal
6. Configure BGP once connection active
```

**No API automation** - Partner portals are managed separately

## Best Practices

### 1. Design Phase
- ✅ Choose locations with device-level diversity
- ✅ Plan for redundancy from day one
- ✅ Select appropriate dataplane version for use case
- ✅ Consider MTU requirements (v1 vs v2)
- ✅ Document capacity requirements and growth projections

### 2. Ordering Phase
- ✅ Verify facility codes match between LOA and data center
- ✅ Double-check fiber types (single-mode for CNI)
- ✅ Confirm optic types (10GBASE-LR or 100GBASE-LR)
- ✅ Keep LOA documents for facility provider
- ✅ Track cross-connect order separately

### 3. Configuration Phase
- ✅ Use /31 subnets for point-to-point links
- ✅ Configure BGP passwords for security
- �� Set appropriate BGP timers
- ✅ Configure BFD if using dataplane v1
- ✅ Test with simple ping before BGP

### 4. Production Phase
- ✅ Enable maintenance notifications immediately
- ✅ Monitor interconnect status programmatically
- ✅ Maintain runbooks for failover procedures
- ✅ Test backup connectivity regularly
- ✅ Document escalation paths to account team

### 5. Security
- ✅ Use API tokens with minimum required permissions
- ✅ Rotate BGP passwords periodically
- ✅ Implement BGP route filtering
- ✅ Monitor for unexpected route advertisements
- ✅ Use Magic Firewall for additional protection

## Troubleshooting

### Status: Pending
**Symptom**: Interconnect stuck in pending state

**Causes:**
- Cross-connect not installed
- RX/TX fibers reversed
- Incorrect fiber type
- Light levels too low

**Resolution:**
1. Verify cross-connect installed at facility
2. Check fiber connection at patch panel
3. Try swapping RX/TX fibers
4. Use optical power meter to check light levels
5. Contact account team if unresolved

### Status: Unhealthy
**Symptom**: Interconnect shows unhealthy status

**Causes:**
- Physical connectivity issue
- Light levels below -20 dBm
- Optic mismatch
- Dirty fiber connectors

**Resolution:**
1. Check physical connections
2. Clean fiber connectors with appropriate tools
3. Verify optic types match specifications
4. Test with known-good optics if available
5. Check patch panel connections
6. Contact account team for Cloudflare-side diagnostics

### BGP Session Not Establishing
**Symptom**: Physical link up but BGP session down

**Causes:**
- Incorrect IP addressing
- Wrong ASN configuration
- BGP password mismatch
- Firewall blocking TCP/179

**Resolution:**
1. Verify IP addresses match CNI object configuration
2. Confirm ASN matches what was provided to Cloudflare
3. Check BGP password if configured
4. Verify no firewall blocking TCP port 179
5. Check BGP logs for specific error messages
6. Review BGP timers (keepalive, hold time)

### Low Throughput
**Symptom**: Traffic flowing but below expected rates

**Causes:**
- MTU mismatch
- Fragmentation occurring
- Single GRE tunnel limiting throughput (v1)
- Routing inefficiencies

**Resolution:**
1. Check MTU configuration (1,500 ingress, 1,476 egress for v1)
2. Test with various packet sizes
3. Add additional GRE tunnels for more bandwidth (v1)
4. Consider upgrading to v2 dataplane for bidirectional 1,500 MTU
5. Review routing tables for suboptimal paths
6. Use LACP to bundle multiple ports (v1 only)

## Limitations

### General
- ❌ No formal SLA (offered at no charge)
- ❌ No dashboard visibility of config details
- ❌ Recovery time may be several days
- ❌ 10 km maximum optical distance
- ❌ Backup Internet connectivity required

### Dataplane v1 Specific
- ❌ Asymmetric MTU (1,500 ingress, 1,476 egress)
- ❌ GRE tunnel overhead for Magic Transit/WAN
- ❌ 1 Gbps per GRE tunnel throughput limit

### Dataplane v2 Specific
- ❌ No VLAN tagging support (yet)
- ❌ No BFD support (yet)
- ❌ No LACP support (use ECMP)
- ❌ No peering use case support
- ❌ Currently in beta

### Cloud Interconnect
- ❌ AWS Hosted Direct Connect not supported
- ❌ Only available for Magic WAN customers
- ❌ BGP routes from GCP Cloud Router ignored
- ❌ Requires manual coordination with account team (AWS)

## Related Cloudflare Products

### Magic Transit
- DDoS protection and traffic acceleration
- Works with CNI for private connectivity
- [Documentation](https://developers.cloudflare.com/magic-transit/)

### Magic WAN
- Corporate network security and performance
- Zero Trust integration
- [Documentation](https://developers.cloudflare.com/magic-wan/)

### Argo Smart Routing
- Intelligent traffic routing
- Can utilize CNI for optimal paths
- [Documentation](https://developers.cloudflare.com/argo/)

## Resources

### Documentation
- [CNI Overview](https://developers.cloudflare.com/network-interconnect/)
- [Get Started Guide](https://developers.cloudflare.com/network-interconnect/get-started/)
- [Monitoring Guide](https://developers.cloudflare.com/network-interconnect/monitoring-and-alerts/)
- [Available Locations PDF](https://developers.cloudflare.com/network-interconnect/static/cni-locations-30-10-2025.pdf)

### API Documentation
- [Network Interconnects API](https://developers.cloudflare.com/api/resources/network_interconnects/)
- [CNIs API](https://developers.cloudflare.com/api/resources/network_interconnects/subresources/cnis/)
- [Interconnects API](https://developers.cloudflare.com/api/resources/network_interconnects/subresources/interconnects/)
- [Slots API](https://developers.cloudflare.com/api/resources/network_interconnects/subresources/slots/)

### SDKs
- [TypeScript SDK](https://github.com/cloudflare/cloudflare-typescript)
- [Python SDK](https://github.com/cloudflare/cloudflare-python)

### Support
- Contact your Cloudflare account team for:
  - CNI eligibility and provisioning
  - Technical configuration assistance
  - LOA generation and cross-connect issues
  - Troubleshooting unhealthy interconnects
  - Location availability and diversity options

## Quick Reference

### Key Commands

```bash
# List interconnects
cf-api GET /accounts/{account_id}/cni/interconnects

# Get status
cf-api GET /accounts/{account_id}/cni/interconnects/{icon}/status

# Download LOA
cf-api GET /accounts/{account_id}/cni/interconnects/{icon}/loa

# List CNI objects
cf-api GET /accounts/{account_id}/cni/cnis

# List available slots
cf-api GET /accounts/{account_id}/cni/slots?occupied=false
```

### Connection Decision Matrix

| Requirement | Recommended Type |
|-------------|-----------------|
| Collocated with Cloudflare | Direct Interconnect |
| Not collocated | Partner Interconnect |
| AWS/GCP workloads | Cloud Interconnect |
| Need 1,500 MTU bidirectional | Dataplane v2 |
| Need VLAN tagging | Dataplane v1 |
| Need public peering | Dataplane v1 |
| Need simplest config | Dataplane v2 |
| Need BFD fast failover | Dataplane v1 |
| Need LACP link bundling | Dataplane v1 |

### Status Quick Guide

| Status | Meaning | Action |
|--------|---------|--------|
| Active | Working normally | Monitor regularly |
| Unhealthy | Connection down | Check physical connectivity |
| Pending | Setup in progress | Complete cross-connect |
