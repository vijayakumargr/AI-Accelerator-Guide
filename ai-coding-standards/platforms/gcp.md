# GCP Platform Instructions

## General Principles
- **ALWAYS use Infrastructure as Code** (Terraform, Deployment Manager, Pulumi)
- **ALWAYS follow Google Cloud Architecture Framework** principles
- **ALWAYS enable audit logging** for all services
- **ALWAYS use service accounts** with minimal permissions
- **NEVER hardcode credentials** in code or configuration
- **NEVER use default service accounts** for production workloads

---

# IAM Best Practices

## Principles
- **Least Privilege**: Grant only permissions needed for the task
- **Use Service Accounts**: Prefer service accounts over user accounts for applications
- **Workload Identity**: Use for GKE workloads to avoid key management
- **Regular Audit**: Review permissions with IAM Recommender

## IAM Binding Structure
```hcl
# GOOD: Specific role binding with conditions
resource "google_project_iam_member" "storage_viewer" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"

  condition {
    title       = "expires_after_2024"
    description = "Temporary access"
    expression  = "request.time < timestamp('2024-12-31T00:00:00Z')"
  }
}

# BAD: Overly permissive
resource "google_project_iam_member" "admin" {
  project = var.project_id
  role    = "roles/owner"  # Never use owner for service accounts
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

## IAM Checklist
- [ ] No primitive roles (Owner, Editor) for service accounts
- [ ] Custom roles for fine-grained permissions
- [ ] IAM conditions for additional restrictions
- [ ] Workload Identity for GKE pods
- [ ] Regular review with IAM Recommender

---

# Compute Services

## Compute Engine Best Practices
- Use custom machine types for optimal sizing
- Enable shielded VMs for enhanced security
- Use managed instance groups for auto-scaling
- Implement preemptible/spot VMs for batch workloads
- Use sole-tenant nodes for compliance requirements

## Cloud Run Best Practices
- Set appropriate CPU and memory limits
- Configure min/max instances for scaling
- Use Cloud Run services for HTTP workloads
- Use Cloud Run jobs for batch processing
- Implement proper startup and liveness probes

## GKE Best Practices
- Use Autopilot for simplified operations
- Enable Workload Identity for IAM
- Use node auto-provisioning
- Implement pod security policies
- Enable GKE Sandbox for untrusted workloads

---

# Data Services

## BigQuery Best Practices
```sql
-- GOOD: Partitioned and clustered table
CREATE TABLE dataset.events
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM source_table;

-- Use partition pruning
SELECT * FROM dataset.events
WHERE DATE(event_timestamp) = '2024-01-01'  -- Scans only one partition

-- Avoid SELECT *
SELECT user_id, event_type, event_timestamp
FROM dataset.events
WHERE DATE(event_timestamp) = '2024-01-01';
```

## BigQuery Checklist
- [ ] Tables partitioned by date/timestamp
- [ ] Clustering on high-cardinality filter columns
- [ ] Authorized views for data access control
- [ ] Slot reservations for predictable performance
- [ ] BI Engine for dashboard acceleration

## Cloud SQL Best Practices
- Enable high availability for production
- Use private IP for secure access
- Enable automated backups with PITR
- Use read replicas for read scaling
- Implement maintenance windows

## Cloud Spanner Best Practices
- Design schema for distributed access patterns
- Use interleaved tables for parent-child relationships
- Avoid hotspots in primary keys
- Use commit timestamps for ordering
- Monitor and adjust node count based on CPU

## Firestore/Datastore Best Practices
- Design for query patterns
- Use composite indexes strategically
- Implement proper data modeling (denormalization)
- Use batch operations for efficiency
- Monitor read/write distribution

---

# Networking

## VPC Best Practices
```
VPC Design:
├── Shared VPC (recommended for multi-project)
│   ├── Host Project
│   │   └── VPC, Subnets, Firewall Rules
│   └── Service Projects
│       └── Resources using shared subnets
├── Private Google Access enabled
├── VPC Flow Logs enabled
└── Cloud NAT for egress
```

## Firewall Rules
```hcl
# GOOD: Specific, tagged firewall rule
resource "google_compute_firewall" "allow_http" {
  name    = "allow-http-to-web"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]  # Only for public-facing
  target_tags   = ["web-server"]

  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}
```

## Private Service Connect
- Use for private access to Google APIs
- Connect to managed services privately
- Reduces exposure to public internet
- Simplifies network architecture

---

# Monitoring & Logging

## Cloud Monitoring Best Practices
- Create custom dashboards for key services
- Use alerting policies with proper notification channels
- Implement uptime checks for external endpoints
- Use SLO monitoring for service reliability
- Create metric-based alerts for anomalies

## Cloud Logging
```
# Required logging configuration
- Admin Activity logs (always on)
- Data Access logs (enable for sensitive resources)
- System Event logs (always on)
- VPC Flow Logs
- Load Balancer logs
```

## Log-Based Metrics
```yaml
# Create metrics from log patterns
resource "google_logging_metric" "error_count" {
  name   = "error_count"
  filter = "severity>=ERROR"
  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
  }
}
```

---

# Cost Optimization

## Principles
- Right-size resources using recommendations
- Use committed use discounts for predictable workloads
- Implement preemptible/spot VMs where appropriate
- Use autoscaling to match demand
- Clean up unused resources

## Cost Checklist
- [ ] Labels applied for cost allocation
- [ ] Budget alerts configured
- [ ] Committed use discounts evaluated
- [ ] Recommender suggestions reviewed
- [ ] Unused resources identified

---

# Security Services

## Required Security Services
- **Security Command Center**: Security posture management
- **Cloud Asset Inventory**: Resource inventory and history
- **VPC Service Controls**: Data exfiltration prevention
- **Binary Authorization**: Container image verification
- **Cloud KMS**: Key management

## VPC Service Controls
```hcl
# Protect sensitive data from exfiltration
resource "google_access_context_manager_service_perimeter" "perimeter" {
  parent = "accessPolicies/${var.access_policy}"
  name   = "accessPolicies/${var.access_policy}/servicePerimeters/secure_perimeter"
  title  = "Secure Perimeter"

  status {
    resources = [
      "projects/${var.project_number}"
    ]
    restricted_services = [
      "bigquery.googleapis.com",
      "storage.googleapis.com"
    ]
  }
}
```

---

# Disaster Recovery

## DR Strategies for GCP
| Strategy | Implementation | RTO | RPO |
|----------|---------------|-----|-----|
| Regional | Multi-zone deployment | Minutes | Near zero |
| Multi-regional | Cross-region replication | Minutes | Minutes |
| Global | Global load balancing | Seconds | Seconds |

## GCP DR Checklist
- [ ] Multi-zone deployment for regional HA
- [ ] Cross-region replication for DR
- [ ] Cloud SQL HA and cross-region replicas
- [ ] GCS dual-region or multi-region buckets
- [ ] Regular DR testing
