# Cloudflare Terraform Provider Skill

**Expert guidance for Cloudflare Terraform Provider - infrastructure as code for Cloudflare resources.**

## Core Principles

- **Provider-first**: Use Terraform provider for ALL infrastructure - never mix with wrangler.toml for the same resources
- **State management**: Always use remote state (S3, Terraform Cloud, etc.) for team environments
- **Modular architecture**: Create reusable modules for common patterns (zones, workers, pages)
- **Version pinning**: Always pin provider version with `~>` for predictable upgrades
- **Secret management**: Use variables + environment vars for sensitive data - never hardcode API tokens

## Provider Setup

### Basic Configuration

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 5.15.0"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token  # or CLOUDFLARE_API_TOKEN env var
}
```

### Authentication Methods (priority order)

1. **API Token** (RECOMMENDED): `api_token` or `CLOUDFLARE_API_TOKEN`
   - Create: Dashboard → My Profile → API Tokens
   - Scope to specific accounts/zones for security
   
2. **Global API Key** (LEGACY): `api_key` + `api_email` or `CLOUDFLARE_API_KEY` + `CLOUDFLARE_EMAIL`
   - Less secure, use tokens instead
   
3. **User Service Key**: `user_service_key` for Origin CA certificates

### Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "cloudflare/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Core Resource Patterns

### 1. Zone Management

```hcl
# Create a zone
resource "cloudflare_zone" "example" {
  account = {
    id = var.account_id
  }
  name = "example.com"
  type = "full"  # full, partial, secondary
}

# Data source for existing zone
data "cloudflare_zone" "existing" {
  name = "example.com"
}

# Zone settings
resource "cloudflare_zone_settings_override" "example" {
  zone_id = cloudflare_zone.example.id
  
  settings {
    ssl                      = "strict"
    automatic_https_rewrites = "on"
    always_use_https         = "on"
    min_tls_version          = "1.2"
    tls_1_3                  = "on"
    http3                    = "on"
    zero_rtt                 = "on"
    brotli                   = "on"
  }
}
```

### 2. DNS Records

```hcl
# A record with proxy
resource "cloudflare_dns_record" "www" {
  zone_id = cloudflare_zone.example.id
  name    = "www"
  content = "192.0.2.1"
  type    = "A"
  ttl     = 1      # Auto when proxied = true
  proxied = true
}

# CNAME record
resource "cloudflare_dns_record" "api" {
  zone_id = cloudflare_zone.example.id
  name    = "api"
  content = "api.example.workers.dev"
  type    = "CNAME"
  proxied = true
}

# MX records
resource "cloudflare_dns_record" "mx" {
  for_each = {
    "10" = "mail1.example.com"
    "20" = "mail2.example.com"
  }
  
  zone_id  = cloudflare_zone.example.id
  name     = "@"
  content  = each.value
  type     = "MX"
  priority = each.key
  ttl      = 3600
}

# TXT record
resource "cloudflare_dns_record" "spf" {
  zone_id = cloudflare_zone.example.id
  name    = "@"
  content = "v=spf1 include:_spf.example.com ~all"
  type    = "TXT"
}
```

### 3. Workers

```hcl
# Worker script
resource "cloudflare_worker_script" "api_worker" {
  account_id         = var.account_id
  name               = "api-worker"
  content            = file("${path.module}/workers/api.js")
  module             = true  # For ES modules
  compatibility_date = "2024-01-01"
  
  # KV namespace binding
  kv_namespace_binding {
    name         = "KV"
    namespace_id = cloudflare_workers_kv_namespace.api_kv.id
  }
  
  # R2 bucket binding
  r2_bucket_binding {
    name        = "BUCKET"
    bucket_name = cloudflare_r2_bucket.assets.name
  }
  
  # D1 database binding
  d1_database_binding {
    name        = "DB"
    database_id = cloudflare_d1_database.api_db.id
  }
  
  # Service binding
  service_binding {
    name        = "AUTH"
    service     = "auth-worker"
    environment = "production"
  }
  
  # Plain text binding (secrets via env vars in real usage)
  plain_text_binding {
    name = "API_KEY"
    text = var.api_key
  }
  
  # Secret text binding
  secret_text_binding {
    name = "SECRET"
    text = var.secret_value
  }
}

