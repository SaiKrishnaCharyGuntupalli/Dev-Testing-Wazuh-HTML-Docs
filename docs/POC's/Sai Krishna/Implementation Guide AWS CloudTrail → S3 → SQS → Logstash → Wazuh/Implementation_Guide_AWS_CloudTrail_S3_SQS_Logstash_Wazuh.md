# IMPLEMENTATION GUIDE: AWS CLOUDTRAIL → S3 → SQS → LOGSTASH → WAZUH


## 1. AWS ACTIVITY LOGS COLLECTION

In AWS, the equivalent of Azure Activity Logs is:

### 1.1 AWS CloudTrail

CloudTrail records:

- Console logins
- IAM changes
- EC2 operations
- S3 access
- API calls
- Security group modifications
- Resource creation/deletion

### 1.2 Where to View AWS Activity Logs

#### AWS Console Path

Go to:

```
AWS Console
→ CloudTrail
→ Event History
```

Official AWS page: [AWS CloudTrail](https://aws.amazon.com/cloudtrail/?utm_source=chatgpt.com)

You can immediately verify:

- Who did what
- Which service
- Timestamp
- Source IP
- API action

Example:

```
TerminateInstances
CreateUser
DeleteBucket
AuthorizeSecurityGroupIngress
```

---

## 2. AWS LOG COLLECTION APPROACHES

![Image 1](<../../../assets/images/POC's/Sai krishna/Implementation_Guide_AWS_CloudTrail_S3_SQS_Logstash_Wazuh/image1.png>)

Your pipeline:

```
AWS CloudTrail
↓
S3 Bucket
↓
SQS Queue
↓
Logstash
↓
Wazuh Manager
↓
Alerts
```

is:

- scalable
- reliable
- near real-time
- cheaper than Kinesis
- easier than Lambda architectures

Since you already:

- have Logstash VM
- have Wazuh Manager VM
- already connected Logstash → Wazuh
- already know custom rules

you are already 70% done.

Now you only need AWS-side ingestion.

### 2.1 Why We Use Each AWS Service

| Service | Why Needed | What It Does |
|---|---|---|
| CloudTrail | Generate activity logs | Captures AWS API/account actions |
| S3 Bucket | Store logs | Keeps JSON log files |
| SQS Queue | Notify new logs | Sends message when new log file arrives |
| IAM Policy/Role | Permissions | Allows services to talk securely |
| Logstash S3/SQS Input | Read logs | Pulls CloudTrail logs |
| Wazuh Manager | Alerting | Matches rules and generates alerts |

---

## 3. WHY CLOUDTRAIL WHEN EVENT HISTORY ALREADY EXISTS?

Very important question.

### 3.1 What Is Event History?

In AWS:

```
AWS Console
→ CloudTrail
→ Event History
```

shows recent AWS activity logs.

BUT Event History is ONLY:

- for viewing in AWS console
- limited retention
- not designed for SIEM ingestion
- not streamable
- not scalable

Think of it like:

AWS built-in viewer

NOT a log delivery pipeline.

### 3.2 Limitations of Event History

| Limitation | Problem |
|---|---|
| Only 90 days | No long retention |
| Console viewing only | Cannot integrate properly |
| No SQS integration | No streaming |
| No S3 export pipeline | SIEM difficult |
| No automation | Not enterprise-ready |

---

## 4. WHY CLOUDTRAIL SERVICE IS REQUIRED

CloudTrail is the ACTUAL logging service.

CloudTrail does:

- capture API activity
- generate JSON logs
- continuously deliver logs
- integrate with S3
- integrate with CloudWatch
- integrate with EventBridge

Event History is just a viewer.

### 4.1 Simple Analogy

| Component | Real Life Example |
|---|---|
| CloudTrail | CCTV Camera recording |
| S3 | Hard disk storage |
| SQS | Notification bell |
| Event History | Small monitor showing recent events |

### 4.2 How CloudTrail Works

When someone creates an EC2 VM:

```
Create EC2 VM
```

AWS internally calls the API:

```
RunInstances
```

CloudTrail captures this API event and creates a JSON log.

Example:

```json
{
  "eventName": "RunInstances",
  "eventSource": "ec2.amazonaws.com",
  "sourceIPAddress": "1.2.3.4"
}
```

Then CloudTrail sends this log to S3.

### 4.3 Why S3 Bucket?

S3 is STORAGE.

CloudTrail itself does NOT permanently store logs for SIEM use.

So:

```
CloudTrail
↓
Stores JSON files
↓
S3 Bucket
```

### 4.4 What S3 Stores

Files like:

```
AWSLogs/123456789012/CloudTrail/ap-south-1/2026/05/12/file.json.gz
```

Inside file:

```json
{
  "Records": [
    {
      "eventName": "RunInstances"
    }
  ]
}
```

### 4.5 Why SQS?

Without SQS:

- Logstash must scan bucket repeatedly
- expensive
- slower
- inefficient

With SQS:

When a new file arrives:

```
S3
→ sends notification
→ SQS
```

SQS says:

```
Hey Logstash,
new file arrived:
AWSLogs/file.json.gz
```

This is MUCH better.

---

## 5. HOW LOGSTASH KNOWS?

Logstash continuously polls SQS.

Example:

```
Logstash
→ checks queue every few seconds
```

When a message arrives:

1. reads SQS notification
2. extracts bucket + filename
3. downloads actual file from S3
4. parses JSON logs
5. forwards to Wazuh

---

## 6. COMPLETE DATA FLOW

```
User/API Action in AWS
↓
CloudTrail captures event
↓
CloudTrail writes JSON log file to S3
↓
S3 sends notification to SQS
↓
Logstash polls SQS
↓
Logstash downloads JSON log from S3
↓
Logstash parses records
↓
Logstash forwards logs to Wazuh
↓
Wazuh rules trigger alerts
```

---

## 7. APPROXIMATE COST

### 7.1 CloudTrail

#### Management Events

First copy:

```
FREE
```

Usually enough for SOC learning.

### 7.2 S3 Cost

Very cheap.

Example:

- few MB/day logs
- testing lab

Approx:

| Usage | Approx Cost |
|---|---|
| Per day | almost ₹0 |
| Per month | ₹5–₹50 |

### 7.3 SQS Cost

Very cheap.

AWS gives:

```
1 million requests free
```

For SOC lab:

```
almost free
```

---

## 8. TOTAL APPROX LAB COST

| Service | Approx Monthly |
|---|---|
| CloudTrail | Free |
| S3 | ₹5–₹50 |
| SQS | ₹0–₹10 |

Total:

```
Usually below ₹100/month
```

for testing environment.

---

## 9. STEP 1: CREATE CLOUDTRAIL

Go to AWS Console:

```
AWS Console
→ CloudTrail
→ Trails
→ Create trail
```

Official AWS page: [AWS CloudTrail Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html?utm_source=chatgpt.com)

### 9.1 Important Settings

#### Trail Name

Example:

```
soc-cloudtrail
```

#### Apply Trail to All Regions

ENABLE THIS

Why?

- Collects logs from every AWS region
- SOC best practice

#### Management Events

ENABLE

Includes:

- IAM changes
- EC2 operations
- API calls
- Security operations

#### Data Events

Optional initially.

These are:

- S3 object access
- Lambda execution

Can become costly.

Start WITHOUT them.

---

## 10. STEP 2: CREATE S3 BUCKET

CloudTrail needs a bucket.

If you add a new S3 bucket while creating CloudTrail, skip this step.

Go to:

```
AWS Console
→ S3
→ Create bucket
```

Example bucket:

```
company-cloudtrail-logs
```

### 10.1 Important Settings

#### Block Public Access

KEEP ENABLED

Never expose CloudTrail logs publicly.

#### Versioning

Optional.

#### Encryption

Enable:

```
SSE-S3
```

Good practice.

### 10.2 What Happens Now?

CloudTrail starts writing logs like:

```json
{
  "Records": [
    {
      "eventTime": "2026-05-11T10:20:00Z",
      "eventName": "CreateUser",
      "userIdentity": {
        "userName": "admin"
      },
      "sourceIPAddress": "1.2.3.4"
    }
  ]
}
```

Stored in S3 path like:

```
s3://company-cloudtrail-logs/AWSLogs/account-id/CloudTrail/
```

---

## 11. STEP 3: CREATE SQS QUEUE

Why needed?

Without SQS:

- Logstash must continuously scan S3
- inefficient
- slower

With SQS:

- S3 sends notification instantly
- Logstash knows exactly which file arrived

This is enterprise best practice.

Go to:

```
AWS Console
→ SQS
→ Create Queue
```

### 11.1 Queue Type

Choose:

```
Standard Queue
```

Why?

- cheaper
- scalable
- enough for logs

FIFO not needed.

Example queue name:

```
cloudtrail-log-notifications
```

---

## 12. STEP 4: CONNECT S3 → SQS

Go to your bucket → *Properties → Event notifications → Create event notification*

```
Name: cloudtrail-new-file-notify
Prefix: AWSLogs/
Suffix: .json.gz
Event types: s3:ObjectCreated:* (all create events)
Destination: SQS queue → select cloudtrail-log-notifications
Save changes
```

**Why:** This is the "bridge" that connects S3 and SQS. Every time CloudTrail writes a new `.json.gz` file to S3, S3 automatically sends a notification message to your SQS queue. The message contains the exact S3 path of the new file. Logstash reads this message to know exactly which file to download — no guessing, no full bucket scanning.

### **Challenge:**

When I am trying to save the changes, it actually gives an error while saving, and the error is as follows:

```
Before Amazon S3 can publish messages to a destination, you must grant the Amazon S3
principal the necessary permissions to call the relevant API to publish messages to an
SNS topic, an SQS queue, or a Lambda function. [Learn more](https://docs.aws.amazon.com/console/s3/notifications)
```

and when I am trying to save the changes it gives the below error:

```
Unknown Error
An unexpected error occurred. Try again later. If the error persists, contact
AWS Support for assistance (https://aws.amazon.com/contact-us).

API response: Unable to validate the following destination configurations
```

The reason is that **S3 cannot send messages to your SQS queue yet because the SQS queue's access policy doesn't have the correct permission** — or the policy you added has a mismatch in the ARN values. S3 validates the permission *before* saving the notification, and if it fails the validation, it throws that "Unable to validate destination" error.

**Root cause:** S3 tests the SQS queue permission *before* saving the event notification. If SQS does not have a policy that explicitly allows `s3.amazonaws.com` to call `sqs:SendMessage` on that exact queue ARN, the validation fails and you get "Unable to validate destination configurations."

The most common reasons are: the SQS access policy was never added, the ARN in the policy has a typo (wrong account ID, wrong region, wrong queue name), or the policy was added to the wrong queue.

### **Solution:**

#### Step 1 — Find Your Exact Values (Do This First)

Before writing the policy you need three exact values. Get them from the AWS Console right now.

**Value 1 — Your AWS Account ID (12-digit number)**

Click your account name top-right in AWS Console → it shows your Account ID. Example: `123456789012`

**Value 2 — Your SQS Queue ARN**

Go to SQS → click your queue `cloudtrail-s3-notify` → scroll down to the **Details** section → copy the **ARN** field exactly.
Format: `arn:aws:sqs:us-east-1:123456789012:cloudtrail-s3-notify`

**Value 3 — Your S3 Bucket ARN**

Go to S3 → click your bucket → Properties tab → scroll to **Bucket ARN** → copy it.
Format: `arn:aws:s3:::soc-cloudtrail-logs-123456789012`

#### Step 2 — Fix the SQS Access Policy with the Correct JSON

1. Go to SQS → your queue → Access policy tab → Edit. (AWS Console → SQS → click `cloudtrail-s3-notify` → scroll down to the Access policy section → click Edit)

2. Replace the entire policy with this — substituting your real values. Replace the three placeholders: `YOUR-REGION`, `YOUR-ACCOUNT-ID`, `YOUR-BUCKET-NAME`:

```json
{
  "Version": "2012-10-17",
  "Id": "S3ToSQSPolicy",
  "Statement": [
    {
      "Sid": "AllowS3ToSendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:YOUR-REGION:YOUR-ACCOUNT-ID:cloudtrail-s3-notify",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::YOUR-BUCKET-NAME"
        },
        "StringEquals": {
          "aws:SourceAccount": "YOUR-ACCOUNT-ID"
        }
      }
    }
  ]
}
```

Click Save — the policy saves immediately. No restart needed. The permission takes effect within seconds.

#### Step 3 — Now Go Back to S3 and Add the Event Notification

1. S3 → your bucket → Properties → Event notifications → Create event notification. Fill in exactly:

```
Event name: cloudtrail-new-file-notify
Prefix: AWSLogs/
Suffix: .json.gz
Event types: check s3:ObjectCreated (All)
Destination: select SQS queue
SQS queue: choose cloudtrail-log-notifications
Click Save changes
```

---

## 13. STEP 5: CREATE IAM USER FOR LOGSTASH

If you already know about AWS and IAM, create the user yourself. But if you don't have good knowledge, then take any DevOps engineer's help while creating it.

Logstash must access:

- SQS
- S3

Create a dedicated IAM user.

Go to:

```
IAM
→ Users
→ Create User
```

Example:

```
logstash-cloudtrail-reader
```

### 13.1 Attach Policy

Use a custom least-privilege policy.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::company-cloudtrail-logs",
        "arn:aws:s3:::company-cloudtrail-logs/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:REGION:ACCOUNT_ID:cloudtrail-log-notifications"
    }
  ]
}
```

Update with correct values.

### 13.2 Save Access Keys

You will get:

- Access Key
- Secret Key

Used in Logstash.

---

## 14. STEP 6: INSTALL REQUIRED LOGSTASH PLUGIN

On the Logstash VM:

```bash
sudo /usr/share/logstash/bin/logstash-plugin install logstash-input-s3-sns-sqs
```

OR use:

```bash
sudo /usr/share/logstash/bin/logstash-plugin install logstash-input-s3
```

But the SQS-based plugin is better.

### 14.1 How Logstash Works Here

#### SQS Does Not Contain Logs

Very important.

SQS only contains a notification:

```json
{
  "bucket": "company-cloudtrail-logs",
  "key": "AWSLogs/file.json.gz"
}
```

Then Logstash:

1. reads SQS message
2. downloads actual file from S3
3. extracts JSON
4. parses CloudTrail events

---


## 15. STEP 7: LOGSTASH CONFIGURATION

##### Meaning:

I already have Logstash, and I have an existing config file with Azure Activity Log processing logic. Now I am just updating my existing config file logic with AWS logic, and my fully updated file is as follows:

### 15.1 Full Updated Config File

**Logstash config file:**

```ruby
# ============================================
# INPUT: Receive logs FROM Filebeat
# ============================================
input {
  beats {
    port => 5044
    host => "0.0.0.0"
  }
}

