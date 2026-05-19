# ⚡ ServiceNow Flow Designer Patterns

> A library of reusable Flow Designer flows, subflow patterns, approval workflows, and automation blueprints — built from enterprise Now Platform implementations across ITSM, HRSD, GRC, and ITAM.

<p>
  <img src="https://img.shields.io/badge/ServiceNow-Flow%20Designer-purple?style=for-the-badge&logo=servicenow&logoColor=white"/>
  <img src="https://img.shields.io/badge/Automation-Enterprise%20Grade-1c2b3a?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Author-Swarup%20Kumar%20Namana-b8922a?style=for-the-badge"/>
</p>

---

## 📌 Overview

This repository documents production-grade Flow Designer patterns for automating business processes across ServiceNow modules. Each pattern includes a flow diagram, trigger configuration, action sequence, subflow design, error handling approach, and reusability notes.

**Impact delivered with these patterns:**
- ⏱️ ~30% reduction in ticket resolution time
- 🤖 Elimination of manual multi-step approval processes
- 📋 Standardized process execution across departments
- 🔔 Consistent notification and escalation handling

---

## 📂 Repository Structure

```
servicenow-flow-designer-patterns/
│
├── 01-itsm-flows/
│   ├── incident-auto-resolution.md       # Auto-resolve incidents via monitoring
│   ├── change-approval-workflow.md       # Multi-level change approval flow
│   ├── problem-investigation.md          # Problem → RCA → Known Error flow
│   └── service-catalog-fulfillment.md    # Catalog item fulfillment automation
│
├── 02-hrsd-flows/
│   ├── employee-onboarding.md            # New hire onboarding end-to-end
│   ├── employee-offboarding.md           # Offboarding & access revocation
│   ├── life-events.md                    # Name change, address update flows
│   └── hr-case-routing.md               # Auto-route HR cases by topic
│
├── 03-grc-flows/
│   ├── control-test-automation.md        # Scheduled control test execution
│   ├── risk-escalation.md               # High-risk auto-escalation flow
│   └── policy-review-cycle.md           # Annual policy review notification
│
├── 04-itam-flows/
│   ├── asset-lifecycle.md               # Asset request → deploy → retire
│   ├── software-reclamation.md          # Unused license reclamation flow
│   └── contract-expiry-alert.md         # Contract expiry notification flow
│
├── 05-subflows/
│   ├── approval-subflow.md              # Reusable multi-level approval
│   ├── notification-subflow.md          # Reusable notification dispatcher
│   ├── record-lookup-subflow.md         # Safe GlideRecord lookup pattern
│   └── rest-call-subflow.md             # Reusable REST API caller
│
└── 06-best-practices/
    ├── flow-naming-conventions.md        # Naming & organization standards
    ├── error-handling-patterns.md        # Try/catch & error flow patterns
    ├── performance-optimization.md       # Avoid common performance pitfalls
    └── testing-strategy.md              # ATF + manual testing approach
```

---

## 🔄 Pattern 1: Employee Onboarding Flow

### Flow Overview
```
Trigger: HRSD Case created (topic = Onboarding)
         │
         ▼
Subflow: Validate Employee Record
         │
         ├── Missing data? → Notify HR → Wait for update
         │
         ▼
Parallel Branches:
  ├── IT Branch:    Create AD account → Assign laptop → Setup email
  ├── HR Branch:    Send welcome email → Schedule orientation
  ├── Facilities:  Assign desk → Order badge
  └── Finance:     Setup payroll → Cost center assignment
         │
         ▼
All branches complete?
  ├── No  → Wait (check every 24h) → Escalate after 3 days
  └── Yes → Mark onboarding case complete → Send completion email
         │
         ▼
Day 30 Check-in → Survey sent to new employee
```