# Worker route (zone-based)
resource "cloudflare_worker_route" "api_route" {
  zone_id     = cloudflare_zone.example.id
  pattern     = "api.example.com/*"
  script_name = cloudflare_worker_script.api_worker.name
}

# Worker domain (custom domain)
resource "cloudflare_worker_domain" "api_domain" {
  account_id = var.account_id
  hostname   = "api.example.com"
  service    = cloudflare_worker_script.api_worker.name
  zone_id    = cloudflare_zone.example.id
}

# Worker cron trigger
resource "cloudflare_worker_cron_trigger" "scheduled_task" {
  account_id  = var.account_id
  script_name = cloudflare_worker_script.api_worker.name
  schedules   = [
    "*/5 * * * *",     # Every 5 minutes
    "0 0 * * *",       # Daily at midnight
  ]
}
```

### 4. Workers KV

```hcl
# KV namespace
resource "cloudflare_workers_kv_namespace" "cache" {
  account_id = var.account_id
  title      = "api-cache"
}

# KV key-value pair
resource "cloudflare_workers_kv" "config" {
  account_id   = var.account_id
  namespace_id = cloudflare_workers_kv_namespace.cache.id
  key_name     = "config"
  value        = jsonencode({
    version = "1.0"
    enabled = true
  })
}

# Multiple KV pairs with for_each
resource "cloudflare_workers_kv" "configs" {
  for_each = {
    "prod"    = jsonencode({ env = "production" })
    "staging" = jsonencode({ env = "staging" })
  }
  
  account_id   = var.account_id
  namespace_id = cloudflare_workers_kv_namespace.cache.id
  key_name     = each.key
  value        = each.value
}
```

### 5. R2 Storage

```hcl
# R2 bucket
resource "cloudflare_r2_bucket" "assets" {
  account_id = var.account_id
  name       = "assets-bucket"
  location   = "WNAM"  # Western North America (optional)
}

# R2 bucket with lifecycle rules (custom configuration)
resource "cloudflare_r2_bucket" "backups" {
  account_id = var.account_id
  name       = "backups-bucket"
  location   = "ENAM"
}
```

### 6. D1 Databases

```hcl
# D1 database
resource "cloudflare_d1_database" "app_db" {
  account_id = var.account_id
  name       = "app-database"
}

# Note: Schema migrations should be done via wrangler or API
# Terraform manages the database resource, not its schema
```

### 7. Pages Projects

```hcl
# Pages project
resource "cloudflare_pages_project" "docs" {
  account_id        = var.account_id
  name              = "docs-site"
  production_branch = "main"
  
  deployment_configs {
    production {
      compatibility_date        = "2024-01-01"
      compatibility_flags       = ["nodejs_compat"]
      environment_variables = {
        NODE_ENV = "production"
      }
      
      kv_namespaces = {
        KV = cloudflare_workers_kv_namespace.cache.id
      }
      
      d1_databases = {
        DB = cloudflare_d1_database.app_db.id
      }
      
      r2_buckets = {
        ASSETS = cloudflare_r2_bucket.assets.name
      }
      
      service_bindings = {
        AUTH = {
          service     = "auth-worker"
          environment = "production"
        }
      }
    }
    
    preview {
      compatibility_date = "2024-01-01"
      environment_variables = {
        NODE_ENV = "development"
      }
    }
  }
  
  build_config {
    build_command   = "npm run build"
    destination_dir = "dist"
    root_dir        = "/"
  }
  
  source {
    type = "github"
    config {
      owner                         = "myorg"
      repo_name                     = "docs"
      production_branch             = "main"
      deployments_enabled           = true
      production_deployment_enabled = true
      preview_deployment_setting    = "all"
    }
  }
}

# Custom domain for Pages
resource "cloudflare_pages_domain" "docs_domain" {
  account_id   = var.account_id
  project_name = cloudflare_pages_project.docs.name
  domain       = "docs.example.com"
}

