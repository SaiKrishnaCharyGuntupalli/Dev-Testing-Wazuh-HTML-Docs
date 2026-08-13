# Microsoft 365 Audit Log Integration with Wazuh SIEM Complete Implementation Guide

## Chapter 1 – Introduction

### 1.1 Project Objective

The objective of this project is to integrate Microsoft 365 Audit Logs with the Wazuh Security Information and Event Management (SIEM) platform. This integration enables centralized monitoring of Microsoft 365 activities such as Outlook, Teams, SharePoint, OneDrive, and Azure Active Directory. Once integrated, every audit activity performed within the Microsoft 365 organization is collected by Wazuh, analyzed using built-in detection rules, and displayed in the Wazuh Dashboard. This allows security analysts to monitor user activities, investigate incidents, and build organization-specific security detections.

### 1.2 Scope

1. Microsoft 365 Tenant Configuration
2. Microsoft Entra ID Configuration
3. Azure App Registration
4. OAuth Authentication
5. Microsoft Graph API Permissions
6. Office 365 Management Activity API Permissions
7. Microsoft Purview Audit Configuration
8. Wazuh Office365 Module Configuration
9. Dashboard Verification
10. Built-in Rule Processing
11. Custom Rule Creation
12. Troubleshooting
13. Validation

### 1.3 Technologies Used