# INPUT 2: Read Azure Activity Logs FROM Event Hub
input {
  azure_event_hubs {
    event_hub_connections => [
      "Endpoint=sb://XXXXXXXXX.servicebus.windows.net/;SharedAccessKeyName=logstash-reader;SharedAccessKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=;EntityPath=activity-logs-event-hub"
    ]
    threads => 4
    decorate_events => true
    consumer_group => "wazuh-consumer"
    storage_connection => "DefaultEndpointsProtocol=https;AccountName=XXXXXXXXXXXXXXX;AccountKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==;EndpointSuffix=core.windows.net"
  }
}

# INPUT 3: Read AWS CloudTrail Logs FROM S3 → SQS
input {
  s3snssqs {
    region => "ap-south-1"

    queue => "cloudtrail-log-notifications"

    access_key_id => "YOUR_AWS_ACCESS_KEY"
    secret_access_key => "YOUR_AWS_SECRET_KEY"

    codec => "json"
  }
}

# ============================================
# FILTER: Transform/parse logs
# ============================================
filter {

  # ══════════════════════════════════════════
  # AWS CLOUDTRAIL HANDLER
  # ══════════════════════════════════════════
  #
  # CloudTrail logs arrive in this format:
  #
  # {
  #   "Records": [
  #     {
  #       "eventName": "RunInstances"
  #     }
  #   ]
  # }
  #
  # We:
  # 1. Split each record
  # 2. Extract important fields
  # 3. Build clean message for Wazuh
  #
  # ══════════════════════════════════════════

  if [Records] {

    # Split Records[] array into individual events
    split {
      field => "Records"
    }

    ruby {
      code => '
        begin
          rec = event.get("Records")

          if rec

            event.set("aws_eventName", rec["eventName"])
            event.set("aws_sourceIP", rec["sourceIPAddress"])
            event.set("aws_region", rec["awsRegion"])
            event.set("aws_eventSource", rec["eventSource"])

            # Extract username safely
            aws_user = "unknown"

            begin
              aws_user = rec.dig("userIdentity", "userName")

              if aws_user.nil?
                aws_user = rec.dig("userIdentity", "arn")
              end
            rescue
            end

            event.set("aws_user", aws_user)

            # Extract EC2 instance ID if available
            resource = "unknown"

            begin
              items = rec.dig("responseElements", "instancesSet", "items")

              if items.is_a?(Array) && items.length > 0
                resource = items[0]["instanceId"]
              end
            rescue
            end

            event.set("aws_resource", resource)

          end
        rescue => e
          event.set("aws_parse_error", e.message)
        end
      '
    }

    # Build clean one-line message
    mutate {
      replace => {
        "message" => "AWSCloudTrail eventName=%{aws_eventName} user=%{aws_user} sourceIP=%{aws_sourceIP} region=%{aws_region} resource=%{aws_resource}"
      }
    }

    # Add AWS tags
    mutate {
      add_tag => [ "aws_cloudtrail" ]

      add_field => {
        "log_source" => "aws_cloudtrail"
      }
    }
  }

  # ══════════════════════════════════════════
  # AZURE ACTIVITY LOG HANDLER
  #
  # HOW WE DETECT AZURE EVENTS:
  # Azure Event Hub messages always start with {"records":
  # Filebeat messages NEVER start with {"records":
  # This is the most reliable detection method based on
  # your actual log output.
  # ══════════════════════════════════════════
  if [message] =~ /^\s*\{.*"records"/ {

    # STEP A: Parse the outer JSON wrapper
    # Input: '{"records": [{ "operationName": "...", ... }]}'
    # Output: azure_raw.records = array of log objects
    json {
      source => "message"
      target => "azure_raw"
      skip_on_invalid_json => true
    }

    # STEP B: Extract fields from first record in records[]
    # Based on your REAL logs, these fields exist directly:
    # operationName, resultType, callerIpAddress, category,
    # resourceId, level, correlationId
    # "caller" does NOT exist directly — it's inside identity.claims
    # "resourceGroupName" does NOT exist — extract from resourceId
    ruby {
      code => "
        begin
          raw = event.get('[azure_raw][records]')
          rec = nil
          if raw.is_a?(Array) && raw.length > 0
            rec = raw[0]
          elsif raw.is_a?(Hash)
            rec = raw
          end

          if rec
            event.set('azure_activity', rec)

            # Extract caller email from deep nested path
            # identity -> claims -> emailaddress
            email_key = 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress'
            caller_email = nil
            begin
              claims = rec.dig('identity', 'claims')
              caller_email = claims[email_key] if claims
            rescue
            end
            event.set('[azure_activity][caller]', caller_email || rec.fetch('callerIpAddress', 'unknown'))

            # Extract resource group from resourceId
            # /SUBSCRIPTIONS/.../RESOURCEGROUPS/ACTIVITY_LOGS/...
            begin
              rid = rec['resourceId'] || ''
              rg_match = rid.match(/RESOURCEGROUPS\\/([^\\/]+)\\//i)
              event.set('[azure_activity][resourceGroupName]', rg_match ? rg_match[1] : 'unknown')
            rescue
              event.set('[azure_activity][resourceGroupName]', 'unknown')
            end

            # Extract VM name from resourceId (last segment)
            begin
              rid = rec['resourceId'] || ''
              vm_match = rid.match(/VIRTUALMACHINES\\/([^\\/]+)$/i)
              event.set('[azure_activity][resourceName]', vm_match ? vm_match[1] : 'unknown')
            rescue
              event.set('[azure_activity][resourceName]', 'unknown')
            end
          end
        rescue => e
          event.set('[azure_parse_error]', e.message)
        end
      "
    }

    # STEP C: Build the clean one-line message Wazuh will match
    # This REPLACES the huge JSON blob with a clean readable line
    mutate {
      replace => {
        "message" => "AzureActivityLog operationName=%{[azure_activity][operationName]} caller=%{[azure_activity][caller]} resultType=%{[azure_activity][resultType]} resourceGroup=%{[azure_activity][resourceGroupName]} resource=%{[azure_activity][resourceName]} level=%{[azure_activity][level]} category=%{[azure_activity][category]} callerIp=%{[azure_activity][callerIpAddress]}"
      }
    }

    # STEP D: Tag so you can identify in dashboard
    mutate {
      add_tag => [ "azure_activity_log" ]
      add_field => { "log_source" => "azure_eventhub" }
    }

  }
  # ══════════════════════════════════════════
  # END OF AZURE HANDLER
  # All original logic below is COMPLETELY UNCHANGED
  # ══════════════════════════════════════════

  # 1. CLEAN: Remove ANSI colors immediately
  mutate {
    gsub => [ "message", "\u001B\[[0-9;]*[mK]", "" ]
  }

  # 2. EXTRACT: A flexible Grok
  grok {
    match => { "message" => "(?:.*INFO )?%{GREEDYDATA:clean_msg}" }
  }

  # 3. REPLACE: Put the clean log into the main message field
  if [clean_msg] {
    mutate {
      replace => { "message" => "%{clean_msg}" }
    }
  }

  # --- TENANT TAGGING --- (UNCHANGED)
  mutate {
    replace => { "message" => "%{message}-tenantA" }
  }

  if [host][name] {
    mutate {
      update => { "[host][name]" => "%{[host][name]}-tenantA" }
    }
  }
  # -----------------------------

  # 4. FINAL POLISH: Remove newlines and carriage returns
  mutate {
    gsub => [
      "message", "\n", " ",
      "message", "\r", ""
    ]
  }

  # 5. TRUNCATE: Prevent oversized packets from being dropped
  ruby {
    code => "
      msg = event.get('message')
      if msg && msg.length > 2000
        event.set('message', msg[0..1999])
      end
    "
  }
}

# ============================================
# OUTPUT: Forward logs TO Wazuh Manager (UNCHANGED)
# ============================================
output {
  tcp {
    host => "XXX.XXX.XX.43"
    port => 9065
    codec => line { format => "%{message}" }
  }

  stdout {
    codec => rubydebug
  }
}
```

After updating:

### 15.2 Validate Config

```bash
sudo /usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/yourfile.conf
```

### 15.3 Restart Logstash

```bash
sudo systemctl restart logstash
```

### 15.4 Monitor Logs

```bash
sudo journalctl -u logstash -f
```

---

## 16. DATA FORMAT AT EACH STAGE

| Stage | Format |
|---|---|
| CloudTrail | JSON |
| S3 Object | JSON.GZ |
| SQS | Notification JSON |
| Logstash after parsing | Structured JSON |
| Wazuh | Decoded event |

---

## 17. DOES WAZUH HAVE DEFAULT AWS RULES?

### 17.1 Short Answer

YES — partially.

Official integration page: [Wazuh AWS Integration](https://documentation.wazuh.com/current/cloud-security/amazon/services/index.html?utm_source=chatgpt.com)

### 17.2 But Important Reality

Default rules:

- are limited
- may not cover your custom scenarios
- often require tuning
- sometimes expect specific decoder formats

Most SOC engineers create custom rules.

Especially for:

- CloudTrail
- GuardDuty
- IAM abuse
- suspicious API activity

### 17.3 Best Practice

Use:

- default rules for baseline
- custom rules for important detections

---

## 18. STEP 8: CUSTOM DECODER

CloudTrail is already JSON.

Usually the Wazuh JSON decoder handles it automatically.

Example log:

```json
{
  "eventName": "DeleteBucket",
  "sourceIPAddress": "1.2.3.4"
}
```

No custom decoder needed initially.

---

## 19. STEP 9: CUSTOM RULES

Example rules:

```xml
<group name="aws,cloudtrail,">

<!-- AWS EC2 VM CREATED -->
<rule id="200100" level="8">
  <match>AWSCloudTrail eventName=RunInstances</match>
  <description>AWS EC2 Instance Created</description>
  <group>aws_ec2_create,</group>
</rule>

<!-- AWS EC2 VM TERMINATED -->
<rule id="200101" level="10">
  <match>AWSCloudTrail eventName=TerminateInstances</match>
  <description>AWS EC2 Instance Deleted</description>
  <group>aws_ec2_delete,</group>
</rule>

</group>
```

### **Challenge:**

When I restart Logstash it shows running when I check its status, but when I check with the `journalctl` command it is silently failing. It is due to the following reason:

Your IAM policy is MISSING:

```
"sqs:GetQueueUrl"
```

VERY IMPORTANT.

The plugin internally calls `GetQueueUrl` before accessing the queue.

Without this permission, AWS returns `NonExistentQueue` even though the queue exists.

#### 19.1 This Is the Root Cause

Add this permission:

```
"sqs:GetQueueUrl"
```

### **Solution:**

I updated the IAM user policy as follows:

#### 19.2 Updated IAM Policy

Use this FULL policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::aws-cloudtrail-logs-XXXXXXXXXXXX-0839549c",
        "arn:aws:s3:::aws-cloudtrail-logs-XXXXXXXXXXXX-0839549c/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:GetQueueUrl",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:ap-south-1:XXXXXXXXXXXX:cloudtrail-log-notifications"
    }
  ]
}
```

After updating, I again restarted my Logstash. Now Logstash is running and I can't see any failure error when I run the `journalctl` command in the Logstash terminal.

### **Challenge:**

After resolving the previous challenge, I am again facing another challenge saying "TypeError: no implicit conversion of nil into String", and this error is explained clearly as follows:

**Main error:**

```
TypeError: no implicit conversion of nil into String
```

**Now the REAL root cause is:**

The plugin expects SNS-wrapped SQS messages, BUT your SQS receives direct S3 notifications.

This is a KNOWN behavior/problem with `logstash-input-s3-sns-sqs`.

#### 19.3 Very Important Understanding

Your plugin name itself says:

```
s3-sns-sqs
```

Meaning the plugin was originally designed for:

```
S3
→ SNS
→ SQS
→ Logstash
```

NOT always reliable for:

```
S3
→ SQS directly
```

#### 19.4 What Is Happening Internally

Your SQS message body is:

```json
{
  "Records": [...]
}
```

But the plugin internally expects something like:

```json
{
  "Message": "{\"Records\":[...]}"
}
```

That structure happens when using SNS between S3 and SQS.

So the plugin tries `JSON.parse(nil)`, which produces `no implicit conversion of nil into String`.

#### 19.5 This Is the Exact Root Cause

Your architecture currently:

```
CloudTrail
→ S3
→ SQS
→ Logstash
```

But the plugin expects:

```
CloudTrail
→ S3
→ SNS
→ SQS
→ Logstash
```

#### 19.6 This Is Why Messages Keep Reappearing

Because:

- Logstash FAILS processing the message
- the message is NOT deleted from the queue
- the SQS visibility timeout expires
- the message becomes visible again

So Logstash retries forever.

That is EXACTLY why you see:

- the same warning repeatedly
- the same messages reappearing

### **Solution:**

#### 19.7 Add SNS Between S3 and SQS

This is the STANDARD AWS production architecture.

Most SOC engineers use:

```
CloudTrail
→ S3
→ SNS
→ SQS
→ Logstash
→ Wazuh
```

#### 19.8 Why SNS Is Required

SNS wraps the message into the expected structure. The plugin correctly parses the SNS notification.

#### 19.9 Your Current Flow

Current:

```
S3 notification
→ direct JSON
```

Plugin fails.

#### 19.10 Required Flow

Required:

```
S3 notification
→ SNS message
→ SQS message
```

Plugin succeeds.

#### 19.11 Exact Fix Steps

##### Step 1: Create SNS Topic

Go:

```
AWS Console
→ SNS
→ Topics
→ Create Topic
```

Type: `Standard`

Name: `cloudtrail-topic`

##### Step 2: Subscribe SQS to SNS

Go:

```
SNS
→ cloudtrail-topic
→ Create Subscription
```

Protocol: `Amazon SQS`

Endpoint: `cloudtrail-log-notifications`

##### Step 3: Update SQS Access Policy

VERY IMPORTANT.

Remove the old policy which is for S3. Add a new policy which is for SNS.

Add SNS permission:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow-SNS-SendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:ap-south-1:XXXXXXXXXXXX:cloudtrail-log-notifications",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:ap-south-1:XXXXXXXXXXXX:cloudtrail-topic"
        }
      }
    }
  ]
}
```

##### Step 4: Change S3 Event Notification

**REMOVE old:** S3 → SQS

**Create NEW:** S3 → SNS

**Go:**

```
S3
→ Bucket
→ Properties
→ Event Notifications
```

Destination: `SNS Topic`

**Choose:** `cloudtrail-topic`

##### Step 5: Purge Queue Again

VERY IMPORTANT.

```
SQS
→ Purge Queue
```

Removes old malformed messages.

##### Step 6: Restart Logstash

```bash
sudo systemctl restart logstash
```

#### 19.12 Expected Result

Now Logstash should:

- poll SQS
- parse the SNS wrapper
- extract the S3 object path
- download the `.json.gz`
- process CloudTrail logs

WITHOUT errors.

#### 19.13 Result

Now Logstash is processing AWS CloudTrail logs, and I can verify those in the Logstash terminal using the `sudo journalctl -u logstash -f | grep "AWSCloudTrail"` command, and the result looks as follows:

![Image 2](<../../../assets/images/POC's/Sai krishna/Implementation_Guide_AWS_CloudTrail_S3_SQS_Logstash_Wazuh/image2.png>)

For these different types of event logs, I extended my custom rule file with more rules to trigger alerts for many event logs.

Wazuh Manager custom rules to trigger alerts for incoming AWS CloudTrail logs are as follows.

**Custom rules:**

```xml
<group name="aws,cloudtrail,">