# DNS record for Pages custom domain
resource "cloudflare_dns_record" "docs" {
  zone_id = cloudflare_zone.example.id
  name    = "docs"
  content = cloudflare_pages_project.docs.subdomain
  type    = "CNAME"
  ttl     = 1
  proxied = true
}
```

### 8. Rulesets (WAF, Transforms, Redirects)

```hcl
# WAF Custom Rules
resource "cloudflare_ruleset" "waf_custom" {
  zone_id     = cloudflare_zone.example.id
  name        = "Custom WAF Rules"
  description = "Block malicious traffic"
  kind        = "zone"
  phase       = "http_request_firewall_custom"
  
  rules {
    action      = "block"
    description = "Block bad bots"
    enabled     = true
    expression  = "(cf.client.bot) and not (cf.verified_bot)"
  }
  
  rules {
    action      = "challenge"
    description = "Challenge suspicious requests"
    enabled     = true
    expression  = "(http.request.uri.path contains \"/admin\")"
  }
}

# HTTP Redirects
resource "cloudflare_ruleset" "redirects" {
  zone_id = cloudflare_zone.example.id
  name    = "Dynamic Redirects"
  kind    = "zone"
  phase   = "http_request_dynamic_redirect"
  
  rules {
    action      = "redirect"
    description = "Redirect old paths"
    enabled     = true
    expression  = "(http.request.uri.path eq \"/old-path\")"
    
    action_parameters {
      from_value {
        status_code = 301
        target_url {
          value = "https://example.com/new-path"
        }
      }
    }
  }
}

# Transform Rules (Header modification)
resource "cloudflare_ruleset" "transforms" {
  zone_id = cloudflare_zone.example.id
  name    = "Request Headers"
  kind    = "zone"
  phase   = "http_request_transform"
  
  rules {
    action      = "rewrite"
    description = "Add custom header"
    enabled     = true
    expression  = "true"
    
    action_parameters {
      headers {
        name      = "X-Custom-Header"
        operation = "set"
        value     = "custom-value"
      }
    }
  }
}

# Cache Rules
resource "cloudflare_ruleset" "cache_rules" {
  zone_id = cloudflare_zone.example.id
  name    = "Cache Configuration"
  kind    = "zone"
  phase   = "http_request_cache_settings"
  
  rules {
    action      = "set_cache_settings"
    description = "Cache static assets"
    enabled     = true
    expression  = "(http.request.uri.path matches \"\\.(jpg|png|gif|css|js)$\")"
    
    action_parameters {
      cache = true
      edge_ttl {
        mode    = "override_origin"
        default = 86400  # 1 day
      }
      browser_ttl {
        mode = "respect_origin"
      }
    }
  }
}
```

### 9. Load Balancers

```hcl
# Load balancer monitor
resource "cloudflare_load_balancer_monitor" "http_monitor" {
  account_id     = var.account_id
  type           = "http"
  expected_codes = "200"
  method         = "GET"
  path           = "/health"
  interval       = 60
  timeout        = 5
  retries        = 2
  description    = "HTTP health check"
}

# Load balancer pool
resource "cloudflare_load_balancer_pool" "api_pool" {
  account_id = var.account_id
  name       = "api-pool"
  
  origins {
    name    = "api-1"
    address = "192.0.2.1"
    enabled = true
  }
  
  origins {
    name    = "api-2"
    address = "192.0.2.2"
    enabled = true
  }
  
  monitor          = cloudflare_load_balancer_monitor.http_monitor.id
  notification_email = "ops@example.com"
}

# Load balancer
resource "cloudflare_load_balancer" "api_lb" {
  zone_id          = cloudflare_zone.example.id
  name             = "api.example.com"
  default_pool_ids = [cloudflare_load_balancer_pool.api_pool.id]
  fallback_pool_id = cloudflare_load_balancer_pool.api_pool.id
  proxied          = true
  
  steering_policy = "random"  # random, dynamic_latency, geo
  
  session_affinity = "cookie"
  session_affinity_ttl = 3600
}
```

### 10. Access (Zero Trust)

```hcl
# Access application
resource "cloudflare_access_application" "admin" {
  account_id = var.account_id
  name       = "Admin Dashboard"
  domain     = "admin.example.com"
  type       = "self_hosted"
  
  session_duration             = "24h"
  auto_redirect_to_identity    = true
  allowed_idps                 = [cloudflare_access_identity_provider.github.id]
  app_launcher_visible         = true
}