![Technologies Used](<../../../assets/images/POC's/Yaseen/Microsoft 365 to Wazuh Integration Complete Imp and Config/Technologies Used.png>)

## 1.4 End-to-End Architecture

![End-to-End Architecture](<../../../assets/images/POC's/Yaseen/Microsoft 365 to Wazuh Integration Complete Imp and Config/End-to-End Architecture .png>)

**Next Step**
The next step is to understand the Microsoft services used in this integration.

---

## Chapter 2 – Microsoft 365 Overview

Before configuring the integration, it is important to understand the Microsoft services involved.

### 2.1 Microsoft 365

**Definition:** Microsoft 365 is Microsoft's cloud-based productivity platform that provides applications and services such as Outlook, Teams, SharePoint, OneDrive, Exchange Online, and Microsoft Entra ID.

**Why is it required?**
Microsoft 365 generates audit logs whenever users perform activities such as:

1. Sending emails
2. Receiving emails
3. Joining Teams meetings
4. Uploading files
5. Downloading files
6. Sharing files
7. Logging into Azure AD

these audit logs are the primary data source for Wazuh monitoring.

### 2.2 Microsoft Entra ID

**Definition:** Microsoft Entra ID (formerly Azure Active Directory) is Microsoft's Identity and Access Management (IAM) service.

**Why is it required?**
It authenticates users and applications.
In this project, Wazuh authenticates using an Azure Application Registration created inside Microsoft Entra ID.

### 2.3 Microsoft Purview

**Definition:** Microsoft Purview is Microsoft's Compliance and Governance platform.

**Why is it required?**
Purview stores all Microsoft 365 audit events.

Examples include:

1. Outlook activities
2. Teams meetings
3. SharePoint uploads
4. OneDrive sharing
5. Azure AD logins

Without Purview Audit, Microsoft 365 audit logs cannot be retrieved.

### 2.4 Microsoft Graph API

**Definition:** Microsoft Graph API is Microsoft's unified REST API used to access Microsoft cloud resources such as users, groups, calendars, devices, and directory information.

**Why did we configure it?**
The Azure App Registration requires Microsoft Graph permissions for organization and directory access.

**Permissions added:**

1. AuditLog.Read.All
2. Directory.Read.All
3. Organization.Read.All

**Important Note:** Although Microsoft Graph permissions were configured, Wazuh does not retrieve audit logs using Microsoft Graph in this implementation.

### 2.5 Office 365 Management Activity API

**Definition:** The Office 365 Management Activity API is a REST API that provides Microsoft 365 audit logs.

**Why is it required?**
Wazuh uses this API to retrieve audit logs from:

1. Outlook
2. Teams
3. SharePoint
4. OneDrive
5. Azure AD

Without this API, Wazuh cannot collect Microsoft 365 audit events.

### 2.6 Difference Between Microsoft Graph API and Office 365 Management API

| Microsoft Graph API | Office 365 Management API |
|---|---|
| Accesses Microsoft resources | Retrieves audit logs |
| User information | Security events |
| Groups | Compliance logs |
| Calendars | Activity logs |
| Devices | Audit subscriptions |

---

## Chapter 3 – Prerequisites

Beforewe starting the implementation,we have to ensure the following requirements are available.

| Requirement | Purpose |
|---|---|
| Microsoft 365 Organization Account | Access Microsoft 365 services |
| Azure Subscription | Required for Azure resources |
| Microsoft Entra ID | Authentication |
| Microsoft 365 Administrator | Grant API permissions |
| Wazuh Manager | Receive audit logs |
| Docker | Wazuh deployment |
| Microsoft Purview | Audit logs |
| Internet Connectivity | API communication |

**Next Step**
The next step is to log in to the Microsoft 365 Portal.

---

## Chapter 4 – Microsoft 365 Portal Setup

**Purpose:** This portal provides access to all Microsoft 365 applications.

> **Step 1**
> Open
> [https://www.office.com](https://www.office.com?utm_source=chatgpt.com)
>
> **Step 2**
> Click
> Sign In
>
> **Step 3**
> Login using
> yaseen@vidyayug.com
>
> **Step 4**
> After successful login you will see
> 1. Outlook
> 2. Teams
> 3. OneDrive
> 4. SharePoint
> 5. Word
> 6. Excel
> 7. PowerPoint

This portal provides access to all Microsoft 365 applications.

**Why is this required?**
The organization account is used throughout the implementation for authentication, Microsoft Purview access, and generating Microsoft 365 audit logs.

**Next Step:**
Open Azure Portal to configure Microsoft Entra ID.

---

## Chapter 5 – Azure Portal & Microsoft Entra ID

**Purpose:** Azure Portal is used to create an application that Wazuh will use for authentication.

> **Step 1**
> Open
> [https://portal.azure.com](https://portal.azure.com)
>
> **Step 2**
> Login using
> yaseen@vidyayug.com
>
> **Step 3**
> Search
> Microsoft Entra ID
> Open it.
>
> **Why?**
> Microsoft Entra ID manages users, applications, authentication, and permissions.

The Wazuh application will be created here.

**Next Step**
Create an Azure Application Registration.

---

## Chapter 6 – Azure App Registration

**Definition:** An App Registration creates an identity for an external application. In this project, Wazuh acts as the external application.

**Why is it required?**
Without App Registration:

1. Wazuh cannot authenticate.
2. Microsoft will reject API requests.

### Navigation path

> Microsoft Entra ID
> ↓
> Applications
> ↓
> App Registrations
> ↓
> New Registration

### Fill

Application Name

```
M365-Wazuh-Integration
```

Supported Account Type

```
Accounts in this organizational directory only
```

Redirect URI

Leave Empty

Click

```
Register
```

### Output

Application created successfully.

### Next Step

Copy the Tenant ID and Client ID.

---

## Chapter 7 – Tenant ID, Client ID & Client Secret

### Tenant ID

**Definition:** Unique identifier of the Microsoft organization.

**Why required?** Wazuh needs to know which Microsoft tenant to connect to.

### Client ID

**Definition:** Unique identifier of the Azure Application.

**Why required?** Microsoft identifies which application is requesting access.

**Copy**

> Application (Client) ID
>
> Directory (Tenant) ID
>
> Store both safely.

### Client Secret

**Definition:** Acts like the password of the Azure Application.

**Navigation**

> Certificates & Secrets
> ↓
> New Client Secret
>
> Description
> Microsoft365-Wazuh
>
> Expiry
> 24 Months
>
> Click
> Add
>
> Immediately copy
> Secret Value

Important: Microsoft displays the Secret Value only once.

### Why required?

During OAuth authentication, Microsoft validates:

1. Tenant ID
2. Client ID
3. Client Secret

Only then is an Access Token generated.

### Next Step

Configure the required API permissions.

---

## Chapter 8 – API Permissions

**Purpose:** API permissions define what resources the Azure Application is allowed to access. Authentication alone is not sufficient. The application must also be authorized to read Microsoft 365 audit data.

> ### Microsoft Graph API Permissions
>
> Navigate to:
>
> API Permissions
> ↓
> Add Permission
> ↓
> Microsoft Graph
> ↓
> Application Permissions
>
> Add:
> 1. AuditLog.Read.All
> 2. Directory.Read.All
> 3. Organization.Read.All
>
> Click Add Permissions.

Why?
These permissions allow the application to read organizational and directory information required by the Azure application.

Note: These permissions were configured as part of the App Registration, but the audit log collection in this project is performed through the Office 365 Management Activity API.

### Office 365 Management Activity API Permissions

Again navigate to:

> API Permissions
> ↓
> Add Permission
> ↓
> Office 365 Management APIs
> ↓
> Application Permissions
>
> Add:
> 1. ActivityFeed.Read
> 2. ActivityFeed.ReadDlp
> 3. ServiceHealth.Read
>
> Click
> Add Permissions.

Why?
These permissions allow Wazuh to retrieve Microsoft 365 audit logs through the Office 365 Management Activity API.

**Next Step**
Request an administrator to grant tenant-wide consent for the configured permissions.

---

## Chapter 9 – Admin Consent

**Definition:** Admin Consent authorizes the Azure Application to use the configured API permissions across the Microsoft 365 organization.

**Why is it required?**
Application permissions affect the entire tenant and therefore require approval from a Microsoft 365 Administrator.

### Procedure

Go to:

API Permissions

Click:

Grant Admin Consent

**Issue Faced**
Initially, the permissions were Not Granted, so Wazuh could authenticate but could not access Microsoft 365 audit subscriptions. This resulted in permission-related errors.

### Resolution

my senior granted Admin Consent. After approval, the permission status changed to: Granted for Vidyayug

### Next Step

Verify that Microsoft Purview Audit is accessible.

---

## Chapter 10 – Microsoft Purview Audit

**Definition:** Microsoft Purview is Microsoft's compliance platform. Its Audit service stores organization-wide Microsoft 365 audit events.

### Why is it required?

Wazuh collects Microsoft 365 activities that are recorded in Microsoft Purview Audit.

### Procedure:

> Open:
> [https://purview.microsoft.com](https://purview.microsoft.com)
>
> Navigate to:
> Solutions
> ↓
> Audit

### Issue Faced

Initially, the Audit page was not accessible because your account did not have the required permissions.

### Resolution

So I contacted our senior,. He verified that Purview Audit was already enabled for the tenant and granted your account access. You also confirmed that granting your account access would not impact other organization users.

### Propagation Delay

Microsoft documents that permission changes may take 1–24 hours to propagate. After waiting, you verified that the Audit page became accessible.

### Verification

Generate a few Microsoft 365 activities, such as:

1. Join a Microsoft Teams meeting.
2. Send or receive an Outlook email.
3. Upload a file to SharePoint.
4. Upload or modify a file in OneDrive.

These activities should appear in the Purview Audit Search page after processing.

### Next Step

The next phase is to configure the Wazuh Office365 module so it can authenticate with Azure, retrieve Microsoft 365 audit logs through the Office 365 Management Activity API, and begin forwarding those events into the Wazuh analysis engine.

---

## Chapter 11 – Wazuh Office365 Module Configuration

### 11.1 Purpose

After configuring Microsoft 365, Azure, API permissions, and Microsoft Purview, the next step is to configure Wazuh so that it can authenticate with Microsoft 365 and retrieve audit logs automatically. The Office365 module is a built-in Wazuh module responsible for communicating with the Office 365 Management Activity API. **Without this configuration, Wazuh will not collect any Microsoft 365 audit logs.**

### 11.2 What is ossec.conf?

**Definition:** ossec.conf is the primary configuration file of the Wazuh Manager.

It controls all Wazuh modules, including:

1. Log collection
2. Active response
3. Vulnerability detection
4. File Integrity Monitoring
5. Office365 Integration
6. Syscheck
7. Rootcheck
8. Remote Agents

Location:

```
/var/ossec/etc/ossec.conf
```

### 11.3 Why Modify ossec.conf?

By default, Wazuh does not know:

1. Which Microsoft 365 tenant to connect to.
2. Which Azure application to use.
3. Which API credentials to use.
4. Which audit subscriptions should be collected.

Therefore, these details must be configured manually.

### 11.4 Office365 Module Configuration

Open:

```
docker exec -it single-node-wazuh.manager-1 bash
```

Open configuration:

```
vi /var/ossec/etc/ossec.conf
```

Add:

```xml
<office365>
<enabled>yes</enabled>
<interval>5m</interval>
<curl_max_size>1M</curl_max_size>
<api_auth>
<tenant_id>YOUR_TENANT_ID</tenant_id>
<client_id>YOUR_CLIENT_ID</client_id>
<client_secret>YOUR_CLIENT_SECRET</client_secret>
</api_auth>
<subscriptions>
<subscription>Audit.General</subscription>
<subscription>Audit.Exchange</subscription>
<subscription>Audit.SharePoint</subscription>
<subscription>Audit.AzureActiveDirectory</subscription>
</subscriptions>
</office365>
```

### 11.5 Explanation of Every Tag

**`<enabled>yes</enabled>`**

Enables the Office365 module.

Without this tag the module never starts.

**`<interval>5m</interval>`**

Checks Microsoft every five minutes for new audit logs.

**`<curl_max_size>1M</curl_max_size>`**

Maximum response size Wazuh accepts from Microsoft.

**`<tenant_id>`**

Identifies the Microsoft organization.

**`<client_id>`**

Identifies the Azure App Registration.

**`<client_secret>`**

Password of the Azure Application.
Used during OAuth authentication.

**`<subscriptions>`**

Defines which Microsoft365 audit categories Wazuh should collect.

**Audit.General**
General Microsoft365 activities

Examples

- Teams
- OneDrive
- SharePoint

**Audit.Exchange**
Exchange Online

Examples

- Email
- Mailbox
- Outlook

**Audit.SharePoint**
SharePoint operations

Examples

- Upload
- Download
- Sharing

**Audit.AzureActiveDirectory**
Azure AD

Examples

- Login
- Authentication
- User management

**Output**
After saving the configuration, Wazuh now knows how to authenticate with Microsoft 365 and which audit logs to retrieve.

**Next Step**
Restart the Wazuh Manager to load the Office365 module.

---

## Chapter 12 – Restarting Wazuh

Restart Wazuh:

```
/var/ossec/bin/wazuh-control restart
```

### Purpose

Reloads:

1. Office365 module
2. Rules
3. Decoders
4. Configuration

### Verification

Check:

```
grep -i office365 /var/ossec/logs/ossec.log
```

Expected output:
Office365 Module Started

### Next Step

Verify Microsoft authentication using OAuth.

---

## Chapter 13 – OAuth Authentication

### Definition

OAuth 2.0 is Microsoft's authentication mechanism.

Instead of sending the Client Secret every time, Azure generates a temporary Access Token.

### Authentication Flow

> Wazuh
> ↓
> Azure Entra ID
> ↓
> Tenant ID
> ↓
> Client ID
> ↓
> Client Secret
> ↓
> OAuth Access Token
> ↓
> Office365 Management API
> ↓
> Audit Logs
> ↓
> Wazuh

### Why is OAuth Required?

Microsoft never allows direct access to APIs using only a password.

Instead:

> Client Secret
> ↓
> Azure
> ↓
> Access Token
> ↓
> API Access

### Generate Access Token

```bash
curl -X POST \
"https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" \
-d "client_id=<CLIENT_ID>" \
-d "client_secret=<CLIENT_SECRET>" \
-d "scope=https://manage.office.com/.default" \
-d "grant_type=client_credentials"
```

### Expected Response

```json
{
"access_token":"eyJ..."
}
```

### What is Access Token?

Temporary credential generated by Azure.

Valid for approximately one hour.

### Next Step

Verify Microsoft365 subscriptions.

---

## Chapter 14 – REST API Testing

### Purpose

Before relying on Wazuh, verify that Microsoft APIs are accessible.

### Step 1

Generate token.

### Step 2

List subscriptions.

```bash
curl -H "Authorization: Bearer $TOKEN" \
"https://manage.office.com/api/v1.0/<TENANT_ID>/activity/feed/subscriptions/list"
```

Expected:

Audit.General

Audit.Exchange

Audit.SharePoint

Audit.AzureActiveDirectory

### Why?

Confirms:

- Authentication successful.
- Permissions correct.
- API accessible.

### Next Step

Allow Wazuh to retrieve audit logs.

---

## Chapter 15 – Office365 Module Internal Working

### Internal Processing Flow

> Microsoft365
> ↓
> Purview Audit
> ↓
> Office365 Management API
> ↓
> Office365 Module
> ↓
> Analysisd
> ↓
> JSON Decoder
> ↓
> Rules
> ↓
> Alerts
> ↓
> Dashboard

### Explanation

The Office365 module periodically calls Microsoft APIs.

Microsoft returns audit logs in JSON format.

These logs are forwarded to Wazuh Analysisd for processing.

### Next Step

Understand how Wazuh decodes Microsoft365 JSON logs.

---

## Chapter 16 – Decoders

### Definition

A decoder converts raw logs into structured fields.

Without decoding, Wazuh cannot understand log contents.

### Decoder Location

```
/var/ossec/ruleset/decoders/
```

### Built-in JSON Decoder

Wazuh already contains a JSON decoder.

No manual configuration is required.

### Example Raw Log

```json
{
"Operation":"MeetingParticipantDetail",
"Workload":"MicrosoftTeams"
}
```

### Decoder Output

```
office365.Operation
MeetingParticipantDetail
office365.Workload
MicrosoftTeams
```

### Why Automatically Selected?

The Office365 module sends JSON logs.
Wazuh automatically detects JSON format and uses the built-in JSON decoder.

No custom decoder was created during this implementation.

### Next Step

Process decoded logs using Wazuh Rules.

---

## Chapter 17 – Wazuh Rules

### Definition

Rules are the decision-making engine of Wazuh. After a log is decoded, Wazuh compares it against its built-in and custom rules. If a rule matches the event, Wazuh generates an alert and displays it in the Wazuh Dashboard.

### Built-in Rules

Wazuh includes approximately 8456 built-in rules covering multiple log sources, including:

1. Linux
2. Windows
3. Office365
4. AWS
5. Azure
6. Firewalls
7. Suricata
8. Syslog
9. Docker

These rules are maintained by Wazuh and are available immediately after installation.

Location

```
/var/ossec/ruleset/rules/
```

### Office365 Rules File

The Microsoft 365 built-in rules are stored in:

```
/var/ossec/ruleset/rules/0755-office365_rules.xml
```

We inspected this file to understand how Microsoft 365 events are processed before creating our custom rule.

### Important Office365 Rule IDs

![Important Office365 Rule IDs](<../../../assets/images/POC's/Yaseen/Microsoft 365 to Wazuh Integration Complete Imp and Config/Important Office365 Rule IDs.png>)

### Built-in Rule Processing Chain

When an Office365 event arrives, Wazuh processes it in the following order:

91531
(Root Office365 Rule)
↓
91532
(Generic Office365 Rule)
↓
91536 / 91537 / 91555 / 91578
(Built-in Office365 Detection Rules)
↓
100500
(Custom Rule Created by Us)

### Why Did We Use Rule 91532?

Before creating a custom rule, we examined the Office365 rules file and observed that most Office365 detection rules inherit from Rule 91532.

Since our required operation (SharingInvitationCreated) did not already exist as a built-in rule, we used 91532 as the parent rule. This ensures that our custom rule executes only after Wazuh has already verified that the event is a valid Office365 event.

### Rule Engine Flow

> Raw Microsoft365 Log
> ↓
> Office365 Module
> ↓
> JSON Decoder
> ↓
> 8456 Built-in Rules
> ↓
> Matching Rule Found
> ↓
> Alert Generated
> ↓
> Displayed in Wazuh Dashboard

### How We Identified the Parent Rule

**Step 1 – Locate the Office365 Rules File**

```bash
find /var/ossec/ruleset/rules/ -iname "*office365*"
```

**Step 2 – Inspect the Parent Rule**

```bash
grep -B5 -A8 'id="91531"' /var/ossec/ruleset/rules/0755-office365_rules.xml
```

**Step 3 – Inspect the Generic Office365 Rule**

```bash
grep -B5 -A8 'id="91532"' /var/ossec/ruleset/rules/0755-office365_rules.xml
```

**Step 4 – Verify Whether the Required Detection Already Exists**

```bash
grep -rl "SharingInvitationCreated" /var/ossec/ruleset/rules/
```

Result:

No output.
This confirmed that Wazuh did not provide a built-in rule for SharingInvitationCreated, so a custom rule was required.

### Observation

We did not choose Rule 91532 randomly. We first analyzed Wazuh's built-in Office365 rules, understood the parent-child rule hierarchy, and confirmed that no existing rule handled our target operation. Only then did we create Custom Rule 100500, using 91532 as its parent to extend Wazuh's default detection logic.

---

## Chapter 18 – Alerts & Dashboard Verification

### 18.1 Purpose

After Wazuh successfully collects Microsoft 365 audit logs and processes them through the decoder and rule engine, security alerts are generated and displayed in the Wazuh Dashboard. This chapter explains how alerts are created, what information they contain, and how to verify that the integration is working correctly.

### 18.2 Alert Generation Flow

> Microsoft 365 Activity
> │
> ▼
> Office365 Module
> │
> ▼
> Raw JSON Log
> │
> ▼
> JSON Decoder
> │
> ▼
> 8456 Built-in Rules
> │
> ▼
> Matching Rule
> │
> ▼
> Alert Generated
> │
> ▼
> Stored by Wazuh
> │
> ▼
> Indexed
> │
> ▼
> Visible in Wazuh Dashboard

### 18.3 What Does Each Alert Contain?

Every alert generated by Wazuh contains useful security information that helps analysts understand what happened.

| Field | Description | Example |
|---|---|---|
| rule.id | Rule that generated the alert | 100500 |
| rule.level | Alert severity | 10 |
| rule.description | Description of the alert | Office365: External file sharing invitation created - possible data loss risk |
| data.integration | Log source | office365 |
| data.office365.UserId | User who performed the activity | yaseen@vidyayug.com |
| data.office365.Operation | Microsoft 365 operation performed | SharingInvitationCreated |
| data.office365.TargetUserOrGroupName | Target user or external recipient | khanyaseenkhan2001@gmail.com |
| data.office365.Workload | Microsoft 365 service | SharePoint / OneDrive |
| GeoLocation.country_name | Country associated with the activity (if available) | United States |
| timestamp | Time when the event occurred | Jun 30, 2026 @ 17:13:55 |
| rule.groups | Categories assigned to the alert | office365_custom, data_loss, SharePoint |

### 18.4 Dashboard Verification

Open the Wazuh Dashboard.

Navigate to:

Security Events
↓
Discover

Use the following search queries to verify different Microsoft 365 activities.

**Search all Office365 events**

```
data.integration:office365
```

**Search by specific operation**

```
data.office365.Operation:"SharingInvitationCreated"
```

**Search using the custom rule**

```
rule.id:100500
```

**Search by user**

```
data.office365.UserId:"yaseen@vidyayug.com"
```

**Search by Microsoft 365 service**

```
data.office365.Workload:"SharePoint"
```

or

```
data.office365.Workload:"MicrosoftTeams"
```

or

```
data.office365.Workload:"Exchange"
```

### 18.5 Verification Result

If the integration is successful, the Wazuh Dashboard will display:

> Microsoft 365 user activities.
>
> 1. The operation performed (for example, SharingInvitationCreated, MeetingParticipantDetail, MailItemsAccessed).
> 2. The user who performed the activity.
> 3. The workload (SharePoint, Teams, Exchange, OneDrive, Azure AD).
> 4. The rule that matched the event.
> 5. The alert severity level.
> 6. The event timestamp.
>
> This confirms that the Office365 integration is functioning correctly and that Wazuh is successfully monitoring organization-wide Microsoft 365 audit activities.

### Next Step

After verifying that alerts are generated successfully, the next step is to create organization-specific custom rules to detect security scenarios that are not covered by Wazuh's default Office365 rules.

---

## Chapter 19 – Custom Rules

Why Custom Rules?
Built-in rules automatically classify what happened — but they don't know what is risky for your specific organization.

Without custom rule:

File shared externally → rule 91532 fires → level 3 (low, generic)

SOC analyst may MISS it in thousands of events

With custom rule:

File shared externally → rule 100500 fires → level 10 (HIGH SEVERITY)

Immediately visible as DATA LOSS RISK

#### Gap We Found

```bash
grep -rl "SharingInvitationCreated" /var/ossec/ruleset/rules/
```

Returned EMPTY — Wazuh has NO built-in rule for external file sharing

This means a critical data loss event only shows as generic level 3 alert

Custom rule needed to close this gap

#### Custom Rule Location

```
/var/ossec/etc/rules/local_rules.xml
```

Note: Never edit built-in rules in /var/ossec/ruleset/rules/ — always add custom rules in local_rules.xml only.

#### Custom Rule We Added

```xml
<group name="office365_custom,">
<rule id="100500" level="10">
<if_sid>91532</if_sid>
<field name="office365.Operation" type="osregex">^SharingInvitationCreated$</field>
<description>Office 365: External file sharing invitation created - possible data loss risk.</description>
<group>office365,data_loss,SharePoint,custom,</group>
</rule>
</group>
```

#### Rule Elements Explained

![Rule Elements Explained](<../../../assets/images/POC's/Yaseen/Microsoft 365 to Wazuh Integration Complete Imp and Config/Rule Elements Explained.png>)

#### How to Add the Rule

**Step 1 — Enter Wazuh container**

```bash
docker exec -it single-node-wazuh.manager-1 bash
```

**Step 2 — Add rule to local_rules.xml**

```bash
cat >> /var/ossec/etc/rules/local_rules.xml << 'EOF'
<group name="office365_custom,">
<rule id="100500" level="10">
<if_sid>91532</if_sid>
<field name="office365.Operation" type="osregex">^SharingInvitationCreated$</field>
<description>Office 365: External file sharing invitation created - possible data loss risk.</description>
<group>office365,data_loss,SharePoint,custom,</group>
</rule>
</group>
EOF
```

**Step 3 — Test syntax (no output = no errors)**

```bash
/var/ossec/bin/wazuh-analysisd -t
```

**Step 4 — Restart Wazuh**

```bash
/var/ossec/bin/wazuh-control restart
```

#### Other Custom Rules You Can Add

```xml
<!-- Brute Force Login -->
<group name="office365_custom,">
<rule id="100501" level="12" frequency="5" timeframe="120">
<if_matched_sid>91531</if_matched_sid>
<field name="office365.Operation">UserLoginFailed</field>
<description>M365: Multiple failed logins - possible brute force attack</description>
<group>office365,brute_force,authentication,</group>
</rule>
</group>
```

```xml
<!-- Anonymous Public Link Created -->
<group name="office365_custom,">
<rule id="100503" level="12">
<if_sid>91531</if_sid>
<field name="office365.Operation" type="osregex">^AnonymousLinkCreated$</field>
<description>M365: Public anonymous link created - file accessible to anyone on internet</description>
<group>office365,data_loss,critical,</group>
</rule>
</group>
```

```xml
<!-- Admin Role Assigned -->
<group name="office365_custom,">
<rule id="100504" level="12">
<if_sid>91531</if_sid>
<field name="office365.Operation">Add member to role.</field>
<description>M365: Admin role assigned to user - verify authorization</description>
<group>office365,privilege_escalation,</group>
</rule>
</group>
```

```xml
<!-- Mass File Download -->
<group name="office365_custom,">
<rule id="100505" level="14" frequency="20" timeframe="300">
<if_matched_sid>91531</if_matched_sid>
<field name="office365.Operation">FileDownloaded</field>
<description>M365: Mass file download detected - possible data exfiltration</description>
<group>office365,data_exfiltration,critical,</group>
</rule>
</group>
```

---

## Chapter 20 – wazuh-logtest

### Purpose

Tests custom rules without waiting for real events — validates that your rule syntax, parent chain, and field matching are all correct before deploying.

#### How to Run

**Step 1 — Launch logtest**

```bash
/var/ossec/bin/wazuh-logtest
```

**Step 2 — Paste sample JSON (after tool loads)**

```json
{"integration":"office365","office365":{"Operation":"SharingInvitationCreated","RecordType":14,"Workload":"SharePoint"}}
```

**Step 3 — Wait for output (DO NOT press Ctrl+C)**

**Step 4 — Exit when done**

Ctrl+C

#### Why This Exact JSON Structure?

```
"integration":"office365" → triggers root rule 91531
"office365.RecordType":14 → triggers parent rule 91532
"office365.Operation":"SharingInvitationCreated" → triggers your rule 100500
```

All three fields must be present — missing any one breaks the parent chain and Phase 3 never runs.

#### Expected Output We can get exact result see in wazuh

```
**Phase 1: Completed pre-decoding.
full event: '{"integration":"office365",...}'

**Phase 2: Completed decoding.
name: 'json'
integration: 'office365'
office365.Operation: 'SharingInvitationCreated'
office365.RecordType: '14'
office365.Workload: 'SharePoint'

**Phase 3: Completed filtering (rules).
id: '100500'
level: '10'
description: 'Office 365: External file sharing invitation created - possible data loss risk.'
groups: '[office365_custom, office365, data_loss, SharePoint, custom]'
firedtimes: '1'

**Alert to be generated.
```

Phase Meanings

![Phase Meanings](<../../../assets/images/POC's/Yaseen/Microsoft 365 to Wazuh Integration Complete Imp and Config/Phase Meanings.png>)

![Image5](<../../../assets/images/POC's/Yaseen/Microsoft 365 to Wazuh Integration Complete Imp and Config/image5.png>)

### Result

The successful appearance of Microsoft 365 audit events in the Wazuh Dashboard confirms that:

1. The Office365 module was configured correctly.
2. OAuth authentication was successful.
3. The Office365 Management Activity API is accessible.
4. Microsoft 365 audit logs are being collected successfully.
5. Wazuh is decoding the incoming events correctly.
6. The built-in Office365 rules are generating alerts.
7. Custom rules are functioning as expected when their conditions are met.

### Conclusion

The Microsoft 365 to Wazuh integration has been successfully implemented and validated. Wazuh is now capable of continuously monitoring organization-wide Microsoft 365 activities—including Exchange Online, SharePoint Online, OneDrive, Microsoft Teams, and Azure Active Directory—and generating security alerts based on both built-in and organization-specific custom rules.
