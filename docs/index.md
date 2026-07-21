# Wazuh MSSP Multi-Tenant Implementation Guide

<div class="intro-card">
This guide provides a practical, production-oriented walkthrough for implementing a multi-tenant Wazuh MSSP deployment with centralized management, OpenSearch-based indexing, and tenant-aware security controls.
</div>

This documentation is structured to help you move from planning and prerequisites to deployment, security configuration, verification, and reference material in a clear sequence.

## What this guide covers

- Multi-tenant index template and Filebeat routing design
- Linux agent deployment and tenant labeling
- Agent configuration for FIM, syscollector, and tenant-aware log collection
- Document Level Security (DLS) rules for per-tenant index access
- RBAC definitions across OpenSearch and Wazuh APIs
- Microsoft Entra ID (Azure) SSO integration and role assignment
- Verification, testing, and operational reference information

## Who should use this guide

- MSSP engineers deploying Wazuh for multiple tenants
- Security architects designing tenant isolation and RBAC
- Administrators configuring Wazuh with OpenSearch and Azure SSO
- Support teams validating access, security, and monitoring workflows

## Recommended reading order

1. Review the prerequisites first.
2. Create the standard index template and configure Filebeat.
3. Deploy and configure agents with tenant labels.
4. Secure OpenSearch with tenant-specific roles and DLS.
5. Configure Microsoft Entra ID SSO and assign tenant roles.
6. Verify access and review the reference material.

## Navigation

- [Getting_started](01-Getting_started/Getting_started.md)
- [Index Template & Filebeat](02-index-template-filebeat/index_template.md)
- [Agent Deployment](03-agent-deployment/agent_deplyment.md)
- [Agent Configuration](04-agent-configuration/agent_config.md)
- [RBAC](06-rbac/rbac.md)
- [Document Level Security (DLS)](07-dls/dls.md)
- [SSO - Microsoft Entra ID](08-sso-entra/sso.md)
- [Cluster and Index verification](09-Cluster and Index Verification/Cluster and Index Verification.md)
- [Reference](10-reference/reference.md)