# Access policy
resource "cloudflare_access_policy" "admin_policy" {
  account_id     = var.account_id
  application_id = cloudflare_access_application.admin.id
  name           = "Allow engineers"
  precedence     = 1
  decision       = "allow"
  
  include {
    email = ["engineer@example.com"]
  }
  
  require {
    email_domain = ["example.com"]
  }
}

# Identity provider (GitHub)
resource "cloudflare_access_identity_provider" "github" {
  account_id = var.account_id
  name       = "GitHub OAuth"
  type       = "github"
  
  config {
    client_id     = var.github_client_id
    client_secret = var.github_client_secret
  }
}
```

## Architecture Patterns

### Pattern 1: Multi-Environment Setup

```hcl
# Directory structure:
# ├── modules/
# │   ├── zone/
# │   ├── worker/
# │   └── pages/
# ├── environments/
# │   ├── production/
# │   │   ├── main.tf
# │   │   └── terraform.tfvars
# │   └── staging/
# │       ├── main.tf
# │       └── terraform.tfvars

# environments/production/main.tf
module "zone" {
  source = "../../modules/zone"
  
  account_id = var.account_id
  zone_name  = "example.com"
  environment = "production"
}

module "api_worker" {
  source = "../../modules/worker"
  
  account_id = var.account_id
  zone_id    = module.zone.zone_id
  name       = "api-worker-prod"
  script     = file("${path.root}/../../workers/api.js")
  environment = "production"
}
```

### Pattern 2: Worker with Multiple Bindings

```hcl
# Complete worker setup with all binding types
locals {
  worker_name = "full-stack-worker"
}

resource "cloudflare_workers_kv_namespace" "app" {
  account_id = var.account_id
  title      = "${local.worker_name}-kv"
}

resource "cloudflare_r2_bucket" "app" {
  account_id = var.account_id
  name       = "${local.worker_name}-bucket"
}

resource "cloudflare_d1_database" "app" {
  account_id = var.account_id
  name       = "${local.worker_name}-db"
}

resource "cloudflare_worker_script" "app" {
  account_id         = var.account_id
  name               = local.worker_name
  content            = file("${path.module}/worker.js")
  module             = true
  compatibility_date = "2024-01-01"
  
  kv_namespace_binding {
    name         = "KV"
    namespace_id = cloudflare_workers_kv_namespace.app.id
  }
  
  r2_bucket_binding {
    name        = "BUCKET"
    bucket_name = cloudflare_r2_bucket.app.name
  }
  
  d1_database_binding {
    name        = "DB"
    database_id = cloudflare_d1_database.app.id
  }
  
  secret_text_binding {
    name = "API_KEY"
    text = var.api_key
  }
}
```

### Pattern 3: Zone with Complete Configuration

```hcl
# Complete zone setup with DNS, workers, and security
resource "cloudflare_zone" "main" {
  account = {
    id = var.account_id
  }
  name = var.domain
}

# DNS records
resource "cloudflare_dns_record" "root" {
  zone_id = cloudflare_zone.main.id
  name    = "@"
  content = cloudflare_pages_project.site.subdomain
  type    = "CNAME"
  proxied = true
}

resource "cloudflare_dns_record" "api" {
  zone_id = cloudflare_zone.main.id
  name    = "api"
  content = "api.${var.domain}"
  type    = "CNAME"
  proxied = true
}

# Zone settings
resource "cloudflare_zone_settings_override" "main" {
  zone_id = cloudflare_zone.main.id
  
  settings {
    ssl                      = "strict"
    always_use_https         = "on"
    min_tls_version          = "1.2"
    tls_1_3                  = "on"
    automatic_https_rewrites = "on"
    http3                    = "on"
    zero_rtt                 = "on"
    brotli                   = "on"
  }
}

