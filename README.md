# Zero-Touch Onboarding Automation on Microsoft 365

An end-to-end identity provisioning system that turns a single Microsoft Form submission into a fully governed new-hire account — created in Microsoft Entra ID, auto-licensed via group inheritance, MFA-enforced via Conditional Access, device-policy-enrolled via Intune, and notified via a templated welcome email — with **zero manual administrator clicks**.

Built on a Microsoft 365 E5 tenant using Microsoft Forms, Power Automate, Microsoft Entra ID, Intune, and SharePoint.

---

# The Problem

In a typical enterprise, onboarding a single new hire takes ~45 minutes of coordinated work across four teams:

- HR enters the new hire into a system of record
- IT creates the user account in Entra ID
- IT manually assigns the appropriate license SKU
- Security adds the user to access groups and applies policies
- IT/Helpdesk dispatches a welcome email with credentials

The process is error-prone (typos in usernames, forgotten license assignments, inconsistent group membership), expensive in admin time, and creates a poor first-day experience for the new hire who waits hours or days for access. It also doesn't scale — onboarding 50 hires in a quarter compounds the same errors 50 times.

# The Solution

A single Microsoft Form acts as the system of record. Submitting it triggers a Power Automate cloud flow that creates the user with the correct attributes set. Everything downstream — license assignment, group membership, security policy enforcement, device baseline — reacts automatically through attribute-based rules already configured in Entra and Intune.

The architectural insight: **the automation is small because the infrastructure is intelligent**. The flow only needs to set the right attributes on the user object; Entra's dynamic groups + group-based licensing + Conditional Access handle the rest as side effects of group membership.

---

# Architecture

```
┌─────────────────┐
│ Microsoft Form  │  "New Hire Request"
│ (HR / Manager)  │
└────────┬────────┘
         │ submission triggers
         ▼
┌─────────────────────────────────────────────────────┐
│ Power Automate Cloud Flow                           │
│                                                     │
│ 1. Get response details                             │
│ 2. Generate UPN: firstname.lastname@tenant          │
│ 3. Generate temp password: Welcome@<8-char-guid>!   │
│ 4. Create user in Entra ID                          │
│    └─ sets: Department, Job Title, Usage Location   │
│ 5. Fetch "Getting Started.pdf" from SharePoint      │
│ 6. Send welcome email with credentials + PDF        │
└────────┬────────────────────────────────────────────┘
         │ user attributes set
         ▼
┌─────────────────────────────────────────────────────┐
│ Microsoft Entra ID — automatic reactions            │
│                                                     │
│ Dynamic Groups (rule-based membership):             │
│   • grp-AllNewHires    ← any active Member user     │
│   • grp-dept-Sales     ← user.department = Sales    │
│   • grp-dept-Engineering, HR, Finance, Marketing    │
│                                                     │
│ Group-Based Licensing:                              │
│   • grp-AllNewHires has Microsoft 365 E5 attached   │
│   • New member → inherits E5 within minutes         │
│                                                     │
│ Conditional Access (CA001):                         │
│   • grp-AllNewHires → Require MFA on all cloud apps │
│   • Break Glass account explicitly excluded         │
│                                                     │
│ Intune Configuration Profile:                       │
│   • Targets grp-AllNewHires                         │
│   • Desktop wallpaper / device baseline pushed      │
│     to any Windows device enrolled by the user      │
└─────────────────────────────────────────────────────┘
```

# Tech Stack

| Layer | Technology |
|---|---|
| Trigger | Microsoft Forms |
| Orchestration | Power Automate (cloud flow) |
| Identity | Microsoft Entra ID (P2) |
| Licensing | Microsoft 365 E5 (group-based assignment) |
| Access Control | Entra Conditional Access |
| Device Management | Microsoft Intune |
| Content Storage | SharePoint Online |
| Notification | Office 365 Outlook |

---

# Logic

# Form schema

The "New Hire Request" form captures seven fields:

| Field | Type | Purpose |
|---|---|---|
| First Name | Text | Used in UPN, display name, mail nickname, email greeting |
| Last Name | Text | Same as above |
| Job Title | Text | Stored on user object for HR/reporting |
| Department | Choice (5 options) | **Drives dynamic group membership** |
| Manager Email | Text (email-validated) | Stored for org hierarchy |
| Personal Email | Text (email-validated) | Destination for welcome email |
| Start Date | Date | Communicated in welcome email |

The Department field is the architectural keystone. Its values (`Sales`, `Engineering`, `HR`, `Finance`, `Marketing`) are constrained Choice options that match — character for character — the dynamic group rules in Entra. This makes typos impossible.

# Power Automate flow

The flow runs as 7 actions:

1. **Trigger** — `When a new response is submitted` from the New Hire Request form
2. **Get response details** — fetches the actual form data (the trigger only provides a response ID)
3. **Compose - UPN** — generates `firstname.lastname@<tenant>.onmicrosoft.com`
4. **Compose - TempPassword** — generates `Welcome@<8-char-guid-fragment>!`, meeting Entra password complexity
5. **Create user (V3)** — creates the user in Entra with attributes set:
   - Account Enabled: true
   - Display Name, Given Name, Surname, Job Title, Department: from form
   - Mail Nickname: `firstname.lastname` (no `@` — critical, Entra rejects `@` in this field)
   - User Principal Name + Password: from the two Compose actions
   - Usage Location: `IN` (required for license assignment to succeed)
   - Force Change Password Next Sign In: true (forces password rotation on first login)
6. **Get Getting Started PDF** — retrieves the onboarding PDF from a SharePoint document library
7. **Send email (V2)** — composes and sends the welcome email to the personal address from the form, with the PDF attached

Once Action 5 completes, Entra's dynamic membership engine evaluates the user's `department` attribute against each dynamic group's rule within ~1-5 minutes. Group membership cascades into license assignment (from `grp-AllNewHires`) and CA/Intune policy targeting.

# Dynamic membership rules

Each department group uses a single-expression rule:

```
(user.department -eq "Sales")
```

The catch-all `grp-AllNewHires` uses:

```
(user.userType -eq "Member") and (user.accountEnabled -eq true)
```

This catch-all is what carries the M365 E5 license assignment, ensuring every active member user is licensed regardless of department.

---

# Security Posture

The project demonstrates four layered enterprise security practices:

# 1. Mandatory MFA via Conditional Access

Policy **CA001 — Require MFA for All New Hires** targets `grp-AllNewHires` and grants access to any cloud app only after multifactor authentication succeeds. Every user created by the flow lands in this group within minutes and inherits the policy. New hires cannot sign in without setting up MFA on first login.

# 2. Break-Glass Account Exclusion

A dedicated `breakglass` account holds Global Administrator privileges and is **explicitly excluded** from CA001. Its credentials are stored physically (paper, offline) and the account is never used for routine administration. This is the recovery path if the regular admin's MFA device is lost — a non-negotiable safeguard in any production tenant. Without it, a single failed authenticator app can permanently lock an organization out of its own tenant.

# 3. Forced Password Rotation

Auto-generated passwords are flagged `forceChangePasswordNextSignIn`. The user cannot complete their first sign-in without immediately rotating the credential. This means no human, including the IT admin who provisions the account, knows the long-term password.

# 4. Device Baseline via Intune

A configuration profile targets `grp-AllNewHires` and pushes a baseline policy (in this implementation: forced desktop wallpaper; in production: BitLocker disk encryption, screen lock timing, USB restrictions, etc.) to any Windows device the user enrolls. Device compliance is tied to identity, not provisioned out-of-band.

---

# Efficiency Gains

| Metric | Manual Process | Automated Process |
|---|---|---|
| Wall-clock time | ~45 minutes | ~90 seconds |
| Human admin time | ~45 minutes across 4 teams | ~10 seconds (HR submits form) |
| Risk of typo | Moderate (free-text entry) | Eliminated (Choice fields + computed values) |
| Risk of missed license assignment | Common (per-user assignment forgotten) | Eliminated (group-based inheritance) |
| Risk of inconsistent group membership | Common | Eliminated (attribute-based rules) |
| Audit trail | Email chains, ticket threads | Native Entra audit log + Power Automate run history |

# Cost-management note

License assignment is performed at the group level, not the user level. This means:

- A user who is removed from `grp-AllNewHires` (e.g., off-boarded) automatically loses the E5 license. The license seat is reclaimed for re-use without admin action.
- Scaling from 25 hires to 2,500 requires no changes to the licensing logic — the group's license assignment is one configuration line.
- License waste from forgotten direct assignments is structurally impossible.

This is the production pattern used in large enterprises for managing thousands of identities. It scales linearly with no per-user admin overhead.

---

# Screenshots

> All screenshots taken from the live M365 E5 tenant during end-to-end testing.

# Infrastructure (Day 1)

| # | Screenshot | What it shows |
|---|---|---|
| 1 | `01-dynamic-groups.png` | Six dynamic groups (5 departmental + 1 catch-all), all Cloud-sourced, Security-typed |
| 2 | `02-group-based-licensing.png` | M365 E5 license assigned to `grp-AllNewHires` via group-based licensing |
| 3 | `03-conditional-access-policy.png` | CA001 policy: scoped to `grp-AllNewHires`, breakglass excluded, MFA required |
| 4 | `04-intune-config-profile.png` | Intune configuration profile assigned to `grp-AllNewHires` |

