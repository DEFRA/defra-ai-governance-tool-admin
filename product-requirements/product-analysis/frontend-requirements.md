# Frontend Requirements Specification

This document outlines the frontend requirements based on the provided mockups, incorporating GDS (Government Digital Service) design principles and detailing both page routing and GDS-style component usage. Each page corresponds to the key application views shown in the wireframes. The goal is to ensure a clear, consistent user experience that aligns with GDS standards.

## 1. Project Pages

### 1.1 Project Listing Page

**Route:**  
`GET /projects` → `/projects` (frontend route)

**Purpose:**  
Displays a list of all Projects, each linking to its detail page. Allows the user to create a new Project.

**Layout & GDS Components:**
1. **Page heading:** Use a GDS "Page heading" (`<h1>` in GDS styles) titled "Projects"
2. **Project list:** Present each project name as a GDS-style link that navigates to the Project Detail page:
   - If only a small number of projects is expected, a simple list or stacked links is fine
   - If many projects are expected, consider using a GDS table or search/filter pattern
3. **Add New button:**
   - A GDS button labeled "Add New" at the bottom (or top-right)
   - This button links to the New Project page (`/projects/new`)

**Data/Actions:**
- GET the list of projects from the `/api/v1/project` endpoint
- Each project name is linked by its `_id`: `/projects/{projectId}`

### 1.2 New Project Page

**Route:**  
`GET /projects/new` → shows the form  
`POST /projects` → handled upon submission

**Purpose:**  
Allows the user to create a new Project by specifying the name, selecting a governance template, and choosing which workflows to include.

**Layout & GDS Components:**
4. **Page heading:** "Add New Project"
5. **Form fields** (all implemented with GDS form components):
   - **Project name (required):**
     - GDS Text input component labeled "Project name"
     - Use GDS field validation if empty
   - **Governance Template dropdown:**
     - GDS "Select" component listing available Governance Templates by `name + version`
     - On change, reload or dynamically show the list of available workflows below
   - **Active Governance Workflows (checkboxes):**
     - A GDS "Checkboxes" component listing the WorkflowTemplates from the selected Governance Template
     - Each workflow that can be activated for this project is shown with a label = workflow `name`
     - The user can check any that apply
6. **Save button** (GDS "button" component):
   - Submits the form data to create the project
   - On success, redirect to Project Detail page for the newly created project

**Data/Actions:**
- GET the list of Governance Templates (`/api/v1/governance-template`)
- On selection of a particular Governance Template, GET its workflows (`/api/v1/workflow-template?governanceTemplateId=...`) and present them as checkboxes
- POST to `/api/v1/project` on form submission
- After creation, redirect to `GET /projects/{newProjectId}` (the Project Detail page)

### 1.3 Project Detail Page

**Route:**  
`GET /projects/{id}` → `/projects/:projectId` (frontend route)

**Purpose:**  
Shows details of the selected project (e.g., name, template version used) and the instantiated workflows and checklist items for that project.

**Layout & GDS Components:**
1. **Page heading:**
   - "Project: `<ProjectName>`"
   - Underneath, a short description or subtitle if available
   - Optionally, show the governance template name and version in smaller text beneath the main heading
2. **List of Workflow Instances:**
   - For each active workflow in this Project, show a subheading with the workflow name (`<h2>` in GDS styles)
   - Under each workflow, list its Checklist Items
3. **Checklist Items** (mirroring your mockup):
   - Each item is displayed with a GDS "Checkbox" or "Radio button" style if appropriate, or a more advanced UI if it has special requirements (document upload, approval, etc.)
   - If the item requires a document or an approval:
     - Provide a GDS "File upload" pattern, or a link to a separate upload modal/page
     - Provide an "Approval" button or link if required (e.g., for sending an approval request)
   - If the item is not required, a GDS link or button labeled "Mark Not Required" that triggers a confirmation or form for an explanation
   - If dependencies are not met, either disable or visually indicate that the item cannot yet be completed, with a GDS "Inset text" explaining which items must be done first
4. **Optional Tools/Buttons:**
   - "Generate" buttons next to items if the backend supports generating documents. Use GDS Secondary buttons or links labeled "Generate"
   - "Upload Approval" or "Upload Document" buttons if required for that item's completion
5. **Navigation links:**
   - A "Back to Projects" link or button at the top or bottom (per GDS patterns, typically near the top-left or bottom) returning to `/projects`

