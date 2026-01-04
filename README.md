# Cloud Identity Architecture: Zero Trust Resiliency & Automated Governance

## 📋 Executive Summary
This project documents the architecture of a production-grade **Hybrid Identity & Governance** environment. The objective was to bridge legacy on-premise Active Directory (ADDS) with Microsoft Entra ID while implementing "Tier-0" security controls (Break Glass, Kill Switches) and automated Lifecycle Management (IGA).

### 🎯 Engineering Highlights
* **Resiliency:** Architected a "Break Glass" emergency protocol that survives total federation failure (AD Connect outage) and MFA service downtime.
* **Hybrid Remediation:** Diagnosed and fixed **Password Writeback** failures by modifying Access Control Lists (ACLs) on the on-premise Domain Controller.
* **Compliance Automation:** Replaced manual user access reviews with **Entra ID Governance**, enforcing weekly recertification cycles for high-risk financial roles.

---

## 🛠️ Phase 1: Identity Resiliency ("Break Glass" Architecture)
*Objective: Create a fail-safe entry point into the tenant during a "Lockout Doom Loop."*

### 1.1 The Cloud-Only Anchor
A dedicated service account was created directly in the cloud (`.onmicrosoft.com` domain). This ensures that even if the on-premise Directory Sync fails, this account remains accessible.

**Cloud Only Creation** <img width="553" height="585" alt="Create-breakglass-user-1" src="https://github.com/user-attachments/assets/0156ab52-32d7-4b0b-8d90-967594029d66" />

*Configuration: Creating the `svc-emergency01` account with a permanent, non-expiring password hash isolated from on-prem policies.*

**Cloud Anchor Verification** <img width="852" height="444" alt="breakglass-user-config-off-prem-2" src="https://github.com/user-attachments/assets/b920dee6-fd56-419d-b4b8-1b00fffb49c9" />

*Verification: Confirmed "On-premises sync enabled: No". This isolation is critical for Business Continuity Planning (BCP).*

### 1.2 "Negative Testing" the MFA Policy
A Conditional Access policy was deployed to enforce MFA for all Global Admins, with a specific **Exclusion** for the emergency account.

**Policy Exclusion** <img width="630" height="607" alt="breakglass-policy-3" src="https://github.com/user-attachments/assets/e0bb7934-fa3d-4819-a311-19714e3ffc27" />

*Architecture: Explicitly excluding the Break Glass account from the standard MFA enforcement policy.*

To validate the safety net without risking a lockout, I utilized **Report-Only Mode** to simulate sign-ins.

**Log Validation** <img width="701" height="472" alt="testing-breakglass-policy-4" src="https://github.com/user-attachments/assets/c0c3e1f9-1173-4143-87f7-806d89094d47" />

*Validation: Sign-in logs confirm the policy result is "Not Matched," proving the exclusion logic functions correctly in a live production scenario.*

---

## 🔧 Phase 2: Hybrid Identity Bridge (Deep Dive)
*Objective: Enable Self-Service Password Reset (SSPR) and Account Unlock for remote users.*

### 2.1 The Engineering Challenge: Writeback Failures
While Entra Connect was syncing users correctly, "Password Writeback" was failing for locked-out users. I diagnosed this as a missing permission on the Service Account (`MSOL_...`) used by the connector.

### 2.2 The Fix: Delegate Control
I modified the AD DS permissions to explicitly grant the connector account authority over user object attributes.

**Permission Selection** <img width="502" height="389" alt="Screenshot 2026-01-03 164304" src="https://github.com/user-attachments/assets/283508fd-8510-42b2-b07c-93cd42f81b65" />

*Configuration: Granting `Read lockoutTime` and `Write lockoutTime` permissions. Standard setups often miss this, preventing the cloud from unlocking on-prem accounts.*

**Delegation Complete** <img width="493" height="390" alt="Screenshot 2026-01-03 164635" src="https://github.com/user-attachments/assets/cccf7ef4-25e2-4d06-af0c-b353eda90a91" />

*Resolution: Successfully delegated control to the `MSOL_` service account, restoring full bidirectional sync capability.*

---

## 🛡️ Phase 3: The "Kill Switch" (Defensive Architecture)
*Objective: Instant containment of identity breaches.*

Designed a "Panic Button" policy that can be toggled to **Block All Access** to the tenant for every user *except* the emergency administrators.

**Kill Switch Staged** <img width="529" height="693" alt="Creating-killswitch-5" src="https://github.com/user-attachments/assets/c70d387f-41cd-4c1d-bbfe-170d930d2d8b" />

*Architecture: The "BLOCK ALL ACCESS" policy is staged in "OFF" state. In an Active Breach scenario, enabling this single policy freezes the attack surface instantly.*

---

## ⚖️ Phase 4: Modern Identity Governance (IGA)
*Objective: Migrate from legacy "Request Access" tickets to Automated Entitlement Management.*

### 4.1 JIT (Just-in-Time) Access Packages
Instead of permanent standing privileges, I architected **Access Packages** that grant permissions for a limited time window.

**Access Package Catalog** <img width="958" height="405" alt="Screenshot 2026-01-03 232130" src="https://github.com/user-attachments/assets/7676581e-2c75-4d79-905f-16c18adce926" />

*Implementation: "Finance Audit Access" configured as a JIT package, reducing the attack surface of privileged accounts.*

### 4.2 Automated Compliance (Weekly Recertification)
To satisfy regulatory requirements (e.g., SOX/NYS DFS), I configured automated **Access Reviews**. If a manager does not re-validate the user's need for access, it is automatically revoked.

**Weekly Review Cycle** <img width="916" height="639" alt="Screenshot 2026-01-03 232515" src="https://github.com/user-attachments/assets/0e164867-0ee4-4bec-9619-1efac4420253" />

*Compliance: High-risk roles are set to a **Weekly** review cycle, ensuring strict adherence to "Least Privilege" principles.*
