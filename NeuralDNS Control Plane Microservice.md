# NeuralDNS Control Plane Microservice (CPMS)

## Overview

The **Control Plane Microservice (CPMS)** is the central orchestration system for the NeuralDNS edge proxy infrastructure. It manages the lifecycle of shards, stations, domains, certificates, and provides comprehensive monitoring, operational capabilities, automated station deployment, and AI-ready features (MCP Protocol, x402 metadata).

## Architecture

### Core Concepts

- **Shard**: A logical grouping of up to 250 domains, backed by 3 EC2 instances (stations) and a Network Load Balancer with static Elastic IPs
- **Station**: An EC2 instance running the NeuralDNS edge proxy (station.jar), handling TLS termination and HTTP/HTTPS requests
- **Domain**: A tenant's domain configured for edge proxy services
- **Region**: AWS regions where shards are deployed (Primary: eu-west-3, Secondary: us-west-1)
- **Mirroring**: Automatic replication of shards to secondary region for high availability

### Infrastructure Architecture

```
                    DNS A Record
                         │
                         ▼
              ┌─────────────────────┐
              │    Elastic IPs      │  ← Static IPs for DNS
              │  (one per AZ)       │
              └─────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │  Network Load       │  ← Layer 4 (TCP)
              │  Balancer (NLB)     │     No TLS termination
              └─────────────────────┘
                    │    │    │
           ┌────────┘    │    └────────┐
           ▼             ▼             ▼
      ┌─────────┐   ┌─────────┐   ┌─────────┐
      │ Station │   │ Station │   │ Station │  ← TLS Termination
      │  (AZ-a) │   │  (AZ-b) │   │  (AZ-c) │     with Keystore
      └─────────┘   └─────────┘   └─────────┘
```

**Why NLB instead of ALB:**
- NLB natively supports Elastic IPs (required for static DNS A records)
- Layer 4 provides lower latency than Layer 7
- Stations already handle TLS termination with JKS keystores
- Simpler architecture, lower cost

### Key Features

1. **Automated Infrastructure Provisioning**
   - EC2 instance deployment with user data scripts
   - Network Load Balancer (NLB) setup with target groups
   - Elastic IP allocation per AZ for static DNS A records
   - Multi-AZ deployment for high availability

2. **Certificate Management**
   - Let's Encrypt integration via ACME protocol (acme4j)
   - HTTP-01 challenge validation
   - Automatic certificate renewal (30-day threshold)
   - JKS keystore generation and management
   - Self-signed certificates for testing

3. **Domain Management**
   - Multi-tenant domain isolation
   - Automatic shard assignment (fewest domains algorithm)
   - Auto-provisioning of new shards when capacity reached
   - Zero-downtime configuration reloads

4. **Station Deployment**
   - Automated JAR deployment via AWS Systems Manager (SSM)
   - Health check verification post-deployment
   - Automatic rollback on health check failure
   - Multiple deployment strategies (Canary, Rolling, Parallel)

5. **MCP Protocol Support**
   - Model Context Protocol for AI agent tool discovery
   - Tool target reachability testing
   - Manifest preview generation

6. **x402 Metadata Support**
   - Economic and operational hints for domains
   - Rate limit recommendations
   - Cost and quality indicators

7. **Development & Testing**
   - Dev endpoints for test deployments
   - Self-signed certificate generation
   - Configs-only mode for manual infrastructure

8. **Monitoring & Alerting**
   - Real-time station heartbeat tracking
   - PostgreSQL-based metrics storage with analytics
   - SNS/SES alerting
   - Built-in metrics retention and cleanup

## Technology Stack