### Key Flow Actions
```
[Trigger]
  Record Created → sn_hr_core_case (topic = Onboarding)

[Action 1] Get Employee Record
  Table Lookup → sys_user where sys_id = trigger.opened_for

[Action 2] Check Required Fields
  Condition: user.email AND user.department AND user.manager
  If false → Create task for HR to complete profile

[Action 3] Create IT Provisioning Task
  Create Record → sc_task
  Assignment Group: IT Provisioning
  Short Description: "Provision accounts for: " + user.name
  Due Date: +2 business days

[Action 4] Send Welcome Email
  Send Email → To: user.email
  Template: "HRSD_Welcome_Email"

[Action 5] Wait for IT Task Completion
  Wait for Condition → sc_task.state = Closed
  Timeout: 3 days → Escalate to IT Manager

[Action 6] Complete Onboarding Case
  Update Record → sn_hr_core_case.state = Closed Complete
```

---

## 🔄 Pattern 2: Multi-Level Change Approval

### Flow Diagram
```
Trigger: Change Request submitted (state = Assess)
         │
         ▼
Check Change Type:
  ├── Standard   → Auto-approve → Schedule
  ├── Normal     → CAB Review → Vote → Approve/Reject
  └── Emergency  → Emergency CAB → Expedited approval
         │
         ▼
[Normal Path]
CAB Meeting Scheduled?
  ├── No  → Schedule next CAB window
  └── Yes → Send agenda to CAB members
         │
         ▼
CAB Vote (Approval Rules)
  ├── All approve       → Move to Scheduled
  ├── Majority approve  → Flag for Change Manager review
  └── Any reject        → Return to requester with comments
         │
         ▼
Scheduled → Implementation window opens
         │
         ▼
Post-Implementation Review (PIR) task created
```

### Subflow: Reusable Approval Handler

```javascript
// Script Step inside Approval Subflow
// Determines approval outcome and routes accordingly

(function() {
    var approvalRecord = fd_data.approval_sys_id;
    var gr = new GlideRecord('sysapproval_approver');
    gr.addQuery('sysapproval', approvalRecord);
    gr.query();

    var approved = 0, rejected = 0, pending = 0;

    while (gr.next()) {
        if (gr.state == 'approved')  approved++;
        if (gr.state == 'rejected')  rejected++;
        if (gr.state == 'requested') pending++;
    }

    // Return outcome to parent flow
    fd_data.approval_outcome = rejected > 0 ? 'rejected' :
                               pending  > 0 ? 'pending'  : 'approved';
    fd_data.approved_count  = approved;
    fd_data.rejected_count  = rejected;
    fd_data.pending_count   = pending;
})();
```

---

## 🔄 Pattern 3: Software License Reclamation

### Flow Overview
```
Trigger: Scheduled (Monthly, 1st of month)
         │
         ▼
Query: Software assets unused > 90 days
         │
         ▼
For Each Unused License:
         │
         ├── Get assigned user
         │
         ├── Send "Are you still using this?" email
         │   (with Yes / No response links)
         │
         ├── Wait 7 days for response
         │
         ├── Response = Yes?  → Keep, log as confirmed active
         │   Response = No?   → Revoke license immediately
         │   No response?     → Auto-revoke + notify manager
         │
         ▼
Update SAM Pro allocation records
         │
         ▼
Generate reclamation report → Send to IT Asset Manager
```

### Script: Find Unused Licenses
```javascript
// Script Step: Query unused software allocations
var unused = [];
var gr = new GlideRecord('alm_license');
gr.addQuery('state', 'in_use');
gr.addQuery('u_last_used', '<', gs.daysAgoStart(90));
gr.query();

while (gr.next()) {
    unused.push({
        sys_id:   gr.sys_id,
        software: gr.getDisplayValue('product_model'),
        user:     gr.getDisplayValue('assigned_to'),
        user_id:  gr.assigned_to.toString(),
        email:    gr.assigned_to.email.toString(),
        days:     gr.u_days_unused.toString()
    });
}

fd_data.unused_licenses = JSON.stringify(unused);
fd_data.count = unused.length;
gs.info('License Reclamation: Found ' + unused.length + ' unused licenses');
```

---

## 🔄 Pattern 4: Service Catalog Fulfillment