# WAF Rules
resource "cloudflare_ruleset" "waf" {
  zone_id = cloudflare_zone.main.id
  name    = "Security Rules"
  kind    = "zone"
  phase   = "http_request_firewall_custom"
  
  rules {
    action      = "block"
    description = "Block common attacks"
    enabled     = true
    expression  = "(http.request.uri.path contains \"..\" or http.request.uri.path contains \"etc/passwd\")"
  }
}

# API Worker
resource "cloudflare_worker_script" "api" {
  account_id = var.account_id
  name       = "api-worker"
  content    = file("${path.module}/workers/api.js")
  module     = true
}

resource "cloudflare_worker_route" "api" {
  zone_id     = cloudflare_zone.main.id
  pattern     = "api.${var.domain}/*"
  script_name = cloudflare_worker_script.api.name
}
```

## Integration with Wrangler

**CRITICAL**: Wrangler and Terraform should NOT manage the same resources.

### Recommended Division

**Use Terraform for:**
- Zone configuration
- DNS records
- Security rules (WAF, rulesets)
- Access policies
- Load balancers
- Worker deployments (when using CI/CD)
- KV namespace creation
- R2 bucket creation
- D1 database creation

**Use Wrangler for:**
- Local development (`wrangler dev`)
- Manual deployments (`wrangler deploy`)
- D1 migrations (`wrangler d1 migrations apply`)
- KV bulk operations (`wrangler kv:bulk put`)
- Logs streaming (`wrangler tail`)
- Testing (`wrangler publish --dry-run`)

### CI/CD Integration Pattern

```hcl
# Terraform creates the infrastructure
resource "cloudflare_workers_kv_namespace" "app" {
  account_id = var.account_id
  title      = "app-kv"
}

resource "cloudflare_d1_database" "app" {
  account_id = var.account_id
  name       = "app-db"
}

# Output IDs for wrangler.toml generation
output "kv_namespace_id" {
  value = cloudflare_workers_kv_namespace.app.id
}

output "d1_database_id" {
  value = cloudflare_d1_database.app.id
}
```

```toml
# wrangler.toml (generated from Terraform outputs)
name = "app-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[kv_namespaces]]
binding = "KV"
id = "${KV_NAMESPACE_ID}"  # From Terraform output

[[d1_databases]]
binding = "DB"
database_id = "${D1_DATABASE_ID}"  # From Terraform output
```

```yaml
# GitHub Actions workflow
- name: Terraform Apply
  run: |
    terraform init
    terraform apply -auto-approve
    
- name: Generate Wrangler Config
  run: |
    export KV_NAMESPACE_ID=$(terraform output -raw kv_namespace_id)
    export D1_DATABASE_ID=$(terraform output -raw d1_database_id)
    envsubst < wrangler.toml.template > wrangler.toml
    
- name: Deploy Worker
  run: wrangler deploy
```

## Common Use Cases

### Use Case 1: Static Site with API Worker

```hcl
# Pages project for frontend
resource "cloudflare_pages_project" "frontend" {
  account_id        = var.account_id
  name              = "frontend"
  production_branch = "main"
  
  build_config {
    build_command   = "npm run build"
    destination_dir = "dist"
  }
}

# Worker for API
resource "cloudflare_worker_script" "api" {
  account_id = var.account_id
  name       = "api"
  content    = file("${path.module}/api-worker.js")
  
  d1_database_binding {
    name        = "DB"
    database_id = cloudflare_d1_database.api_db.id
  }
}

# DNS records
resource "cloudflare_dns_record" "frontend" {
  zone_id = cloudflare_zone.main.id
  name    = "app"
  content = cloudflare_pages_project.frontend.subdomain
  type    = "CNAME"
  proxied = true
}

resource "cloudflare_dns_record" "api" {
  zone_id = cloudflare_zone.main.id
  name    = "api"
  content = "api.example.com"
  type    = "CNAME"
  proxied = true
}

