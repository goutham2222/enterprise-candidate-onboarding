# Enterprise Candidate Onboarding & Lifecycle Engine

[![Power Apps](https://img.shields.io/badge/Power%20Apps-Canvas-742774?logo=powerapps&logoColor=white)](https://powerapps.microsoft.com/)
[![Power Automate](https://img.shields.io/badge/Power%20Automate-Cloud%20Flows-0066FF?logo=powerautomate&logoColor=white)](https://flow.microsoft.com/)
[![SharePoint](https://img.shields.io/badge/SharePoint-List%20Storage-0078D4?logo=microsoftsharepoint&logoColor=white)](https://www.microsoft.com/microsoft-365/sharepoint)
[![Architecture](https://img.shields.io/badge/Pattern-Dual--Channel%20Approval-teal)](#system-architecture)

An end-to-end, multi-stage enterprise candidate intake, approval orchestration, and automated provisioning platform. Built using **Microsoft Power Apps**, **Power Automate**, and **SharePoint Online**, this platform solves parallel webhook race conditions, enforces strict SLA escalations, and incorporates a structured Try-Catch fault tolerance pattern.

---

## Architecture Overview

The system addresses state synchronization across asynchronous human tasks by coordinating email actionable messaging alongside real-time Power Apps dashboard approvals.

                       +-------------------------------------+
                       | 1. Candidate Intake (Power Apps)     |
                       | scr_Intake / SharePoint Create Event |
                       +------------------+------------------+
                                          |
                                          v
                  +-----------------------------------------------+
                  | 2. Orchestrator Engine (Manager Escalator)     |
                  | Scope_Try (Parallel Non-Blocking Architecture) |
                  +-----------------------+-----------------------+
                                          |
                 +------------------------+------------------------+
                 |                                                 |
                 v                                                 v
+------------------------------------+             +------------------------------------+
| Path A: Real-Time State Watcher    |             | Path B: Actionable Webhook Chain   |
| - Do-Until SharePoint Poller       |             | - Manager Approval (48h SLA)       |
| - Polls status every 60 seconds    |             | - Reminder Escalation (48h SLA)    |
| - Terminate(Succeeded) on decision |             | - HR Escalation Fallback (48h SLA) |
+------------------+-----------------+             +------------------+-----------------+
|                                                  |
+------------------------+-------------------------+
|
v
+-------------------------------+
| Status Patched to "Approved"  |
+---------------+---------------+
|
v
+-----------------------------------------------+
| 3. Onboarding Fulfillment Engine (Flow 3)     |
| - Trigger: Split On with @equals('Approved')  |
| - Dynamic HTML Document Compilation           |
| - OneDrive Serverless PDF Conversion          |
| - Candidate Dispatch & File Lifecycle Cleanup |
+-----------------------------------------------+


---

## Core System Modules

### 1. Multi-Stage Approval Orchestrator (`flows/02-manager-escalator-definition.json`)
* **Dual-Channel Polling & Termination:** Resolves native Power Automate parallel execution limits by pairing a `Do-Until` loop with an early-exit `Terminate(Succeeded)` action, immediately killing lingering email webhooks upon dashboard decision.
* **SLA Escalation Sequence:** Enforces a 96-hour escalation hierarchy across Manager Initial Review (48h), Manager Reminder (48h), and HR Takeover (48h).
* **Enterprise Try-Catch Fault Handling:** Encapsulates core logic in `Scope_Try`. If any connector fails, execution routes to `Scope_Catch`, patches SharePoint to `Error / Manual Review Required`, and issues an IT incident alert.

### 2. Candidate Intake & Operations Console (`power-apps/src/`)
* **`scr_Intake`**: Self-service onboarding portal featuring client-side input validation, dynamic hardware selection, and direct SharePoint schema patching.
* **`scr_ManagerQueue`**: Filtered operational queue displaying records matching `ManagerEmail.Email = User().Email` with one-click decision controls.
* **`scr_HRDashboard`**: Executive roster and oversight view providing SLA metrics (`Pending`, `Approved`, `Attention Required`), department sorting, and administrative override actions.

### 3. Serverless Document Engine (`flows/03-onboarding-fulfillment-definition.json`)
* **Trigger Isolation:** Filtered via `@equals(triggerBody()?['OnboardingStatus']?['Value'], 'Approved')` to execute strictly on terminal authorization.
* **Transient PDF Generation:** Compiles personalized corporate welcome packets in HTML, converts to PDF via OneDrive API services, dispatches via Outlook, and deletes the transient source file within the same pipeline.

---

## SharePoint Schema Definition

| Column Internal Name | Column Display Name | Type | Notes |
| :--- | :--- | :--- | :--- |
| `Title` | Candidate Name | Single line of text | Required system field |
| `PersonalEmail` | Personal Email | Single line of text | Target recipient address |
| `Department` | Department | Choice | Engineering, HR, Finance, IT, Ops |
| `StartDate` | Target Start Date | Date and Time | Date only |
| `EquipmentPreference` | Hardware Choice | Choice | Mac workstation, ThinkPad, Linux |
| `ManagerEmail` | Reporting Manager | Person or Group | Dynamic manager reference |
| `OnboardingStatus` | Onboarding Status | Choice | `Pending Approval`, `Approved`, `Rejected`, `Escalated to HR`, `Error / Manual Review Required` |

---

## Deployment & Configuration

1. **SharePoint Setup:**
   * Create the target SharePoint list with the columns defined above.
   * Note the List GUID via `List Settings > Web Address (URL)`.

2. **Power Automate Import:**
   * Import the JSON definition files in `flows/`.
   * Rebind the connection references to your tenant's `Office 365 Outlook`, `SharePoint Online`, and `OneDrive for Business` accounts.
   * In `02-manager-escalator-definition.json`, configure the run-after properties:
     * Set `Scope_Catch` to run after `Scope_Try` has failed, timed out, or been skipped.
     * Ensure the terminating condition verifies `OnboardingStatus/Value` equals `Approved`.

3. **Power Apps Deployment:**
   * Pack the source directory using the Power Platform CLI:
     ```bash
     pac canvas pack --sources "./power-apps/src" --msapp "EnterpriseOnboarding.msapp"
     ```
   * Import the `.msapp` package into your target Power Apps environment and connect the data source to your SharePoint list.