<!-- ===================================================== -->
<!-- EXISTING RULES -->
<!-- ===================================================== -->

<!-- AWS EC2 VM CREATED -->
<rule id="200100" level="8">
  <match>AWSCloudTrail eventName=RunInstances</match>
  <description>AWS EC2 Instance Created</description>
  <group>aws_ec2_create,</group>
</rule>

<!-- AWS EC2 VM TERMINATED -->
<rule id="200101" level="10">
  <match>AWSCloudTrail eventName=TerminateInstances</match>
  <description>AWS EC2 Instance Deleted</description>
  <group>aws_ec2_delete,</group>
</rule>

<!-- ===================================================== -->
<!-- EC2 INSTANCE / INFRASTRUCTURE ACTIVITY -->
<!-- ===================================================== -->

<!-- EC2 Describe Instances -->
<rule id="200102" level="3">
  <match>AWSCloudTrail eventName=DescribeInstances</match>
  <description>AWS EC2 DescribeInstances API Called</description>
  <group>aws_ec2_describe,</group>
</rule>

<!-- EC2 Instance Status Check -->
<rule id="200103" level="3">
  <match>AWSCloudTrail eventName=DescribeInstanceStatus</match>
  <description>AWS EC2 Instance Status Checked</description>
  <group>aws_ec2_status,</group>