resource "cloudflare_worker_route" "api" {
  zone_id     = cloudflare_zone.main.id
  pattern     = "api.example.com/*"
  script_name = cloudflare_worker_script.api.name
}
```

### Use Case 2: Multi-Region Load Balancing

```hcl
# US Pool
resource "cloudflare_load_balancer_pool" "us" {
  account_id = var.account_id
  name       = "us-pool"
  
  origins {
    name    = "us-east-1"
    address = var.us_east_ip
  }
  
  origins {
    name    = "us-west-1"
    address = var.us_west_ip
  }
  
  monitor = cloudflare_load_balancer_monitor.http.id
}

# EU Pool
resource "cloudflare_load_balancer_pool" "eu" {
  account_id = var.account_id
  name       = "eu-pool"
  
  origins {
    name    = "eu-west-1"
    address = var.eu_west_ip
  }
  
  monitor = cloudflare_load_balancer_monitor.http.id
}

# Geo-steering load balancer
resource "cloudflare_load_balancer" "global" {
  zone_id          = cloudflare_zone.main.id
  name             = "api.example.com"
  default_pool_ids = [cloudflare_load_balancer_pool.us.id]
  
  steering_policy = "geo"
  
  region_pools {
    region   = "WNAM"  # Western North America
    pool_ids = [cloudflare_load_balancer_pool.us.id]
  }
  
  region_pools {
    region   = "WEU"  # Western Europe
    pool_ids = [cloudflare_load_balancer_pool.eu.id]
  }
}
```

### Use Case 3: Secure Admin Panel with Access

```hcl
# Pages project for admin panel
resource "cloudflare_pages_project" "admin" {
  account_id        = var.account_id
  name              = "admin-panel"
  production_branch = "main"
}

# Access application
resource "cloudflare_access_application" "admin" {
  account_id = var.account_id
  name       = "Admin Panel"
  domain     = "admin.example.com"
  type       = "self_hosted"
  
  session_duration = "24h"
  allowed_idps     = [cloudflare_access_identity_provider.google.id]
}

# Access policy - only allow specific emails
resource "cloudflare_access_policy" "admin_allow" {
  account_id     = var.account_id
  application_id = cloudflare_access_application.admin.id
  name           = "Allow admins"
  decision       = "allow"
  precedence     = 1
  
  include {
    email = var.admin_emails
  }
}

# DNS record
resource "cloudflare_dns_record" "admin" {
  zone_id = cloudflare_zone.main.id
  name    = "admin"
  content = cloudflare_pages_project.admin.subdomain
  type    = "CNAME"
  proxied = true
}
```

## Best Practices

### 1. Resource Naming

```hcl
# Good: Consistent naming with environment
locals {
  env_prefix = "${var.environment}-${var.project_name}"
}

resource "cloudflare_worker_script" "api" {
  name = "${local.env_prefix}-api"
}

resource "cloudflare_workers_kv_namespace" "cache" {
  title = "${local.env_prefix}-cache"
}
```

### 2. Output Important Values

```hcl
output "zone_id" {
  value       = cloudflare_zone.main.id
  description = "Zone ID for DNS management"
}

output "worker_url" {
  value       = "https://${cloudflare_worker_domain.api.hostname}"
  description = "Worker API endpoint"
}

output "kv_namespace_id" {
  value       = cloudflare_workers_kv_namespace.app.id
  sensitive   = false
}
```

### 3. Use Data Sources for Existing Resources

```hcl
# Reference existing zone
data "cloudflare_zone" "main" {
  name = var.domain
}

# Reference existing account
data "cloudflare_accounts" "main" {
  name = var.account_name
}

# Use in resources
resource "cloudflare_worker_route" "api" {
  zone_id = data.cloudflare_zone.main.id
  # ...
}
```

### 4. Separate Secrets from Code

```hcl
# variables.tf
variable "cloudflare_api_token" {
  type        = string
  sensitive   = true
  description = "Cloudflare API token"
}

# terraform.tfvars (gitignored)
cloudflare_api_token = "actual-token-here"

# Or use environment variables
# export TF_VAR_cloudflare_api_token="actual-token-here"
```

### 5. Use Terraform Workspaces or Separate Directories

```hcl
# Option 1: Workspaces
terraform workspace new production
terraform workspace new staging

# Option 2: Separate directories (RECOMMENDED)
# environments/
#   production/
#   staging/
#   development/
```

### 6. Version Control State Locking

```hcl
# S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "cloudflare/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

