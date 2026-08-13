# Getting Started

<!-- **Author:** Abhinav   -->
**Version:** 1.0  
<!-- **Created Date:** 2026-07-28  
**Updated Date:** 2026-07-28 -->

This section captures the core requirements needed before starting the implementation. It focuses on the environment, access, and preparation items that are essential for a successful deployment.

## 1. Supported Environment

- Linux-based host for the Wazuh manager and supporting services
- OpenSearch deployment compatible with the Wazuh version in use
- Microsoft Entra ID tenant for SSO integration
- Modern browser for Wazuh and OpenSearch Dashboards access

## 2. Required Software

- Docker Engine 20.10+ or Docker Desktop with Linux containers
- Docker Compose plugin or the `docker compose` command
- Wazuh manager and agent versions that are compatible with each other
- OpenSearch and OpenSearch Dashboards versions that match the Wazuh stack
- Python 3.10+ for local MkDocs preview and documentation build

## 3. Hardware Recommendations

- Minimum 8 GB RAM for a proof-of-concept
- Recommended 16 GB RAM or higher depending on tenant count, agent count, log ingestion rate, and retention requirements
- Minimum 4 CPU cores
- Sufficient disk space for log retention and index storage

## 4. Network and Access Requirements

- Open network access for Wazuh manager, API, OpenSearch, and dashboard endpoints
- Access to Microsoft Entra ID for application and role configuration
- SSH access to Linux hosts for agent deployment and configuration
- DNS or host resolution that allows service-to-service communication

## 5. Accounts and Permissions

- Administrator access to the Docker host(s)
- Wazuh manager administrative credentials
- OpenSearch admin or equivalent privileges for security configuration
- Azure tenant administrator or equivalent permissions for Enterprise Application and role assignment

## 6. Recommended Preparation

- Confirm the OpenSearch security plugin and required index settings are enabled
- Verify the Wazuh manager can reach OpenSearch over the expected network path
- Collect tenant names and mapped user groups before starting implementation
- Prepare a service account or admin user for Azure SSO setup

> Note: This guide assumes that the base Wazuh and OpenSearch environment is already available. The focus here is on multi-tenant security and operational configuration rather than the initial installation of the platform.