</rule>

<!-- EC2 Instance Attribute Viewed -->
<rule id="200104" level="4">
  <match>AWSCloudTrail eventName=DescribeInstanceAttribute</match>
  <description>AWS EC2 Instance Attribute Accessed</description>
  <group>aws_ec2_attribute,</group>
</rule>

<!-- EC2 Network Interfaces -->
<rule id="200105" level="4">
  <match>AWSCloudTrail eventName=DescribeNetworkInterfaces</match>
  <description>AWS Network Interfaces Viewed</description>
  <group>aws_network_interface,</group>
</rule>

<!-- Elastic IP Addresses -->
<rule id="200106" level="4">
  <match>AWSCloudTrail eventName=DescribeAddresses</match>
  <description>AWS Elastic IP Addresses Viewed</description>
  <group>aws_ec2_addresses,</group>
</rule>

<!-- Root Volume Tasks -->
<rule id="200107" level="5">
  <match>AWSCloudTrail eventName=DescribeReplaceRootVolumeTasks</match>
  <description>AWS Root Volume Replacement Tasks Viewed</description>
  <group>aws_root_volume,</group>
</rule>

<!-- EC2 Credit Specifications -->
<rule id="200108" level="4">
  <match>AWSCloudTrail eventName=DescribeInstanceCreditSpecifications</match>
  <description>AWS EC2 Credit Specifications Viewed</description>
  <group>aws_ec2_credit,</group>
