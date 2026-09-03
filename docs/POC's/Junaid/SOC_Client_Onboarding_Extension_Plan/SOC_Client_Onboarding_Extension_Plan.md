# SOC CLIENT ONBOARDING PLAN


## PURPOSE

Define a standardized onboarding process for SOC client integrations and evaluate the value of grouping in the onboarding lifecycle.

---

## REAL CUSTOMER SERVICES

### 1. Pre-requisite

a. Number of endpoints

b. Type of endpoints: Linux, Windows, Mac, Docker, VMs, Cloud logs: Azure logs, GCP logs, AWS, Database logs, Server, Application logs.

c. Data volume expectations

d. Networking requirement for accessing the endpoints (public with protected access/private)

e. Subscription/Licensing limits (TBD)

&nbsp;&nbsp;&nbsp;&nbsp;i. Wazuh Indexer Indices

&nbsp;&nbsp;&nbsp;&nbsp;ii. RBAC roles/default

&nbsp;&nbsp;&nbsp;&nbsp;iii. Dashboards

&nbsp;&nbsp;&nbsp;&nbsp;iv. Wazuh groups

&nbsp;&nbsp;&nbsp;&nbsp;v. wazuh/Logstash routes preference

&nbsp;&nbsp;&nbsp;&nbsp;vi. Decoders logics/Alert rules

f. Retention policy: standard (defined by CISO/custom as per the client requirement)

g. Tenant name

h. Customer ID

i. Region

j. Compliance requirements

k. Backup strategy

l. Dedicated SOC platform/sharable SOC platform

&nbsp;&nbsp;&nbsp;&nbsp;i. Inform sharable platform guidelines/limitations

&nbsp;&nbsp;&nbsp;&nbsp;ii. In terms of complete isolation (share the isolation scope)

m. Check SSO requirement for platform access

&nbsp;&nbsp;&nbsp;&nbsp;i. If SSO requested (provide RBACK role details)

n. Access privileges for RBAC

&nbsp;&nbsp;&nbsp;&nbsp;i. Users/members data collection to access the platform

&nbsp;&nbsp;&nbsp;&nbsp;ii. User wise role assignments (role assignment)

o. SIEM (TBD)

p. SOAR (TBD)

q. Ticketing system (TBD)

r. Point of contact/Contact details

s. Agreement of start date/ Total agreement period/Tenure

t. Alert reporting style (Email or slack or platform)

u. Report selection (On demand/frequency based)

v. Onboarding confirmation/Access handover

w. Maintenance/support (TBD)

**Current selection: Windows**

i. Regular/Default monitoring service applicable [OS level, security level, Terminal, etc]

ii. Optional (On-Client request): Sysmon, External firewall, Active response, etc.

### 2. Tenant Registration/Organization

a. Application: Tenant register with all the details collected above

b. Enroll a new Tenant as per above information collected from the client

c. Prepare backend engineering checklist or raise a ticket

d. Send request/ticket to backend Engineering team to setup

### 3. Begin Platform (MSSP) Setup

a. Template Creation

b. Tenant specific Index creation

c. Update Filebeat configuration to accommodate alerts routing

d. Endpoint/Agent/Non-agent deployment

e. Verify connectivity

f. Confirm agent health

g. RBAC

h. Index patterns creation

i. Platform validations

### 4. RBAC Permission & User Matrix

a. User/member details

b. User-wise role assignment

c. User/team-wise permitted actions

d. User/team-wise resource access scope

e. Permission approval/owner

### 5. Resource Ownership & Mapping

a. Tenant-to-resource mapping

b. Team-to-resource mapping

c. Resource ownership for agents, rules, decoders, and lists

d. Cross-tenant resource isolation requirements

### 6. Rules, Decoders & CDB Lists

a. Custom rule file names and ownership

b. Custom decoder file names and ownership

c. CDB list names and ownership

d. Read/write/modify permissions for each resource

e. Resource-level access requirements

### 7. Agent-Level Access & Group Mapping

a. Agent ID and hostname mapping

b. Agent-to-tenant mapping

c. Agent-to-group mapping

d. Individual agent access requirements

e. Agent group-level access requirements

f. Multiple-group membership requirements

### 8. Index, Dashboard & Tenant Isolation

a. Tenant-specific index requirements

b. Index pattern mapping

c. Dashboard access mapping

d. Cross-tenant index/data visibility restrictions

e. Define complete isolation scope for shared SOC platform

### 9. RBAC & Isolation Validation

a. Low-privilege test account

b. Allowed-access validation

c. Denied-access validation

d. Cross-tenant access validation

e. Rule/decoder/list access validation

f. Agent/group access validation

g. Final RBAC approval and access handover

### 10. Wazuh Grouping Benefits in Client Onboarding

a. Centralized agent configuration management.

b. Bulk deployment of agent.conf to multiple endpoints

c. Faster onboarding of new agents.

d. Grouping by operating system, environment, and application.

e. Simplified configuration updates and maintenance.

f. Supports multiple group membership for a single agent.

---

## 1. STANDARDIZED SOC CLIENT ONBOARDING PROCESS

**A. Agent Enrollment:** Validate client requirements, onboard endpoints, verify connectivity, confirm agent health.

**B. Template Creation:** Create standard log templates, naming conventions, field mappings, parsing rules.

**C. Index Creation:** Create Elasticsearch/Kibana indexes, retention policies, lifecycle management and validation.

**D. Filebeat Changes at Manager Level:** Configure Filebeat centrally, update modules, validate forwarding and monitoring.

**E. RBAC:** Define SOC analyst, engineer and admin roles; implement least-privilege access and approval workflow.

---

## 2. ONBOARDING CHECKLIST

| Step | Activity | Owner | Status |
|---|---|---|---|
| Requirement Review | Collect client scope and log sources | SOC/PM | Pending |
| Agent Enrollment | Deploy and validate agents | Engineering | |
| Template Creation | Create log templates | Engineering | |
| Index Creation | Configure indexes | Platform Team | |
| Filebeat Changes | Manager-level updates | Platform Team | |
| RBAC Setup | Role provisioning | Security Admin | |
| Validation | End-to-end testing | SOC Team | |
| Go-Live | Production sign-off | Customer & SOC | |

---

## 3. GROUPING ASSESSMENT AND RECOMMENDATIONS

Potential benefits of implementing grouping:

- Standardized onboarding packages for similar clients.
- Reduced manual configuration effort through reusable templates.
- Easier RBAC management by applying permissions to groups rather than individuals.
- Simplified Filebeat and agent deployment management.
- Improved scalability and auditability.

**Recommendation:** Implement grouping for clients with similar log sources, compliance requirements, and onboarding patterns. This can reduce deployment effort, improve consistency, and accelerate onboarding time while maintaining governance.

---
