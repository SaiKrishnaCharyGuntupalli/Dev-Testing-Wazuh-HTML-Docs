# Wazuh Windows Agent Deployment and Security Detection POC

## 1\. POC Objective

This Proof of Concept validates that a Windows endpoint can be onboarded into Wazuh and that security-relevant activity on the endpoint is detected and displayed to the SOC team.

The end-to-end flow is:

Windows Endpoint  
→ Wazuh Agent  
→ Wazuh Manager  
→ Wazuh Alert Rules  
→ Filebeat  
→ Wazuh Indexer  
→ Wazuh Dashboard / SOC Analyst

The POC validates:

- Successful Windows Wazuh Agent installation and registration.
- Collection of Windows Security and System telemetry.
- Detection of authentication failures, account changes, administrator-group changes, scheduled tasks, file modifications, and registry persistence activity.
- Alert generation in the Wazuh Manager.
- Visibility of alerts in the Wazuh Dashboard.

## 2\. Prerequisites

| Component               | Requirement                                                                  |
| ----------------------- | ---------------------------------------------------------------------------- |
| Wazuh Dashboard         | Accessible at <https://<IP-address>>   or any available wazuh server                                     |
| Wazuh Manager           | Running and reachable from the Windows endpoint                              |
| Windows endpoint        | Local Administrator access available                                         |
| Network                 | Windows endpoint can reach Wazuh Manager on TCP port 1514 and TCP port 1515  |
| Wazuh Docker deployment | Manager container name: single-node-wazuh.manager-1                          |
| Test environment        | Perform all tests only on an approved lab or non-production Windows endpoint |

Note: The Wazuh Dashboard may show a browser certificate warning when it uses a self-signed certificate. Confirm the certificate with the Wazuh administrator before proceeding.

## 3\. Step 1 - Access the Wazuh Dashboard



- Open a browser.
- Navigate to:

<https://<IP-address>>

- Log in using the Wazuh Dashboard credentials.
- Navigate to:

Agents → Deploy new agent → Windows

- Confirm that the manager IP address shown in the deployment command is:

  <IP-address>

**Expected result:** The Dashboard is accessible and ready to receive the Windows agent.  
When you click on windows, and add the details such as  
1.server IP address:&lt;manager_ip&gt;  
2.agent name:&lt;agnet \_name&gt;  
Then the appropriate commands will be shown execute those accordingly  
as shown below

## 4\. Step 2 - Install the Wazuh Agent on Windows

**Where:** Windows endpoint  
**Run as:** PowerShell Administrator  
**Purpose:** Download and install the Wazuh Agent, then configure it to communicate with the Wazuh Manager.  
<br/>

Open **PowerShell as Administrator** and run:

Invoke-WebRequest -Uri "<https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi>" \`  
\-OutFile "\$env:TEMP\\wazuh-agent-4.14.4-1.msi"  
<br/>msiexec.exe /i "\$env:TEMP\\wazuh-agent-4.14.4-1.msi" /q \`  
WAZUH_MANAGER="<IP-address>" \`  
WAZUH_AGENT_NAME="\$env:COMPUTERNAME"

**What this does:**

- Downloads the Wazuh Windows Agent installer.
- Installs the agent silently.
- Configures the agent to communicate with the Wazuh Manager at <IP-address>.
- Uses the Windows computer name as the Wazuh Agent name.

**Expected result:** The Wazuh Agent is installed on the Windows endpoint.

## 5\. Step 3 - Start and Validate the Wazuh Agent

**Where:** Windows endpoint  
**Run as:** PowerShell Administrator  
**Purpose:** Start the Wazuh service and confirm that it is running.

Run:

Start-Service WazuhSvc  
Get-Service -Name WazuhSvc

If the service is already running, PowerShell may show that it has already been started. This is normal.

To verify the agent connection logs, run:

Get-Content "C:\\Program Files (x86)\\ossec-agent\\ossec.log" -Tail 50

**Expected result:**

Status : Running

Then return to the Wazuh Dashboard:

Agents → Agents

The Windows endpoint should appear as **Active**.

## 6\. Step 4 - Enable Required Windows Audit Policies

**Where:** Windows endpoint  
**Run as:** Command Prompt Administrator  
**Purpose:** Enable Windows logging required for Wazuh to detect authentication, account-management, group-management, and process-creation activity.

Open **Command Prompt as Administrator** and run:

auditpol /set /subcategory:"Logon" /success:enable /failure:enable  
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable  
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable  
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable

Enable command-line capture for process creation events:

reg add "HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies\\System\\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f

Verify the audit configuration:

auditpol /get /category:\*

**What this does:**

- Enables failed and successful logon logging.
- Enables local user account creation and deletion logging.
- Enables administrator-group membership change logging.
- Enables process creation logging, including command-line arguments.

**Expected result:** Windows begins generating Security Event Logs that Wazuh can collect and analyze.

## 7\. Step 5 - Validate Failed Logon Detection

**Where:** Windows endpoint and a separate approved test machine  
**Purpose:** Generate Windows failed authentication events and validate that Wazuh detects them.

Use a controlled test account that does not exist or enter an intentionally incorrect password when connecting to the Windows endpoint from a separate machine.

For example, from another Windows test machine:

net use \\\\WINDOWS-ENDPOINT-IP\\IPC\$ /user:fakeadmin WrongPassword123!

Replace WINDOWS-ENDPOINT-IP with the IP address of the monitored Windows endpoint.

Repeat the command several times.

On the monitored Windows endpoint, validate that failed logon events were generated:

Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10

**What this does:**

- Generates Windows Event ID 4625, which represents a failed logon.
- Sends the event to the Wazuh Manager through the Wazuh Agent.
- Allows Wazuh to generate authentication-failure alerts and, depending on rules and thresholds, brute-force correlation alerts.

**Expected result:** Authentication failure alerts appear in the Wazuh Dashboard.

## 8\. Step 6 - Validate Local User and Administrator Group Monitoring

**Where:** Windows endpoint  
**Run as:** Command Prompt Administrator  
**Purpose:** Simulate a common attacker technique: creating a local account and adding it to the Administrators group.

Run:

net user wazuh_test_user Password123! /add  
net localgroup Administrators wazuh_test_user /add  
net localgroup Administrators wazuh_test_user /delete  
net user wazuh_test_user /delete

**What this does:**

- Creates a temporary local user.
- Adds the user to the local Administrators group.
- Removes the user from the Administrators group.
- Deletes the temporary user.

**Expected Windows events:**

| Activity                                     | Typical Windows Event ID |
| -------------------------------------------- | ------------------------ |
| Local user created                           | 4720                     |
| User added to local Administrators group     | 4732                     |
| User removed from local Administrators group | 4733                     |
| Local user deleted                           | 4726                     |

**Expected result:** Wazuh generates account-management and privileged-group modification alerts.

## 9\. Step 7 - Validate Process Creation and Scheduled Task Detection

**Where:** Windows endpoint  
**Run as:** Command Prompt Administrator  
**Purpose:** Simulate scheduled-task persistence, a technique commonly used by attackers to run programs automatically.

Create a harmless scheduled task:

schtasks /create /tn "WazuhTestTask" /tr "cmd.exe /c echo WazuhPOC" /sc ONSTART /f

Verify that the task exists:

schtasks /query /tn "WazuhTestTask"

Remove the task after validation:

schtasks /delete /tn "WazuhTestTask" /f

**What this does:**

- Creates a harmless scheduled task that runs at system startup.
- Generates Windows Task Scheduler and process-related telemetry.
- Allows Wazuh to identify possible persistence-related activity.

**Expected result:** Wazuh generates scheduled-task or process-creation alerts, depending on the enabled Wazuh rules and telemetry source.

## 10\. Step 8 - Validate PowerShell Command-Line Visibility

**Where:** Windows endpoint  
**Run as:** PowerShell Administrator  
**Purpose:** Confirm that Wazuh can receive process execution telemetry, including PowerShell command lines.

Run this harmless command:

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Write-Output 'Wazuh PowerShell POC Test'"

Then verify process creation events locally:

Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} -MaxEvents 10

**What this does:**

- Generates a Windows process creation event.
- Confirms that command-line auditing is enabled.
- Helps validate Wazuh visibility into PowerShell activity.

**Expected result:** A process-creation event is visible in Windows Security logs and can be searched in Wazuh.

Note: A generic PowerShell command does not always generate a high-severity Wazuh alert. Detection severity depends on the command, Wazuh ruleset, Sysmon configuration, and any custom detection rules.

## 11\. Step 9 - Validate File Integrity Monitoring

**Where:** Windows endpoint  
**Run as:** PowerShell Administrator  
**Purpose:** Confirm that Wazuh detects monitored file changes.

Before running this test, ensure the selected directory is included in the Wazuh Agent File Integrity Monitoring configuration.

Create a dedicated test directory:

New-Item -Path "C:\\WazuhFIMTest" -ItemType Directory -Force

Create a test file:

New-Item -Path "C:\\WazuhFIMTest\\wazuh_test.txt" -ItemType File -Force

Modify the file:

Add-Content -Path "C:\\WazuhFIMTest\\wazuh_test.txt" -Value "Wazuh FIM validation test"

Delete the file:

Remove-Item -Path "C:\\WazuhFIMTest\\wazuh_test.txt" -Force

**What this does:**

- Creates, modifies, and deletes a test file.
- Validates whether Wazuh File Integrity Monitoring detects file activity in the configured directory.

**Expected result:** Wazuh generates file-added, file-modified, and file-deleted events.

Important: C:\\WazuhFIMTest must be added to the Wazuh Agent &lt;syscheck&gt; configuration before this test. Otherwise, Wazuh will not monitor the directory.

## 12\. Step 10 - Validate Registry Persistence Monitoring

**Where:** Windows endpoint  
**Run as:** Command Prompt Administrator  
**Purpose:** Simulate a common persistence technique by creating a Windows Run registry key.

Create a harmless Run key:

reg add "HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run" /v WazuhPOCTest /t REG_SZ /d "cmd.exe /c echo WazuhPOC" /f

Verify the key:

reg query "HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run" /v WazuhPOCTest

Remove the key after validation:

reg delete "HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run" /v WazuhPOCTest /f

**What this does:**

- Adds a program to the Windows startup Run key.
- Simulates a persistence method used by attackers.
- Validates Wazuh registry monitoring or event-based detection.

**Expected result:** Wazuh generates a registry modification or persistence-related alert if registry monitoring, Sysmon, or relevant custom rules are enabled.

## 13\. Step 11 - Monitor Alerts from the Wazuh Manager

**Where:** Wazuh Manager host  
**Purpose:** View alerts directly from the Wazuh Manager container while the Windows tests are running.

Run this command on the Wazuh Docker host:

docker exec -it single-node-wazuh.manager-1 sh -c '  
tail -F /var/ossec/logs/alerts/alerts.json | python3 -c "  
import sys  
import json  
<br/>for line in sys.stdin:  
try:  
event = json.loads(line)  
rule = event.get(\\"rule\\", {})  
agent = event.get(\\"agent\\", {})  
level = int(rule.get(\\"level\\", 0))  
<br/>if level >= 3:  
print(  
\\"Agent: {} | Rule: {} | Level: {} | Description: {}\\".format(  
agent.get(\\"name\\", \\"unknown\\"),  
rule.get(\\"id\\", \\"unknown\\"),  
level,  
rule.get(\\"description\\", \\"\\")  
)  
)  
except Exception:  
pass  
"  
'

**What this does:**

- Reads the live Wazuh alert stream.
- Displays the agent name, rule ID, severity level, and alert description.
- Confirms that the Wazuh Manager is receiving and processing endpoint events.

**Expected result:** Alerts appear in the terminal while tests are executed on the Windows endpoint.

## 14\. Step 12 - Validate Alerts in the Wazuh Dashboard

**Where:** Wazuh Dashboard  
**Purpose:** Confirm that alerts are indexed and visible to SOC analysts.

- Open the Wazuh Dashboard.
- Navigate to:

Sidebar - Explore -Discover

and also for clear logs that is being created

- Filter using the Windows agent name.
- Search for the following terms:

wazuh_test_user  
WazuhTestTask  
WazuhPOCTest  
4625  
4720  
4732  
4726  
4688

**Expected result:** The SOC analyst can view endpoint events, alert severity, affected agent, timestamp, rule description, and event details.

## 15\. Expected POC Outcome

The POC is successful when the following is confirmed:

| Validation Area           | Success Criteria                                      |
| ------------------------- | ----------------------------------------------------- |
| Agent onboarding          | Windows agent appears as Active in Wazuh Dashboard    |
| Log collection            | Windows Security events are received by Wazuh         |
| Authentication monitoring | Failed logon events are visible                       |
| Account monitoring        | User creation and deletion events are visible         |
| Privilege monitoring      | Administrator-group changes are visible               |
| Process monitoring        | PowerShell and scheduled-task activity is visible     |
| File monitoring           | FIM detects file creation, modification, and deletion |
| Registry monitoring       | Run-key modification is visible when configured       |
| SOC visibility            | Alerts are searchable in Wazuh Dashboard              |

## 16\. POC Conclusion

This POC demonstrates that the Windows endpoint is successfully integrated into the Wazuh SIEM platform.

It confirms that:

- The Wazuh Agent collects endpoint security telemetry.
- The Wazuh Manager analyzes that telemetry using detection rules.
- Security events are converted into alerts.
- Alerts are sent to the Wazuh Indexer.
- SOC analysts can investigate alerts through the Wazuh Dashboard.

The final operational flow is:

Windows Activity  
→ Wazuh Agent Collection  
→ Wazuh Manager Analysis  
→ Wazuh Alert Generation  
→ Indexer Storage  
→ Dashboard Visibility  
→ SOC Investigation