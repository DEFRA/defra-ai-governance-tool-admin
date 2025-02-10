# Governance Checklist Application – Detailed PRD (Augmented)

This document details the features for both the backend API (built on Node.js/Express with MongoDB) and the frontend (built using React and following GOV.UK design standards). Each user story includes endpoint specifications, frontend design notes, BDD acceptance criteria, and architectural design details. It now also includes expanded details on how the interface prototype (with upload and approval mechanisms) is supported and how administrators configure the various checklist item types.

## 1. Architectural Design Notes

### Backend (API)

#### Platform & Framework
- Node.js with Express

#### Database
MongoDB with collections for:
- GovernanceTemplate
- WorkflowTemplate
- ChecklistItemTemplate
- Project
- WorkflowInstance
- ChecklistItemInstance
- DocumentUpload
- AuditLog

#### Security
- Use HTTPS for API endpoints
- Authentication via JWT tokens and role-based access control

#### File Storage
- Integration with cloud storage (e.g., AWS S3) for document uploads

#### Audit Logging
- Middleware to capture state changes, file uploads, and dependency adjustments

#### API Style
- RESTful endpoints with JSON payloads

#### Scalability
- Proper indexing on MongoDB fields for dependency lookups
- Horizontal scaling for both the API and database as needed

### Frontend

#### Framework & Libraries
- React (or similar SPA framework)
- GOV.UK Frontend for consistent design and accessibility

#### Design Principles
- GOV.UK design standards (typography, spacing, color, accessibility)
- Responsive design for desktop and tablet
- Clear error states, inline validation, and contextual help

#### Component Structure
Separate components for:
- Template Management (governance templates, workflows, checklist items)
- Project Creation (wizard to pick template snapshots)
- Checklist Display (for end users to see and update item status)
- Document Upload modals or inline blocks
- Admin Setup screens that define different item types and dependencies

#### State Management
Use Redux or React Context for global state, especially around:
- Audits
- Checklist dependencies
- Current project / user context

## 2. Agile User Stories with Detailed Specifications

### User Story 1: Governance Template Management
No changes beyond existing PRD details.

As a Governance Administrator, I want to create, retrieve, and update governance templates so that I can define a version-controlled blueprint for the governance process.

#### Tasks & API Endpoints
- Create GovernanceTemplate (POST /api/governance/templates)
- Get GovernanceTemplate (GET /api/governance/templates/:id)
- Update GovernanceTemplate (PUT /api/governance/templates/:id)

#### Frontend Design Notes
- Form Layout: GOV.UK form components with version and description fields
- Navigation: Admin dashboard listing all templates

#### BDD Acceptance Criteria
(See original PRD.)

### User Story 2: Workflow Template Management
No changes beyond existing PRD details.

As a Governance Administrator, I want to add and manage workflow templates under a governance template so that I can define sub-processes for different business areas.

#### Tasks & API Endpoints
- Add WorkflowTemplate (POST /api/governance/templates/:id/workflows)

#### Frontend Design Notes
- Form Components: Possibly wizard-style to handle multi-step processes
- UI Cues: Inline validations and tooltips for metadata fields

#### BDD Acceptance Criteria
(See original PRD.)

### User Story 3: Checklist Item Template Management
Expanded to address different item "types" (e.g., "Generate Doc," "Approval," "Upload," "Mark Not Required") and dependencies.

As a Governance Administrator, I want to create checklist item templates within a workflow template so that I can specify tasks (of various types) with dependencies and configurations.

#### Key Requirements From the Prototype
- Checkbox Items: Items can be shown as required or not, with the ability to mark them complete or not required
- Generate Document: Some items require generating or linking to a document (e.g., "SDP doc v1.0")
- Upload & Approvals: Some items require file uploads (e.g., "Upload Approval Document") and an approval action by a specific role or user
- Dependency Indicators: Items can be blocked until a dependency is met
- Mark Not Required: Administrators can define certain item types or states where an end user can skip/mark the item "Not Required," including a reason

#### Tasks & API Endpoints

##### Endpoint: Add ChecklistItemTemplate
- Method: POST
- URL: /api/workflows/:workflowId/checklist-items
- Payload example:

```json
{
  "itemKey": "approval-1",
  "name": "Upload Approval Document",
  "description": "Upload evidence of approval",
  "type": "approval",
  "config": { 
    "requiresDocument": true, 
    "allowMarkNotRequired": false
  },
  "dependencies": ["doc-1"]
}
```