</rule>

<!-- ===================================================== -->
<!-- CLOUDWATCH / MONITORING -->
<!-- ===================================================== -->

<!-- CloudWatch Metric Filters -->
<rule id="200109" level="5">
  <match>AWSCloudTrail eventName=DescribeMetricFilters</match>
  <description>AWS CloudWatch Metric Filters Viewed</description>
  <group>aws_cloudwatch_metric,</group>
</rule>

<!-- CloudWatch Alarms -->
<rule id="200110" level="5">
  <match>AWSCloudTrail eventName=DescribeAlarms</match>
  <description>AWS CloudWatch Alarms Viewed</description>
  <group>aws_cloudwatch_alarm,</group>
</rule>

<!-- ===================================================== -->
<!-- CLOUDTRAIL / EVENT HISTORY -->
<!-- ===================================================== -->

<!-- CloudTrail Event Lookup -->
<rule id="200111" level="6">
  <match>AWSCloudTrail eventName=LookupEvents</match>
  <description>AWS CloudTrail Event History Accessed</description>
  <group>aws_cloudtrail_lookup,</group>
</rule>

<!-- AWS Events Viewed -->
<rule id="200112" level="4">
  <match>AWSCloudTrail eventName=DescribeEvents</match>
  <description>AWS Events Information Viewed</description>
  <group>aws_events_describe,</group>