**Data/Actions:**
- GET the project from `/api/v1/project/{id}`
- GET the project's workflow instances from `/api/v1/workflow-instance?projectId={id}` (or included in the same call if the backend aggregates)
- GET the checklist item instances from `/api/v1/checklist-item-instance?workflowInstanceId=...`
- For updates (e.g., marking an item "complete"):
  - PUT the relevant ChecklistItemInstance or create an AuditLog entry as needed
- For file uploads:
  - Possibly open a separate page or modal that POSTs to `/api/v1/document-upload`

## 2. Governance Template Pages

### 2.1 Governance Template Listing Page

**Route:**  
`GET /governance-templates` → `/governance-templates`

**Purpose:**  
Lists all governance templates, showing each name/version and linking to its detail page.

**Layout & GDS Components:**
6. **Page heading:** "Governance Templates"
7. **List (or table) of templates:**
   - Each item labeled as `name (version)` is a link to `/governance-templates/{id}`
8. **Add New button:**
   - A GDS button labeled "Add New," linking to `/governance-templates/new`

**Data/Actions:**
- GET from `/api/v1/governance-template`
- Navigate to the detail page of a template on click

### 2.2 New Governance Template Page

**Route:**  
`GET /governance-templates/new` → displays form  
`POST /governance-templates` → upon submission

**Purpose:**  
Allows creating a new Governance Template (with at least a name and version).

**Layout & GDS Components:**
9. **Page heading:** "Add Governance Template"
10. **Form fields:**
   - Governance Template Name (GDS text input)
   - Governance Template Version (GDS text input)
   - (Optionally, a text area for "description" if you wish to support it)
11. **Save button:** GDS style