# Automation (Day 2)

| # | Screenshot | What it shows |
|---|---|---|
| 5 | `05-microsoft-form.png` | New Hire Request form with 7 fields, including constrained Department choices |
| 6 | `11-flow-all-actions.png` | The complete Power Automate cloud flow, 7 actions |
| 7 | `12-flow-run-all-green.png` | A successful end-to-end run with every action green-checked |

# Reactions (downstream of user creation)

| # | Screenshot | What it shows |
|---|---|---|
| 8 | `13-welcome-email-received.png` | The welcome email as received, with attached Getting Started PDF (password redacted) |
| 9 | `14-user-in-entra.png` | The newly created user in Entra → Users with department/title populated |
| 10 | `15-dynamic-group-member.png` | The user listed as a member of `grp-dept-<Department>` (dynamic membership fired) |
| 11 | `16-license-inherited.png` | The user's Licenses tab showing E5 **inherited** from `grp-AllNewHires` |
| 12 | `18-audit-log-user-creation.png` | Entra audit log entry for the automated user creation event |

---

# Production Considerations

A few honest notes on what would change if this were deployed in a real organization rather than a lab tenant:

- **Custom verified domain** — The lab uses the default `<tenant>.onmicrosoft.com`. A production deployment uses a verified custom domain (`@company.com`) with SPF, DKIM, and DMARC DNS records configured. Without sender reputation, welcome emails sent from a fresh `.onmicrosoft.com` tenant are aggressively spam-filtered by external providers like Gmail (this was observed during testing).
- **Service account for the flow's connection** — The flow currently runs under the admin's user context. In production, a dedicated service principal with narrowly-scoped permissions (User.ReadWrite.All, Group.Read.All, Mail.Send) would own the connection. This decouples the automation from any individual admin's account lifecycle.
- **Form access control** — The form is currently open to anyone in the tenant. Production would restrict it to HR/managers via a group, with approval gating before user creation if regulatory requirements dictate.
- **Off-boarding parallel** — A symmetrical "Departing Hire" flow would disable the user, remove from groups (which auto-revokes license + access), and trigger Microsoft Purview retention on the user's mailbox/OneDrive.
- **Logging and alerting** — Power Automate run failures would route to a Teams channel or PagerDuty so failed onboarding doesn't sit unnoticed.

---

# Lessons Learned

A few non-obvious gotchas encountered during the build, documented here for anyone reproducing it:

- **Usage Location is mandatory for license assignment.** Without it, the user is created successfully but Entra silently fails to attach the license, with a non-obvious "usage location not specified" warning surfacing minutes later. The flow sets `Usage Location: IN` on every user.
- **Mail Nickname cannot contain `@`.** It's the local-part of the email address, not the full address. Using the UPN as the mail nickname fails with a confusing "invalid value" error.
- **Microsoft 365 Developer Program no longer offers free sandbox subscriptions** to most applicants since early 2024. This project was built on a 30-day Microsoft 365 E5 trial tenant instead, which provides functionally equivalent capabilities (Entra ID P2, Intune, Conditional Access) within the trial window.
- **Email deliverability to Gmail from new `.onmicrosoft.com` tenants is unreliable.** The fix in production is a verified custom domain with proper SPF/DKIM/DMARC; in a lab, testing against non-Gmail addresses (Outlook, Yahoo, institutional email) sidesteps the spam-filter issue.
- **Dynamic group membership is eventually consistent.** Plan for 1-5 minutes between user attribute changes and group membership reflecting downstream. This is fine for onboarding (where the user is being set up over many minutes anyway) but matters in scenarios with tight time constraints.

---

# Tenant Cleanup

This project was built on a 30-day trial. To prevent accidental conversion to a paid subscription, the trial was cancelled before its expiration date.

---

# Repository Contents

```
.
├── README.md                          (this file)
├── /screenshots
│   ├── 01-dynamic-groups.png
│   ├── 02-group-based-licensing.png
│   ├── 03-conditional-access-policy.png
│   ├── 04-intune-config-profile.png
│   ├── 05-microsoft-form.png
│   ├── 11-flow-all-actions.png
│   ├── 12-flow-run-all-green.png
│   ├── 13-welcome-email-received.png
│   ├── 14-user-in-entra.png
│   ├── 15-dynamic-group-member.png
│   ├── 16-license-inherited.png
│   ├── 17-ca-policy-enforced.png
│   └── 18-audit-log-user-creation.png
└── /flow-export
    └── ZeroTouchOnboarding.json       (optional: Power Automate flow definition export)
```

---

# Author

Hinesh Bandlamudi 
Built: May 2026  
Connect: https://www.linkedin.com/in/hinesh-b-657624125/ | hinesh@outlook.in