</rule>

<!-- Managed Notification Events -->
<rule id="200113" level="4">
  <match>AWSCloudTrail eventName=ListManagedNotificationEvents</match>
  <description>AWS Managed Notification Events Listed</description>
  <group>aws_notifications,</group>
</rule>

<!-- ===================================================== -->
<!-- IAM / ROLE / AUTHENTICATION -->
<!-- ===================================================== -->

<!-- IAM Assume Role -->
<rule id="200114" level="9">
  <match>AWSCloudTrail eventName=AssumeRole</match>
  <description>AWS IAM Role Assumed</description>
  <group>aws_iam_assumerole,authentication_success,</group>
</rule>

<!-- Enrollment Status -->
<rule id="200115" level="5">
  <match>AWSCloudTrail eventName=GetEnrollmentStatus</match>
  <description>AWS Enrollment Status Checked</description>
  <group>aws_enrollment,</group>
</rule>

<!-- ===================================================== -->
<!-- SOURCE CODE / GIT -->
<!-- ===================================================== -->

<!-- Git Push Activity -->
<rule id="200116" level="7">
  <match>AWSCloudTrail eventName=GitPush</match>
  <description>AWS Git Repository Push Activity Detected</description>
  <group>aws_git_push,</group>
</rule>

<!-- AWS CloudTrail Activity -->
<rule id="200117" level="12">
  <match>AWSCloudTrail eventName=DeleteTrail</match>
  <description>WARNING: AWS CloudTrail Logging Disabled</description>
  <group>aws_cloudtrail_disable,compliance,critical,</group>
