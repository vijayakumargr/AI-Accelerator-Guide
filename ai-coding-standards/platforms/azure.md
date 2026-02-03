# Azure Platform Instructions

## General Principles
- **ALWAYS use Infrastructure as Code** (Terraform, Bicep, ARM templates)
- **ALWAYS follow Azure Well-Architected Framework** principles
- **ALWAYS enable diagnostic settings** for logging
- **ALWAYS use Managed Identities** instead of service principals with secrets
- **NEVER hardcode credentials** in code or configuration
- **NEVER use classic (ASM) resources** - use ARM only

---

# Identity & Access Management

## Principles
- **Least Privilege**: Grant only permissions needed for the task
- **Use Managed Identities**: System or user-assigned for Azure resources
- **RBAC**: Use built-in roles where possible, custom roles when needed
- **PIM**: Use Privileged Identity Management for elevated access

## RBAC Assignment
```hcl
# GOOD: Specific role assignment with scope
resource "azurerm_role_assignment" "storage_reader" {
  scope                = azurerm_storage_account.example.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = azurerm_user_assigned_identity.app.principal_id
}

# BAD: Overly permissive at subscription level
resource "azurerm_role_assignment" "contributor" {
  scope                = data.azurerm_subscription.current.id
  role_definition_name = "Contributor"  # Too broad
  principal_id         = azurerm_user_assigned_identity.app.principal_id
}
```

## IAM Checklist
- [ ] Managed Identities used instead of service principal secrets
- [ ] RBAC assignments at appropriate scope (resource, not subscription)
- [ ] Custom roles for fine-grained permissions
- [ ] PIM enabled for privileged roles
- [ ] Regular access reviews configured

---

# Compute Services

## Virtual Machines Best Practices
- Use Azure Hybrid Benefit for Windows/SQL licensing savings
- Enable Azure Backup for data protection
- Use Availability Sets or Availability Zones for HA
- Implement Azure Spot VMs for batch workloads
- Use VM Scale Sets for auto-scaling

## App Service Best Practices
- Use deployment slots for zero-downtime deployments
- Enable Always On for production apps
- Configure auto-scaling rules
- Use managed identities for Azure resource access
- Enable Application Insights for monitoring

## Azure Kubernetes Service (AKS) Best Practices
- Use managed identity for cluster
- Enable Azure AD integration for RBAC
- Use Azure CNI for enterprise networking
- Implement pod-managed identities (AAD Pod Identity v2)
- Enable Azure Policy for Kubernetes

## Azure Functions Best Practices
- Choose appropriate hosting plan (Consumption vs Premium)
- Use durable functions for orchestration
- Implement proper error handling and retries
- Use managed identities for Azure service access
- Enable Application Insights

---

# Data Services

## Azure SQL Best Practices
```sql
-- Enable automatic tuning
ALTER DATABASE [YourDatabase] SET AUTOMATIC_TUNING = AUTO;

-- Use appropriate service tier
-- General Purpose: Most workloads
-- Business Critical: Low-latency, HA requirements
-- Hyperscale: Large databases, rapid scaling
```

## Azure SQL Checklist
- [ ] Transparent Data Encryption (TDE) enabled
- [ ] Azure AD authentication configured
- [ ] Auditing enabled to Log Analytics
- [ ] Geo-replication for DR
- [ ] Elastic pools for multi-tenant scenarios

## Cosmos DB Best Practices
- Choose appropriate API (SQL, MongoDB, Cassandra, etc.)
- Design partition keys for even distribution
- Use provisioned throughput for predictable workloads
- Enable autoscale for variable workloads
- Configure appropriate consistency level

## Azure Storage Best Practices
```hcl
# GOOD: Secure storage account
resource "azurerm_storage_account" "secure" {
  name                     = "securestorage"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  min_tls_version                 = "TLS1_2"
  enable_https_traffic_only       = true
  allow_nested_items_to_be_public = false

  network_rules {
    default_action             = "Deny"
    virtual_network_subnet_ids = [azurerm_subnet.example.id]
  }

  identity {
    type = "SystemAssigned"
  }
}
```

---

# Networking

## Virtual Network Best Practices
```
Hub-Spoke Architecture:
├── Hub VNet
│   ├── Azure Firewall
│   ├── VPN/ExpressRoute Gateway
│   └── Shared Services
└── Spoke VNets (peered to hub)
    ├── Production workloads
    ├── Development workloads
    └── Data workloads
```

## Network Security Groups
```hcl
# GOOD: Specific NSG rule
resource "azurerm_network_security_rule" "allow_https" {
  name                        = "AllowHTTPS"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "443"
  source_address_prefix       = "Internet"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.example.name
  network_security_group_name = azurerm_network_security_group.example.name
}
```

## Private Endpoints
- Use for secure access to PaaS services
- Eliminates public endpoint exposure
- Works with Private DNS Zones
- Required for compliance scenarios

---

# Monitoring & Logging

## Azure Monitor Best Practices
- Create dashboards in Azure Portal or Grafana
- Configure Action Groups for alerting
- Use Log Analytics for centralized logging
- Implement Application Insights for APM
- Use Azure Workbooks for reporting

## Required Diagnostic Settings
```hcl
resource "azurerm_monitor_diagnostic_setting" "example" {
  name                       = "send-to-log-analytics"
  target_resource_id         = azurerm_key_vault.example.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.example.id

  enabled_log {
    category = "AuditEvent"
  }

  metric {
    category = "AllMetrics"
  }
}
```

## Alerting Standards
- Activity Log alerts for control plane operations
- Metric alerts for resource health
- Log alerts for application-specific conditions
- Smart detection for anomaly detection

---

# Cost Optimization

## Principles
- Use Azure Advisor recommendations
- Implement Reserved Instances for predictable workloads
- Use Azure Spot VMs for interruptible workloads
- Right-size resources based on metrics
- Implement auto-scaling

## Cost Checklist
- [ ] Tags applied for cost allocation
- [ ] Budget alerts configured
- [ ] Reserved Instances evaluated
- [ ] Azure Advisor recommendations reviewed
- [ ] Unused resources identified with Azure Orphan Resources

---

# Security Services

## Required Security Services
- **Microsoft Defender for Cloud**: Security posture management
- **Azure Policy**: Governance and compliance
- **Azure Key Vault**: Secrets and key management
- **Azure DDoS Protection**: Network protection
- **Microsoft Sentinel**: SIEM and SOAR

## Azure Policy Examples
```hcl
# Require tags on resources
resource "azurerm_policy_assignment" "require_tags" {
  name                 = "require-tags"
  scope                = data.azurerm_subscription.current.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/require-tag-on-resource-groups"

  parameters = jsonencode({
    tagName = { value = "Environment" }
  })
}
```

## Key Vault Best Practices
- Enable soft delete and purge protection
- Use RBAC for data plane access (not access policies)
- Enable diagnostic logging
- Implement key rotation
- Use managed identities for access

---

# Disaster Recovery

## Azure DR Options
| Strategy | Implementation | RTO | RPO |
|----------|---------------|-----|-----|
| Availability Zones | Zone-redundant deployment | Minutes | Near zero |
| Azure Site Recovery | Cross-region replication | Minutes-Hours | Minutes |
| Geo-Replication | Database/storage replication | Minutes | Seconds-Minutes |

## DR Checklist
- [ ] Availability Zones used for regional HA
- [ ] Azure Site Recovery configured for VMs
- [ ] Geo-redundant storage for critical data
- [ ] SQL geo-replication or auto-failover groups
- [ ] Regular DR drills conducted
- [ ] Recovery procedures documented