- **Framework**: Spring Boot 3.2
- **Language**: Java 17
- **Database**: PostgreSQL with JSONB support
- **ORM**: Spring Data JPA / Hibernate
- **AWS SDK**: AWS SDK for Java v2 (EC2, ELB, S3, SSM, SNS, SES)
- **Security**: Spring Security with API key authentication
- **Certificate**: BouncyCastle, acme4j (Let's Encrypt)
- **Build**: Maven

## Project Structure

```
cpms/
├── src/main/java/com/freename/neuraldns/cpms/
│   ├── aws/                          # AWS Service Integrations
│   │   ├── Ec2ProvisioningService.java
│   │   ├── NlbProvisioningService.java   # Network Load Balancer
│   │   └── S3ConfigService.java
│   ├── certificate/                  # Certificate Management
│   │   └── CertificateService.java        # Keystore + self-signed certs
│   ├── config/                       # Configuration Classes
│   │   ├── AwsConfig.java                 # AWS configuration
│   │   └── ...
│   ├── controller/                   # REST Controllers
│   │   ├── ConsolidatedDomainController.java
│   │   ├── OperationsController.java
│   │   ├── MonitoringController.java
│   │   ├── MetricsController.java         # Metrics analytics API
│   │   ├── StationController.java
│   │   ├── DevTestController.java         # Dev/test endpoints
│   │   └── AcmeChallengeController.java
│   ├── domain/                       # JPA Entities
│   │   └── ...
│   ├── dto/                          # Data Transfer Objects
│   │   └── ...
│   ├── event/                        # Spring Events
│   │   ├── KeystoreUpdatedEvent.java
│   │   └── KeystoreEventListener.java
│   ├── repository/                   # Spring Data Repositories
│   │   └── ...
│   ├── security/                     # Security Components
│   │   ├── SecurityConfig.java
│   │   └── ...
│   └── service/                      # Business Logic Services
│       ├── DevTestDeploymentService.java  # Test deployment logic
│       ├── CertificateRenewalService.java
│       ├── DomainManagementService.java
│       ├── MetricsForwardingService.java  # Metrics to PostgreSQL
│       ├── MetricsQueryService.java       # Metrics analytics
│       └── ...
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       └── V1_5_0__alb_to_nlb_migration.sql
└── pom.xml
```

---

## API Reference

### Authentication

All API endpoints require authentication:

| API Group | Authentication Method |
|-----------|----------------------|
| Domain API (`/api/v1/domains/**`) | `X-API-Key` header |
| Tenant API (`/api/v1/tenants/**`) | `X-API-Key` header |
| Operations API (`/api/v1/operations/**`) | `X-API-Key` header + `?confirm=true` |
| Metrics API (`/api/v1/metrics/**`) | `X-API-Key` header |
| Station API (`/api/v1/stations/**`) | `Authorization: Bearer {token}` + IP validation |
| Monitoring API (`/api/v1/monitoring/**`) | `X-API-Key` header |
| Dev API (`/api/v1/dev/**`) | `X-API-Key` header (must be enabled) |
| Health Endpoints (`/actuator/**`) | No authentication |
| ACME Challenge (`/.well-known/acme-challenge/**`) | No authentication |

---

## Domain Management API

### PUT /api/v1/domains/{domain}

Full domain upsert - one call to configure everything.

```http
PUT /api/v1/domains/example.com
X-API-Key: {api-key}
Content-Type: application/json
```

```json
{
  "tenantId": "acme-corp",
  "snapshotVersion": "v1",
  "routing": [{
    "subdomain": "",
    "pathPrefix": "/",
    "upstreamUrl": "https://backend.internal",
    "priority": 0
  }],
  "cache": { "enabled": true, "ttlSeconds": 300 },
  "rateLimit": { "enabled": true, "requests": 1000, "windowSeconds": 60 },
  "mcp": { "enabled": true, "manifests": [...] },
  "x402": { "rateLimit": { "recommendedRps": 100 }, "cost": { "model": "free" } }
}
```

### GET /api/v1/domains/{domain}

Get complete domain info (config + status + MCP + x402).

### DELETE /api/v1/domains/{domain}?tenantId={tenantId}

Delete a domain.

### GET /api/v1/domains

List domains with filters (`tenantId`, `shardId`, `active`, `limit`, `offset`).

---

## Tenant Bulk Operations

### PUT /api/v1/tenants/{tenantId}/domains

Bulk upsert multiple domains for a tenant.

### GET /api/v1/tenants/{tenantId}/domains

List all domains for a tenant.

---

## Operations API

All mutation operations require `?confirm=true`.

### POST /api/v1/operations/shards/provision

Provision a new shard in a region.

### POST /api/v1/operations/shards/{shardId}/reload

Trigger configuration reload for all stations in a shard.

### POST /api/v1/operations/deployments

Deploy station JAR version with strategy (canary, rolling, parallel).

### GET /api/v1/operations/{jobId}

Get operation status.

### GET /api/v1/operations/running

List running operations.

---

## Metrics API

Query station metrics stored in PostgreSQL. Provides analytics for dashboards and monitoring.

### GET /api/v1/metrics/shards/{shardId}/summary

Get metrics summary for a shard including request counts, response times, error rates.

**Query Parameters:**
- `minutes` (optional, default: 60) - Time window in minutes

**Response:**
```json
{
  "success": true,
  "data": {
    "shardId": "test-shard-1",
    "periodMinutes": 60,
    "totalRequests": 15420,
    "errorRequests": 23,
    "errorRate": 0.15,
    "avgResponseTimeMs": 45.2,
    "p95ResponseTimeMs": 120.5,
    "statusCodeDistribution": {
      "200": 14500,
      "304": 850,
      "404": 47,
      "500": 23
    },
    "topDomains": {
      "example.com": 8500,
      "api.example.com": 6920
    }
  }
}
```

### GET /api/v1/metrics/shards/{shardId}/raw

Get raw metric records for a shard (max 1000 records).

**Query Parameters:**
- `minutes` (optional, default: 15) - Time window in minutes

### GET /api/v1/metrics/domains/{domain}/cache

Get cache statistics for a domain.

**Query Parameters:**
- `minutes` (optional, default: 60) - Time window in minutes

**Response:**
```json
{
  "success": true,
  "data": {
    "domain": "example.com",
    "periodMinutes": 60,
    "cacheHitRate": 78.5
  }
}
```

### GET /api/v1/metrics/domains/{domain}/raw

Get raw metric records for a domain (max 1000 records).

### GET /api/v1/metrics/stats

Get quick stats for dashboard.

**Query Parameters:**
- `shardId` (optional) - Filter by shard
- `minutes` (optional, default: 15) - Time window in minutes

**Response:**
```json
{
  "success": true,
  "data": {
    "shardId": "test-shard-1",
    "periodMinutes": 15,
    "totalRequests": 3855,
    "requestsPerMinute": 257.0,
    "avgResponseTimeMs": 42.1,
    "p95ResponseTimeMs": 115.3,
    "errorRate": 0.12
  }
}
```

---

## Station API (Internal)

Used by stations to communicate with CPMS.

### POST /api/v1/stations/metrics

Ingest metrics batch from a station.

```json
{
  "shard_id": "shard-1-eu-west-3",
  "station_id": "station-uuid-123",
  "instance_id": "i-0abc123def456",
  "batch_type": "REQUEST_METRICS",
  "metrics": [...]
}
```

### POST /api/v1/stations/reload-callback

Report configuration reload result.

```json
{
  "shard_id": "shard-1-eu-west-3",
  "station_id": "station-uuid-123",
  "instance_id": "i-0abc123def456",
  "reload_event_id": "evt-abc-123",
  "success": true,
  "reload_duration_ms": 245
}
```

### POST /api/v1/stations/heartbeat

Station heartbeat for health monitoring.

---

## Dev/Test API

**⚠️ Must be enabled via `DEV_ENDPOINTS_ENABLED=true`**

### GET /api/v1/dev/status

Check if dev endpoints are enabled.

### POST /api/v1/dev/test-deployment?confirm=true

Create a full test deployment (EC2 + NLB + S3 configs).

```json
{
  "shardName": "test-shard-1",
  "region": "eu-west-3",
  "stationCount": 1,
  "domains": [
    {
      "domainName": "test.example.com",
      "upstreamUrl": "https://httpbin.org",
      "cacheEnabled": false,
      "rateLimitEnabled": false
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "shardId": "test-shard-1",
    "region": "eu-west-3",
    "status": "DEPLOYED",
    "elasticIps": ["52.47.x.x"],
    "nlbDnsName": "test-shard-1-xxx.elb.eu-west-3.amazonaws.com",
    "stationInstanceIds": ["i-0abc123"],
    "keystorePassword": "generated-uuid",
    "testInstructions": {
      "1_dns_setup": "Point test.example.com DNS A record to 52.47.x.x",
      "2_wait": "Wait 1-2 minutes for health check",
      "3_test_curl": "curl -k https://test.example.com/"
    }
  }
}
```

### POST /api/v1/dev/test-deployment/configs-only?confirm=true

Create S3 configs only (no infrastructure). Use when you have EC2/NLB manually created.

### GET /api/v1/dev/test-deployment/{shardId}

Get test deployment status.

### DELETE /api/v1/dev/test-deployment/{shardId}?confirm=true&deleteInfrastructure=true

Cleanup test deployment. Set `deleteInfrastructure=true` to terminate EC2 and delete NLB.

---

## Monitoring API

### GET /api/v1/monitoring/shards

Shard monitoring data with filters.

### GET /api/v1/monitoring/stations

Station monitoring data.

### GET /api/v1/monitoring/domains

Domain monitoring data.

### GET /api/v1/monitoring/metrics/summary

Aggregated metrics summary.

### GET /api/v1/monitoring/failures

Operation failure history.

---

## Configuration

### Environment Variables

```bash
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=cpms
DB_USERNAME=cpms
DB_PASSWORD=changeit

# AWS
AWS_ACCESS_KEY_ID=<your-access-key>
AWS_SECRET_ACCESS_KEY=<your-secret-key>
AWS_REGION=eu-west-3

# Security
FREENAME_API_KEY=<your-api-key>
KEYSTORE_PASSWORD=<keystore-password>

# Let's Encrypt
LETSENCRYPT_ACCOUNT_KEY=/var/cpms/letsencrypt-account.pem
LETSENCRYPT_STAGING=false

# Timestream
TIMESTREAM_DATABASE=neuraldns
TIMESTREAM_TABLE=station-metrics

# Dev/Test (DISABLE IN PRODUCTION)
DEV_ENDPOINTS_ENABLED=false
```

### application.yml

```yaml
aws:
  regions:
    primary: eu-west-3
    secondary: us-west-1
  s3:
    bucket-name: freename-neuraldns
  timestream:
    database: neuraldns
    table: station-metrics

neuraldns:
  shard:
    max-domains-per-shard: 250
    instances-per-shard: 3
  dev:
    enabled: false  # Enable for testing
```

---

## Scheduled Jobs

| Job | Frequency | Purpose |
|-----|-----------|---------|
| Certificate Renewal | Daily at 2 AM | Check and renew expiring certificates |
| Metrics Forwarding | Every 60 seconds | Forward metrics to Timestream |
| Station Monitoring | Every 60 seconds | Check heartbeats, update shard status |

---

## Database Migration

### ALB to NLB Migration (v1.5.0)

```bash
psql -d cpms -f V1_5_0__alb_to_nlb_migration.sql
```

This migration:
- Renames `alb_arn` → `nlb_arn`
- Renames `alb_dns_name` → `nlb_dns_name`
- Adds `elastic_ips` (comma-separated, multiple EIPs)
- Adds `elastic_ip_allocation_ids`

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.5.0 | Jan 2026 | NLB migration, security hardening, dev endpoints, event-driven architecture |
| 1.4.0 | Dec 2025 | API consolidation, Let's Encrypt integration |
| 1.3.0 | Dec 2025 | MCP Protocol, x402 Metadata support |
| 1.2.0 | Dec 2025 | Station deployment, health checks, auto-rollback |
| 1.1.0 | Dec 2025 | Cache invalidation, rate limiting |
| 1.0.0 | Dec 2025 | Initial release |

### v1.5.0 Highlights

- **ALB → NLB Migration**: Static Elastic IPs per AZ for DNS A records
- **Security Hardening**: All `/api/v1/**` endpoints require authentication
- **Dev/Test Endpoints**: Quick test deployments with self-signed certs
- **Event-Driven Architecture**: Decoupled keystore updates via Spring Events
- **Bug Fixes**: Multiple critical and high-priority fixes

---

## GEO Module - Generative Engine Optimization

### Overview

The **GEO (Generative Engine Optimization)** module provides brand monitoring capabilities for AI/LLM search engines. It measures and tracks how well a brand is represented in AI-generated responses, diagnoses visibility issues, and generates remediation tasks.

**Key Principle**: All scoring, diagnosis, and drift detection logic is **deterministic** (no LLM involvement). LLMs are only used for probe execution and optional remediation task enrichment.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GEO Audit Pipeline                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐           │
│  │  Probe   │───▶│   Signal     │───▶│    Scoring      │           │
│  │Execution │    │  Extraction  │    │    Engine       │           │
│  │  (LLM)   │    │(Deterministic)│   │ (Deterministic) │           │
│  └──────────┘    └──────────────┘    └────────┬────────┘           │
│                                               │                     │
│                                               ▼                     │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐           │
│  │  Drift   │◀───│  Diagnosis   │◀───│   5-Dimension   │           │
│  │Detection │    │   Service    │    │     Scores      │           │
│  └────┬─────┘    └──────┬───────┘    └─────────────────┘           │
│       │                 │                                           │
│       ▼                 ▼                                           │
│  ┌──────────┐    ┌──────────────┐                                  │
│  │  Alerts  │    │ Remediation  │                                  │
│  │ Webhooks │    │    Tasks     │                                  │
│  └──────────┘    └──────────────┘                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Scoring Dimensions

The GEO scoring engine evaluates brand visibility across 5 weighted dimensions:

| Dimension | Weight | Description |
|-----------|--------|-------------|
| **Existence** | 30% | Is the brand mentioned? Exact match, alias matches, mention count |
| **Clarity** | 25% | Role match quality via Jaccard similarity with canonical phrase |
| **Authority** | 20% | Third-party validations (Gartner, Forrester, etc.), citations |
| **Consistency** | 15% | Variance across probes and historical scores |
| **Safety** | 10% | Penalty for ambiguity indicators, competitor dominance |

### Issue Codes

The diagnosis service identifies issues based on score thresholds:

| Issue Code | Severity | Trigger Condition |
|------------|----------|-------------------|
| `MISSING_ENTITY_GROUNDING` | Critical | Existence < 40 |
| `ROLE_MISMATCH` | Critical | Clarity < 30 |
| `WEAK_AUTHORITY` | Critical | Authority < 25 |
| `NAME_AMBIGUITY` | Critical | Safety penalty > 20 |
| `LOW_VISIBILITY` | High | Existence < 60 |
| `INCONSISTENT_FRAMING` | High | Clarity < 50 or Consistency < 50 |
| `COMPETITOR_DOMINANCE` | High | Competitor mentions > brand mentions |
| `NO_CITATIONS` | Medium | Zero citations detected |

---

## GEO API Reference

### Authentication

GEO endpoints use the same `X-API-Key` authentication as other CPMS APIs.

### Project Management

#### POST /api/v1/geo/projects

Create a new GEO project.

```http
POST /api/v1/geo/projects
X-API-Key: {api-key}
X-Tenant-Id: {tenant-id}
Content-Type: application/json
```

```json
{
  "domainName": "example.com",
  "brandName": "Acme Corp",
  "brandAliases": ["Acme", "ACME Inc"],
  "canonicalPhrase": "enterprise cloud solutions provider",
  "probeProvider": "openai",
  "probeConfig": { "model": "gpt-4o", "temperature": 0.3 },
  "webhookUrl": "https://hooks.example.com/geo",
  "webhookSecret": "whsec_xxx",
  "webhookEnabled": true,
  "regressionThreshold": 10
}
```

#### GET /api/v1/geo/projects/{projectId}

Get project details.

#### GET /api/v1/geo/projects/domain/{domainName}

Get project by domain name.

#### PUT /api/v1/geo/projects/{projectId}

Update project configuration.

#### DELETE /api/v1/geo/projects/{projectId}?hardDelete=false

Soft delete (deactivate) or hard delete a project.

#### GET /api/v1/geo/tenants/{tenantId}/projects?activeOnly=true

List all projects for a tenant.

### Audit Operations

#### POST /api/v1/geo/projects/{projectId}/audits

Trigger a new audit run.

```json
{ "triggeredBy": "user@example.com" }
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "run-uuid",
    "projectId": "project-uuid",
    "status": "PENDING",
    "triggeredBy": "user@example.com",
    "createdAt": "2026-01-15T12:00:00Z"
  }
}
```

#### GET /api/v1/geo/runs/{runId}

Get detailed audit run results including probes, issues, tasks, and alerts.

#### POST /api/v1/geo/runs/{runId}/cancel?cancelledBy={user}

Cancel a running audit.

#### GET /api/v1/geo/projects/{projectId}/audits?limit=10

List recent audits for a project.

### Probe Results

#### GET /api/v1/geo/runs/{runId}/probes

List all probe results for an audit run.

#### GET /api/v1/geo/probes/{probeId}

Get detailed probe result including extracted signals.

### Diagnosis Issues

#### GET /api/v1/geo/runs/{runId}/issues

List issues identified in an audit run (sorted by severity).

#### GET /api/v1/geo/issues/{issueId}

Get issue details including affected dimensions and evidence.

### Remediation Tasks

#### GET /api/v1/geo/runs/{runId}/tasks

List remediation tasks generated for an audit run.

#### GET /api/v1/geo/tasks/{taskId}

Get task details including actions and estimated impact.

#### PUT /api/v1/geo/tasks/{taskId}/status

Update task status.

```json
{ "status": "IN_PROGRESS" }
```

Valid statuses: `OPEN`, `IN_PROGRESS`, `COMPLETED`, `WONT_FIX`

#### GET /api/v1/geo/projects/{projectId}/tasks/open

List all open tasks for a project.

### Drift Alerts

#### GET /api/v1/geo/runs/{runId}/alerts

List drift alerts for an audit run.

#### GET /api/v1/geo/alerts/{alertId}

Get alert details.

#### POST /api/v1/geo/alerts/{alertId}/acknowledge

Acknowledge an alert.

```json
{ "acknowledgedBy": "user@example.com" }
```

#### GET /api/v1/geo/projects/{projectId}/alerts/unacknowledged

List unacknowledged alerts for a project.

### Analytics

#### GET /api/v1/geo/projects/{projectId}/scores/history?days=30

Get score history and trends.

**Response:**
```json
{
  "projectId": "project-uuid",
  "history": [
    {
      "scoreDate": "2026-01-15",
      "totalScore": 72.5,
      "existenceScore": 85.0,
      "clarityScore": 70.0,
      "authorityScore": 65.0,
      "consistencyScore": 80.0,
      "safetyScore": 62.5
    }
  ],
  "averageScore": 71.2,
  "trend": 3.5
}
```

#### GET /api/v1/geo/projects/{projectId}/citations/metrics?days=30

Get citation analytics.

### Templates

#### GET /api/v1/geo/templates

List available probe templates.

#### GET /api/v1/geo/templates/{templateId}

Get template details.

### Webhooks

#### POST /api/v1/geo/webhooks/test

Test webhook connectivity.

```json
{
  "webhookUrl": "https://hooks.example.com/geo",
  "webhookSecret": "whsec_xxx"
}
```

#### GET /api/v1/geo/projects/{projectId}/webhooks/status

Get webhook configuration status.

---

## GEO Configuration

### Environment Variables

```bash
# GEO Module
NEURALDNS_GEO_ENABLED=true
NEURALDNS_GEO_OPENAI_API_KEY=sk-xxx
NEURALDNS_GEO_DEFAULT_PROVIDER=openai  # or 'mock' for testing

# Probe Execution
NEURALDNS_GEO_MAX_CONCURRENT_PROBES=5
NEURALDNS_GEO_PROBE_DELAY_MS=200
NEURALDNS_GEO_PROBE_TIMEOUT_SECONDS=60

# Webhooks
NEURALDNS_GEO_WEBHOOKS_ENABLED=true
NEURALDNS_GEO_WEBHOOK_TIMEOUT_SECONDS=30

# Scoring
NEURALDNS_GEO_DEFAULT_REGRESSION_THRESHOLD=10
NEURALDNS_GEO_CONSISTENCY_WINDOW=5
NEURALDNS_GEO_ROLE_MATCH_THRESHOLD=0.7
```

### application.yml

```yaml
neuraldns:
  geo:
    enabled: true
    openai-api-key: ${NEURALDNS_GEO_OPENAI_API_KEY}
    default-provider: openai
    max-concurrent-probes: 5
    probe-delay-ms: 200
    probe-timeout-seconds: 60
    webhooks-enabled: true
    webhook-timeout-seconds: 30
    jobs:
      check-interval-seconds: 30
      max-concurrent-runs: 3
      stale-run-timeout-minutes: 60
    scoring:
      default-regression-threshold: 10
      consistency-window: 5
      role-match-threshold: 0.7
```

---

## GEO Webhook Payloads

GEO sends webhook notifications for drift alerts. Payloads are signed with HMAC-SHA256.

### Headers

```
X-GEO-Signature: sha256=<hmac-signature>
X-GEO-Event: drift.alert
Content-Type: application/json
```

### Payload Structure

```json
{
  "eventType": "DRIFT_ALERT",
  "timestamp": "2026-01-15T12:00:00Z",
  "project": {
    "id": "project-uuid",
    "domainName": "example.com",
    "brandName": "Acme Corp"
  },
  "alert": {
    "id": "alert-uuid",
    "type": "SCORE_REGRESSION",
    "severity": "HIGH",
    "previousScore": 75.0,
    "currentScore": 58.0,
    "delta": -17.0,
    "affectedDimensions": ["existence", "authority"],
    "message": "Total score dropped by 17 points"
  },
  "runId": "run-uuid"
}
```

---

## GEO Database Schema

The GEO module adds 10 tables to the CPMS database:

| Table | Purpose |
|-------|---------|
| `geo_projects` | Project configuration and brand settings |
| `geo_probe_templates` | Reusable probe query templates |
| `geo_scoring_configs` | Versioned scoring weight configurations |
| `geo_audit_runs` | Audit execution tracking |
| `geo_probe_results` | Individual probe responses and signals |
| `geo_diagnosis_issues` | Identified visibility issues |
| `geo_remediation_tasks` | Generated remediation tasks |
| `geo_drift_alerts` | Score regression alerts |
| `geo_score_history` | Historical score time series |
| `geo_citation_events` | Citation tracking events |

---

## GEO Scheduled Jobs

| Job | Frequency | Purpose |
|-----|-----------|---------|
| GEO Audit Processing | Every 30 seconds | Process pending audit runs |
| GEO Stale Run Cleanup | Every 5 minutes | Timeout stale runs after 60 minutes |

---

## Version History (Updated)

| Version | Date | Changes |
|---------|------|---------|
| 1.7.0 | Feb 2026 | **PostgreSQL Metrics**: Replaced Timestream with PostgreSQL for metrics storage |
| 1.6.0 | Jan 2026 | **GEO Module**: Brand monitoring for AI search engines |
| 1.5.0 | Jan 2026 | NLB migration, security hardening, dev endpoints |
| 1.4.0 | Dec 2025 | API consolidation, Let's Encrypt integration |
| 1.3.0 | Dec 2025 | MCP Protocol, x402 Metadata support |
| 1.2.0 | Dec 2025 | Station deployment, health checks, auto-rollback |
| 1.1.0 | Dec 2025 | Cache invalidation, rate limiting |
| 1.0.0 | Dec 2025 | Initial release |

### v1.7.0 Highlights

- **PostgreSQL Metrics Storage**: Replaced AWS Timestream with PostgreSQL for better availability and cost
- **Metrics API**: New REST endpoints for querying metrics analytics
- **Built-in Analytics**: Average/P95 response times, error rates, cache hit rates, status code distribution
- **Automatic Retention**: Configurable retention with daily cleanup job (default: 30 days)
- **StationMetric Entity**: New JPA entity with optimized indexes for time-series queries

### v1.6.0 Highlights

- **GEO Module**: Complete brand monitoring system for AI/LLM visibility
- **5-Dimension Scoring**: Existence, Clarity, Authority, Consistency, Safety
- **Deterministic Analysis**: All scoring and diagnosis without LLM involvement
- **Drift Detection**: Automatic alerts for score regressions
- **Remediation Tasks**: Actionable tasks with owner assignment and impact estimates
- **Webhook Integration**: HMAC-signed notifications for drift alerts

---

## License

Proprietary - Freename NeuralDNS Platform

Copyright (c) 2025-2026 Freename. All rights reserved.