</rule>

</group>
```

After adding the rules, save and reload if you added them from the UI.

Now alerts are generating for the incoming AWS CloudTrail logs for the different events with different rule IDs and levels in Wazuh Manager, and those generated alerts look the same as follows in the `alerts.json` file.

![Image 3](<../../../assets/images/POC's/Sai krishna/Implementation_Guide_AWS_CloudTrail_S3_SQS_Logstash_Wazuh/image3.png>)

In the Wazuh UI it looks like the following:

![Image 4](<../../../assets/images/POC's/Sai krishna/Implementation_Guide_AWS_CloudTrail_S3_SQS_Logstash_Wazuh/image4.png>)

---

## 20. ADDING METADATA TO THE AWS ACTIVITY LOGS

Update the existing Logstash config file with the following metadata:

```ruby
# ==========================================
# AWS METADATA ENRICHMENT
# ==========================================

if "aws_cloudtrail" in [tags] {

  mutate {
    add_field => {
      "source_platform" => "aws"
      "event_category" => "cloud_security"
      "environment" => "poc"
      "tenant" => "aws_poc"
      "non_agent_id" => "8001"
      "device_type" => "aws_cloudtrail"
      "cloud_provider" => "aws"
    }
  }

  mutate {
    replace => {
      "message" => "%{message} tenant=aws_poc non_agent_id=8001 source_platform=aws event_category=cloud_security environment=poc device_type=aws_cloudtrail"
    }
  }

}
```

After adding the metadata section to the existing Logstash config file, restart Logstash to apply the changes using the following command:

```bash
systemctl restart logstash
```

### 20.1 Custom Decoder

Create the custom decoder in Wazuh to extract the metadata from the incoming log as fields.

You can create a custom decoder either from the Wazuh browser UI or the `local_decoder` file.

**Custom decoder:**

```xml
<decoder name="aws_cloudtrail_metadata">
  <prematch>tenant=aws_poc</prematch>
  <regex>tenant=(\S+) non_agent_id=(\S+) source_platform=(\S+) event_category=(\S+) environment=(\S+) device_type=(\S+) cloud_provider=(\S+)</regex>
  <order>tenant, non_agent_id, source_platform, event_category, environment, device_type, cloud_provider</order>
</decoder>
```

After adding the custom decoder, restart the Wazuh Manager, or reload if you added it from the browser, to apply the changes.

Now the metadata will be displayed as fields, the same as shown in the image below.

![Image 5](<../../../assets/images/POC's/Sai krishna/Implementation_Guide_AWS_CloudTrail_S3_SQS_Logstash_Wazuh/image5.png>)

---