**Data/Actions:**
- POST to `/api/v1/governance-template`
- On success, redirect to `/governance-templates/{newId}` (the new template's detail page)

### 2.3 Governance Template Detail Page

**Route:**  
`GET /governance-templates/{id}`

**Purpose:**  
Shows the details of a specific Governance Template, including its version, name, and list of Workflow Templates.

**Layout & GDS Components:**
12. **Page heading:** `Template: <name> - v<version>`
13. **List of Workflow Templates** (within this Governance Template):
   - Each workflow is a link to the Workflow Template Detail page (e.g. `/governance-templates/{id}/workflows/{workflowId}`)
14. **Add New Workflow button:**
   - A GDS button labeled "Add New Workflow"
   - Links to `/governance-templates/{id}/workflows/new`

**Data/Actions:**
- GET from `/api/v1/governance-template/{id}` for template info
- GET from `/api/v1/workflow-template?governanceTemplateId={id}` for the workflows in that template

## 3. Workflow Template Pages

### 3.1 New Workflow Template Page

**Route:**  
`GET /governance-templates/{id}/workflows/new` → shows form  
`POST /workflow-template`

**Purpose:**  
Allows the user to add a new workflow (WorkflowTemplate) to an existing Governance Template.

**Layout & GDS Components:**
15. **Page heading:** "Add Workflow Template"
   - Subheading or small text: `Template: <GovernanceTemplateName> - v<version>`
16. **Form fields:**
   - Workflow Name (GDS text input)
   - (Optional) Description or metadata (depending on your scope)
17. **Save button:** GDS style

**Data/Actions:**
- POST to `/api/v1/workflow-template` with `governanceTemplateId` set to the currently viewed template's ID
- On success, redirect to the Workflow Template Detail page for the newly created workflow

### 3.2 Workflow Template Detail Page

**Route:**  
`GET /governance-templates/{governanceTemplateId}/workflows/{workflowTemplateId}`

**Purpose:**  
Displays a specific WorkflowTemplate's details and lists its Checklist Item Templates.

**Layout & GDS Components:**
18. **Page heading:**
   - "Workflow Template: <WorkflowName>"
   - Subheading: `Template: <GovernanceTemplateName> - v<version>`
19. **Checklist Items section:**
   - For each ChecklistItemTemplate, show:
     - An ID or `itemKey` for reference (could be in smaller text)
     - A Name (GDS text input or read-only text)
     - Requires Approval (checkbox)
     - File Upload (checkbox) if relevant
     - Dependencies (a text input listing dependent item keys or IDs)
     - Save button (GDS) next to each checklist item to apply changes immediately
   - Add New Checklist Item button at the bottom, linking to the "New Checklist Item Template Page"

**Data/Actions:**
- GET from `/api/v1/workflow-template/{workflowTemplateId}` for workflow details
- GET from `/api/v1/checklist-item-template?workflowTemplateId={workflowTemplateId}` for all checklist item templates within this workflow
- PUT each item individually when the user clicks "Save"

## 4. Checklist Item Template Pages

### 4.1 New Checklist Item Template Page

**Route:**  
`GET /governance-templates/{govTemplateId}/workflows/{workflowTemplateId}/checklist-items/new` → shows the form  
`POST /api/v1/checklist-item-template` → on submission

**Purpose:**  
Allows the user to add a new Checklist Item Template to a given Workflow Template.

**Layout & GDS Components:**
20. **Page heading:** "Add Checklist Item Template"
   - Subheading: `Template: <GovernanceTemplateName> - v<version>`
   - `Workflow: <WorkflowName>`
21. **Form fields:**
   - Name (GDS text input)
   - Requires Approval (GDS checkbox)
   - File Upload (GDS checkbox)
   - Dependent Checklist Items (GDS text input or a multi-select if you want a more advanced UI)
22. **Save button** (GDS): Creates the new Checklist Item Template

**Data/Actions:**
- POST to `/api/v1/checklist-item-template` with `workflowTemplateId` set to `workflowTemplateId`
- After success, redirect to the Workflow Template Detail page to see the newly added item

## 5. General GDS Design Notes

### Typography & Headings
- Follow [GDS Typography](https://design-system.service.gov.uk/styles/typography/) with consistent heading levels (`<h1>` for page titles, `<h2>` for sub-sections, etc.)
- Use GDS classes (e.g. `.govuk-heading-xl`, `.govuk-heading-l`, etc.) as appropriate

### Form Layout & Validation
- Use [GDS form patterns](https://design-system.service.gov.uk/components/text-input/) with clear labels, hint text if necessary, and error messages displayed using the standard [GDS error summary](https://design-system.service.gov.uk/components/error-summary/)

### Buttons and Links
- Use [GDS buttons](https://design-system.service.gov.uk/components/button/) for primary actions (e.g. "Save", "Add New")
- Use [GDS link styles](https://design-system.service.gov.uk/styles/typography/#links) for navigational links

### Checkboxes & Radios
- For multi-select fields (like picking which workflows apply to a project), use [GDS checkboxes](https://design-system.service.gov.uk/components/checkboxes/)
- For single-select fields, consider GDS radio buttons or a dropdown (based on the design)

### Tables or Lists
- Where you need tabular data (like listing multiple items with columns for name, version, etc.), use the [GDS table component](https://design-system.service.gov.uk/components/table/)
- For simpler lists, GDS styled bullet or unnumbered lists can suffice

### Back Links
- Consider using a GDS "Back link" above the page heading to help users navigate step by step

### File Uploads
- If direct file upload within the form is needed, follow the [GDS file upload guidance](https://design-system.service.gov.uk/components/file-upload/)
- If large or multiple documents are expected, you may use a separate page or modal to handle them

### Accessibility & Usability
- Ensure every interactive element has accessible labels and helper text as needed
- Provide inline help or additional content (e.g. "When to mark something as 'Not Required'" or "What is an Approval?") using GDS inset text or GDS details components

### Responsive Design
- All GDS components are responsive by default; ensure your layouts follow the fluid grid approach and test on multiple device sizes

## 6. Routing Summary

A recommended (but flexible) routing structure could look like this:

### Projects
- `/projects` (list)
- `/projects/new` (create)
- `/projects/:projectId` (detail)

### Governance Templates
- `/governance-templates` (list)
- `/governance-templates/new` (create)
- `/governance-templates/:templateId` (detail)

### Workflow Templates
- `/governance-templates/:templateId/workflows/new`
- `/governance-templates/:templateId/workflows/:workflowId` (detail)

### Checklist Item Templates
- `/governance-templates/:templateId/workflows/:workflowId/checklist-items/new`

This structure cleanly nests workflow templates under their governance template and checklist items under their workflow.

## 7. Putting It All Together

When building the frontend:

23. Global navigation might have quick links to "Projects" and "Governance Templates," reflecting your top-level resources
24. Users generally start on the Project Listing page to either see existing projects or create a new one:
   - Creating a new project leads them to pick a template and relevant workflows
   - Viewing a project leads them to see (and complete) its active governance tasks
25. Administrators or advanced users might manage governance templates (create new versions, define or edit workflows, add or remove checklist items) in the Governance Templates section
26. The entire UI uses GDS patterns to ensure clarity, consistency, and accessibility

By following these page flows and GDS design principles, your application will provide a straightforward, compliant user experience for managing governance processes. Each page's data is tied directly to the corresponding backend API endpoints, maintaining a consistent, RESTful approach across the system.