## Troubleshooting

### Issue: "Error: couldn't find resource"

**Cause**: Resource was deleted outside Terraform  
**Fix**:
```bash
terraform import cloudflare_zone.example <zone-id>
# Or remove from state:
terraform state rm cloudflare_zone.example
```

### Issue: "409 Conflict" on worker deployment

**Cause**: Worker being deployed by both Terraform and wrangler  
**Fix**: Choose one deployment method. If using Terraform, remove wrangler deployments.

### Issue: DNS record already exists

**Cause**: Existing record not imported into Terraform  
**Fix**:
```bash
# Find record ID in Cloudflare dashboard
terraform import cloudflare_dns_record.example <zone-id>/<record-id>
```

### Issue: "Invalid provider configuration"

**Cause**: API token missing or invalid  
**Fix**:
```bash
export CLOUDFLARE_API_TOKEN="your-token"
# Or check token permissions in dashboard
```

## Data Sources Reference

```hcl
# Get zone by name
data "cloudflare_zone" "example" {
  name = "example.com"
}

# List all accounts
data "cloudflare_accounts" "main" {
  name = "My Account"
}

# Get existing worker script
data "cloudflare_worker_script" "existing" {
  account_id = var.account_id
  name       = "existing-worker"
}

# Get KV namespace
data "cloudflare_workers_kv_namespace" "existing" {
  account_id   = var.account_id
  namespace_id = "abc123"
}

# Get IP ranges (for firewall rules)
data "cloudflare_ip_ranges" "cloudflare" {}

output "ipv4_cidrs" {
  value = data.cloudflare_ip_ranges.cloudflare.ipv4_cidr_blocks
}
```

## Module Example

```hcl
# modules/cloudflare-zone/main.tf
variable "account_id" {
  type = string
}

variable "domain" {
  type = string
}

variable "ssl_mode" {
  type    = string
  default = "strict"
}

resource "cloudflare_zone" "main" {
  account = {
    id = var.account_id
  }
  name = var.domain
}

resource "cloudflare_zone_settings_override" "main" {
  zone_id = cloudflare_zone.main.id
  
  settings {
    ssl              = var.ssl_mode
    always_use_https = "on"
    min_tls_version  = "1.2"
  }
}

output "zone_id" {
  value = cloudflare_zone.main.id
}

output "name_servers" {
  value = cloudflare_zone.main.name_servers
}

# Usage:
module "production_zone" {
  source = "./modules/cloudflare-zone"
  
  account_id = var.account_id
  domain     = "example.com"
  ssl_mode   = "strict"
}
```

## Quick Reference: Common Commands

```bash
# Initialize provider
terraform init

# Plan changes
terraform plan

# Apply changes
terraform apply

# Apply with auto-approve
terraform apply -auto-approve

# Destroy resources
terraform destroy

# Import existing resource
terraform import cloudflare_zone.example <zone-id>

# Show current state
terraform show

# List resources in state
terraform state list

# Remove resource from state (without destroying)
terraform state rm cloudflare_zone.example

# Refresh state from actual infrastructure
terraform refresh

# Format code
terraform fmt -recursive

# Validate configuration
terraform validate

# Show outputs
terraform output

# Show specific output
terraform output zone_id
```

## Security Considerations

1. **Never commit secrets**: Use variables + environment vars or secret management tools
2. **Scope API tokens**: Create tokens with minimal required permissions
3. **Enable state encryption**: Use encrypted S3 backend or Terraform Cloud
4. **Use separate tokens per environment**: Different tokens for prod/staging
5. **Rotate tokens regularly**: Update tokens in CI/CD systems
6. **Review terraform plans**: Always review before applying
7. **Use Access for sensitive applications**: Don't expose admin panels publicly

## Additional Resources

- **Provider Documentation**: https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs
- **Provider GitHub**: https://github.com/cloudflare/terraform-provider-cloudflare
- **Cloudflare API Docs**: https://developers.cloudflare.com/api
- **Terraform Best Practices**: https://www.terraform-best-practices.com/
- **Cloudflare Developer Discord**: Join for provider support questions
