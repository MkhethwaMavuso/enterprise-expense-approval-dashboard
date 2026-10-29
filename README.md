# enterprise-expense-approval-dashboard
A multi-tier procurement and expense approval workflow platform built on Retool and PostgreSQL featuring threshold routing, automated approval escalation, and immutable audit trails.

📋 Table of Contents📌 1. Product Requirements Document (PRD)🗄️ 2. Database Schema & Entity Architecture🏗️ 3. System Architecture & Retool Data Flow🛡️ 4. Security & Role-Based Access Control (RBAC)🔌 5. API Contract Specifications🚀 Setup & Installation📌 1. Product Requirements Document (PRD)❓ The ProblemUnstructured Approval Pathways: Expense requests lack defined threshold routing, causing low-value items to clog executive queues while high-value expenses bypass proper authorization.Absence of Audit Trails: Approvals handled across emails and messaging apps leave no verifiable, immutable records for accounting and compliance audits.Inconsistent Status Tracking: Requesters lack visibility into approval states, causing duplicate submissions and administrative delays.💡 The SolutionA centralized Enterprise Expense Approval Dashboard that automates threshold-based routing (auto-routing requests under $5k to Line Managers and $5k+ to VPs), enforces atomic transaction processing, and maintains immutable audit logs for every state transition.🎯 Key Functional Requirements🔐 FR-1 [Auth & Role Scoping]: System supports role-based user management across Requesters, Line Managers, VP Approvers, and Finance Admins.⚡ FR-2 [Automated Multi-Tier Routing]: Requests under $5,000 auto-route to Line Managers; requests $5,000 and above automatically escalate to VP Approvers.📝 FR-3 [Expense Request Submission]: Employees submit expense details including title, category, amount, vendor, description, and supporting receipt document links.📊 FR-4 [Immutable Audit Logging & Dashboards]: Every state change is immutably logged with timestamp and actor ID, accompanied by executive metrics on spending and approval bottlenecks.🗄️ 2. Database Schema & Entity ArchitectureThe backend database runs on Retool Database (PostgreSQL), adhering to strict relational constraints and PL/pgSQL sequences for automated identifier generation.📊 Entity-Relationship OverviewTable NamePrimary KeyForeign KeysKey Attributes & Description👤 usersid (SERIAL)Nonefull_name, email (UNIQUE), password_hash, role, department💰 expense_requestsid (SERIAL)requester_id -> users(id)request_code (UNIQUE), title, amount, vendor, status, current_approver_role📑 audit_logsid (SERIAL)expense_id -> expense_requests(id), actor_id -> users(id)previous_status, new_status, action_comments, timestamp⚡ Automated Key Generation (PostgreSQL)SQLCREATE SEQUENCE IF NOT EXISTS expense_code_seq START WITH 1001 INCREMENT BY 1;

CREATE OR REPLACE FUNCTION generate_expense_code()
RETURNS TEXT AS $$
BEGIN
    RETURN 'EXP-' || LPAD(nextval('expense_code_seq')::TEXT, 6, '0');
END;
$$ LANGUAGE plpgsql;
🏗️ 3. System Architecture & Retool Data FlowThe architecture follows a multi-tier structure separating client presentation, logic transformation, and database persistence.┌──────────────────────────────────────────────────────────────────┐
│                   CLIENT LAYER (Retool Canvas UI)                │
│                                                                  │
│  ┌───────────────────────┐             ┌───────────────────────┐ │
│  │ Navigation Header     │             │ Dynamic Data Grids    │ │
│  │ (Tabbed Layout)       │             │ (Tables & Form Views) │ │
│  └───────────────────────┘             └───────────────────────┘ │
│                                                                  │
│  • Session State Manager: `app_user`                             │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 │ Parametrized Data Binding
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│              APPLICATION LOGIC & TRANSFORMATION TIER            │
│                                                                  │
│  • Client JS Transformers: Threshold Evaluation & RBAC Guards    │
│  • Retool Event Pipelines: Trigger Cascades & Table Refreshes    │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 │ Parameterized SQL Transactions
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                PERSISTENCE TIER (PostgreSQL Database)            │
│                                                                  │
│  • Security Engine: `pgcrypto` (Bcrypt Password Verification)    │
│  • Sequence Engine: PL/pgSQL Functions                           │
│  • Storage Engine: Relational Tables & Audit Constraints         │
└──────────────────────────────────────────────────────────────────┘
🔄 Data Execution Lifecycle🖱️ User Action: Employee submits expense form with title, amount, vendor, and receipt link.⚙️ Client Validation & Routing: JS transformer evaluates amount threshold (< $5k vs >= $5k) to assign initial approver role.🗄️ Query Execution: Retool dispatches a parameterized SQL mutation (submitExpense) to PostgreSQL.⚡ Database Atomicity: PostgreSQL generates the EXP-xxxxxx code, inserts the record, and initializes tracking.🔔 Event Cascade: On success, a toast notification displays and re-fetches the active queue tables.🛡️ 4. Security & Role-Based Access Control (RBAC)🔐 Password Hashing (pgcrypto)All passwords are encrypted server-side using PostgreSQL's pgcrypto module with standard bcrypt blowfish salt rounds.🎭 Access Control MatrixRoleSubmit ExpenseView Own ExpensesApprove / Reject (< $5k)Approve / Reject (>= $5k)Mark PaidAccess Audit Logs👤 Requester✅✅❌❌❌❌👔 Line Manager✅✅✅❌❌❌VP Approver✅✅Escalated✅❌❌💳 Finance Admin✅✅Full ViewFull View✅✅🛡️ Component Security GuardsRetool UI component properties evaluate the app_user session variable to restrict approval buttons and financial audit views dynamically.🔌 5. API Contract Specifications📥 5.1 Expense Submission API (submitExpense)Type: Transactional Mutation QueryAccess Level: Requesters & All EmployeesInput Parameters:JSON{
  "requester_id": "number",
  "title": "string",
  "category": "string",
  "amount": "number",
  "vendor": "string",
  "description": "string",
  "receipt_url": "string"
}
SQL Implementation:SQLINSERT INTO expense_requests (
  request_code, requester_id, title, category, amount, vendor, description, receipt_url, current_approver_role
)
VALUES (
  generate_expense_code(),
  {{ form.data.requesterIdInput }},
  {{ form.data.titleInput }},
  {{ form.data.categorySelect }},
  {{ form.data.amountInput }},
  {{ form.data.vendorInput }},
  {{ form.data.descriptionInput }},
  {{ form.data.receiptUrlInput }},
  CASE WHEN {{ form.data.amountInput }} < 5000 THEN 'Line Manager' ELSE 'VP Approver' END
)
RETURNING id, request_code, current_approver_role;
📤 5.2 Approval Queue Query (getApprovalQueue)Type: Parametrized SQL Read QueryAccess Level: Line Managers & VP ApproversSQL Implementation:SQLSELECT * FROM expense_requests
WHERE current_approver_role = {{ app_user.value.role }}
  AND status LIKE 'Pending%'
ORDER BY created_at ASC;
🚀 Setup & InstallationDatabase Provisioning:Open your Retool Database console.Run the DDL script to initialize users, expense_requests, and audit_logs tables, alongside sequences and trigger functions.App Import:Import the provided application blueprint JSON file into your Retool workspace.Session Configuration:Ensure your login query populates the app_user state variable with the correct user id, email, and role.