### Generic Catalog Fulfillment Flow
```
Trigger: RITM created → Catalog Item category = "Hardware Request"
         │
         ▼
Extract Variables from RITM
  (item_type, quantity, justification, manager)
         │
         ▼
Manager Approval Required?
  ├── Cost > $500  → Yes → Request manager approval
  └── Cost ≤ $500  → No  → Auto-approve
         │
         ▼
Approved → Create Procurement Task
  Assign to: Procurement Team
  Due: +5 business days
         │
         ▼
Asset received → Assign to user
  Create Asset record in ITAM
  Send "Your equipment is ready" notification
         │
         ▼
Close RITM → Survey sent
```

---

## ⚙️ Reusable Subflow: Notification Dispatcher

### Purpose
A single reusable subflow for sending notifications consistently across all flows — avoids duplicating email logic in every flow.

### Inputs
| Input | Type | Description |
|-------|------|-------------|
| recipient_id | String | sys_user sys_id |
| template_name | String | Email notification name |
| record_sys_id | String | The triggering record |
| additional_data | Object | Extra key-value pairs for template |

### Subflow Steps
```
[Step 1] Get User Email
  Table Lookup → sys_user → email, name, time_zone

[Step 2] Load Notification Template
  Script → GlideRecord lookup on sys_notification

[Step 3] Build Dynamic Content
  Script → Replace template variables with actual values

[Step 4] Send Email
  ServiceNow Action → Send Email
  Handle: Delivery failure → Log + Create follow-up task

[Step 5] Log Notification
  Create Record → u_notification_log
  Fields: recipient, template, sent_time, status
```

---

## 🏆 Best Practices

### Naming Conventions
```
Flows:     [Module] - [Process] - [Action]
           Example: ITSM - Incident - Auto Escalation

Subflows:  SUB - [Function Name]
           Example: SUB - Multi Level Approval

Actions:   ACT - [What It Does]
           Example: ACT - Create Asset Record
```

### Error Handling Pattern
```
Every Flow should have:

[Try Path]
  Normal business logic steps
        │
        ▼
[Error Handler - Always runs on exception]
  ├── Log error to Application Log
  ├── Create incident for integration team (if integration)
  ├── Send alert email to system admin group
  └── Update source record with error state + message

[Finally]
  Update audit trail record regardless of outcome
```

### Performance Rules
| Rule | Why |
|------|-----|
| Never query inside a loop | Use GlideRecord once, iterate results |
| Use subflows for repeated logic | Single update propagates everywhere |
| Set explicit timeouts on waits | Flows don't hang indefinitely |
| Use async flows for non-urgent tasks | Don't block user transactions |
| Limit flow triggers to necessary conditions | Reduce unnecessary flow executions |

---

## ✅ ATF Testing Strategy

```javascript
// ATF Test Step: Verify onboarding flow creates IT task
var atfHelper = new sn_atf.UserStory();

// Step 1: Create HRSD onboarding case
var hrCase = atfHelper.createRecord('sn_hr_core_case', {
    topic:       'onboarding',
    opened_for:  gs.getUserID(),
    state:       'ready'
});

// Step 2: Wait for flow to execute
atfHelper.waitForCondition(function() {
    var task = new GlideRecord('sc_task');
    task.addQuery('request_item.opened_for', hrCase.opened_for);
    task.addQuery('short_description', 'CONTAINS', 'Provision accounts');
    task.query();
    return task.next();
}, 10000); // 10 second timeout

// Step 3: Assert task was created correctly
atfHelper.assert(task.state == '1', 'IT provisioning task should be in Open state');
atfHelper.assert(task.assignment_group != '', 'Task should have an assignment group');
```

---

## 👤 Author

**Swarup Kumar Namana**
Senior ServiceNow Developer & Platform Architect
Columbus, Ohio, USA

[![Portfolio](https://img.shields.io/badge/Portfolio-swarup--namana.netlify.app-b8922a?style=flat-square)](https://swarup-namana.netlify.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-swarupnamana-0077B5?style=flat-square&logo=linkedin)](https://linkedin.com/in/swarupnamana)
[![Email](https://img.shields.io/badge/Email-swarupnamana03%40gmail.com-D14836?style=flat-square&logo=gmail)](mailto:swarupnamana03@gmail.com)

---

*Patterns abstracted from real enterprise implementations. No proprietary or client-specific logic included.*