type can be:
- "generateDocument" (triggers a "Generate" button in the UI)
- "upload" (requires file uploads)
- "approval" (tracked by an approver's name and date)
- "check" (simple checklist/boolean item)
- "optional" (can be marked "Not Required" by an authorized role)

config can hold specialized flags (e.g., requiresDocument, allowMarkNotRequired, etc.).
dependencies references other item keys in the same workflow.

Response: Created ChecklistItemTemplate object.

#### Frontend Design Notes

##### Form Structure
- Fields for item key, name, type (dropdown for "generateDocument," "upload," "approval," etc.)
- Config inputs (check boxes or toggles) to define behavior like "requiresDocument," "allowMarkNotRequired"
- A multi-select/tagging component for dependencies

##### Usage in the Prototype
- Generate Document items: The UI shows a "Generate" link or button (e.g., "Generate Architecture Documentation")
- Approval items: The UI includes an "Upload Approval" or "Approve" button that sets the item's state to approved with date/time and user info
- Mark Not Required items: An inline option or link reading "Mark Not Required" plus a text area for the reason

#### BDD Acceptance Criteria

```gherkin
Scenario: Create a new checklist item template with different item types
  Given a governance administrator is editing a workflow template
  When they enter the checklist item details (name, type, config, dependencies) and submit
  Then the system should create a new ChecklistItemTemplate
  And display it in the workflow's list of checklist items
```

### User Story 4: Project Setup and Template Snapshot
No changes beyond existing PRD details.

As a Project Administrator, I want to create a new project using a specific governance template version and select applicable workflows so that the project uses a snapshot of the approved process.

#### Tasks & API Endpoints
- Create Project (POST /api/projects)
- Instantiates WorkflowInstances and ChecklistItemInstances

#### Frontend Design Notes
- Wizard Interface: Pick governance template, select workflows, confirm a "snapshot"

#### BDD Acceptance Criteria
(See original PRD.)

### User Story 5: User Checklist Interface – Viewing & Updating Checklist Items
Expanded to support "Generate," "Not Required," and "Dependency" logic in the UI.

As an End User, I want to view my project's checklist (grouped by workflow) and update the state of checklist items so that I can track my progress.

#### Tasks & API Endpoints

##### Retrieve Consolidated Checklist
- Method: GET
- URL: /api/projects/:id/checklist
- Response: Groups ChecklistItemInstances by WorkflowInstance, including item type, state, dependencies, and any associated documents or approvals

##### Update Checklist Item State
- Method: PUT
- URL: /api/checklist-items/:id/state
- Payload example:

```json
{
  "state": "complete",
  "typeData": {
    "approvedAt": "2025-02-10T15:00:00Z",
    "approver": "John Doe"
  }
}
```

typeData is optional, depending on item type (e.g., approval info, generation time, or reason for "not required").
Response: Updated ChecklistItemInstance + an AuditLog entry.

#### Frontend Design Notes

##### Checklist Layout
- Accordion or tabbed layout grouped by workflow
- Each item shows:
  - Title, description
  - Current state (e.g., "Not Started," "Complete," "Not Required," "Approval Needed")
  - Dependencies (if unmet, show an icon and tooltip)
  - Action buttons ("Generate Document," "Upload," "Mark Complete," "Mark Not Required," etc.)
- Completed or "Not Required" items appear visually distinct (e.g., check icon, strikethrough)

##### Interaction Design
- If an item has unmet dependencies, any completion attempt is blocked with an inline error
- For "Generate Document" items:
  - Clicking "Generate" might just store a timestamp or create a placeholder doc link
  - Could open a modal or external link, then upon return mark the item as generated
- For "Mark Not Required":
  - A user with the right role sees a "Not Required" button. Prompt for a reason, store it in typeData or a specialized field

#### BDD Acceptance Criteria

```gherkin
Scenario: Marking an item as "Not Required"
  Given an end user has an item that can be marked not required
  When they click "Mark Not Required" and provide a reason
  Then the system updates the checklist item state to "Not Required"
  And logs the reason and user information
  And displays that status in the UI with the reason

Scenario: Generating a document for an item
  Given an item is of type "generateDocument"
  When a user clicks the "Generate" button
  Then the system records a generation timestamp
  And if needed, prompts the user to upload or link the generated file
  And marks the item as "inProgress" or "complete," depending on config
```

### User Story 6: Document Uploads for Checklist Items
As an End User, I want to upload one or more documents for a checklist item so that I can provide evidence or related documentation.

#### Tasks & API Endpoints
- Upload Document (POST /api/checklist-items/:id/upload)
  - Multipart form-data with file
  - S3 (or similar) for storage
  - AuditLog entry created

#### Frontend Design Notes
##### File Upload UI
- GOV.UK styled file upload components
- Show progress and file list on completion
- Validate type and size before submission

##### Approvals with Upload
- Items of type "approval" might require an uploaded file (e.g., "SDP doc v1.0") followed by a sign-off

#### BDD Acceptance Criteria
(See original PRD.)

### User Story 7: Audit Logging & Dependency Management
No changes beyond existing PRD details.

As a System, I want every important change (checklist state change, document upload, dependency override) to be recorded in an immutable audit log for compliance and traceability.

#### Tasks & Implementation Details
- Automatic Audit Logging of all state changes and uploads
- Dependency Checks on each state change

#### BDD Acceptance Criteria
(See original PRD.)

## 3. Admin Functions and Configuring Different Checklist Item Types

A critical enhancement to support the interface in the prototype is the ability for admins to configure multiple item "types" within the same workflow, each with distinct behaviors. The system must allow:

### Defining Item Types
- Admins can create or edit a "ChecklistItemTemplate," specifying type (e.g., "generateDocument", "upload", "approval", "check", "optional") and config fields
- "GenerateDocument" items display a "Generate" link or button in the UI
- "Upload" items include an upload widget
- "Approval" items include an "Approve" button (potentially with a user role restriction)
- "Optional" items can be "marked not required" by certain roles

### Dependency Configuration
- For each item, admins can specify any other item keys as dependencies
- The UI prevents completing an item if its dependencies are not all complete (or not required)

### Not Required Logic
- An admin might define that an item can be skipped entirely. For instance, a "Content Assessment" might be "Not Required" for a backend-only service
- When the user marks the item "Not Required," they must provide a reason (stored in typeData.reason) and the system logs it in AuditLog

### Examples

#### Spend Control
- Type: "check"
- Dependencies: ["SDP_Approval"]
- On the UI, the item remains disabled or "blocked" until the "Service Design Pack Review" is completed

#### Content Assessment
- Type: "optional"
- Config: { "allowMarkNotRequired": true }
- If the user has a reason (e.g., "Backend service only"), they can mark it "Not Required"

#### Service Design Pack (SDP)
- Type: "upload"
- Config: { "requiresDocument": true }
- The item is only considered "complete" once a document is uploaded

## 4. Interface Prototype Details and Usage Examples

Below are scenario-based examples illustrating how the final UI will work with the newly defined item types:

### User Views a Governance Checklist
- The user sees items like "GRIP Meeting," "Service Design Pack (SDP)," "Service Design Pack (SDP) Review," "Spend Control," and so on
- Each item has a checkbox or button to mark it "Complete" or "Not Required," depending on the item's config

### Generate a Document
- For "Alpha Service Assessment" with type "generateDocument," the user sees a "Generate" link. Clicking it updates the item's status to "Generated," perhaps storing a doc link

### Upload and Approve
- For "Service Design Pack (SDP) Review," the user must upload an approval document. Once uploaded, the item can be marked "Approved" by an authorized role. The system logs "SDP Approved 2025-01-10 by Mary A."

### Mark an Item Not Required
- "Content Assessment" is shown with a "Mark Not Required" link. Clicking it prompts for a reason. After submission, the UI displays:
  - Not Required Reason: "Content assessment deemed not relevant to this backend service. (Todd Anderson, 2024-01-15)"

### Dependency Example
- "Spend Control" item is locked until "Service Design Pack (SDP) Review Approval" is complete. If the user tries to complete it prematurely, the UI shows an error: "Unmet Dependency: Spend Control Document."

## 5. Detailed API Endpoints Summary

Unchanged from original PRD except for clarifications about new item types.

| Endpoint | Method | Purpose | Notes |
|----------|---------|---------|--------|
| /api/governance/templates | POST | Create a new GovernanceTemplate | Validates versioning and required fields |
| /api/governance/templates/:id | GET | Retrieve a GovernanceTemplate | Returns template details, optionally with workflows |
| /api/governance/templates/:id | PUT | Update an existing GovernanceTemplate | Partial updates allowed |
| /api/governance/templates/:id/workflows | POST | Create a new WorkflowTemplate | Captures metadata for business applicability |
| /api/workflows/:workflowId/checklist-items | POST | Create a new ChecklistItemTemplate | Supports multiple item types, config, and dependencies |
| /api/projects | POST | Create a new project & snapshot selected templates | Automatically instantiates WorkflowInstances and ChecklistItemInstances |
| /api/projects/:id/checklist | GET | Retrieve consolidated checklist for a project | Returns grouped checklist items by workflow |
| /api/checklist-items/:id/state | PUT | Update checklist item state (dependency validation) | Ensures active dependencies are met before updating state; logs an AuditLog entry |
| /api/checklist-items/:id/upload | POST | Upload a document associated with a checklist item | Handles multipart form-data; logs an AuditLog entry |

## 6. Frontend Design Notes (GOV.UK Standards)

Emphasizing the new item types and "Mark Not Required."

### Governance Admin Screen
- Dashboard listing templates, with expansion for workflows and their checklist items
- "Create Checklist Item" form includes type dropdown, config toggles, and dependencies fields

### Project Admin Screen
- Wizard to create a project, pick governance template, and choose relevant workflows
- Final "Preview" step shows the "snapshot" of items that will appear in the project

### User Checklist Screen
- Accordion or tabbed interface grouping items by workflow
- Items of type "generateDocument" show a "Generate" button
- Items of type "