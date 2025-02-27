# Initial Product Requirements

This document outlines the requirements and design for our Governance Checklist System. The system uses version‑controlled templates—**GovernanceTemplates** that include **WorkflowTemplates** and **ChecklistItemTemplates**—to instantiate project‑specific workflows (via **WorkflowInstances** and **ChecklistItemInstances**).

**Recent Updates:**
- The legacy `itemKey` and `dependencies` fields have been removed from checklist item templates.
- Dependencies are now tracked using the `dependencies_requires` array (an array of ObjectIds). API endpoints will accept this array as a list of objectId strings.
- In addition, GET endpoints for checklist item templates now also return a computed field, `dependencies_requiredBy`, which contains all ChecklistItemTemplate objects that reference the current object in their `dependencies_requires` array. This field is not stored in the database but is computed at runtime.
- Completed features and stories are marked with a ✅.

---

## Schema & Database Setup

Below is the current MongoDB configuration, including schema validations. Notice that the **checklistItemTemplates** schema now includes `dependencies_requires` in place of the old dependency structure. (The computed `dependencies_requiredBy` is added on GET and is not persisted in the database.)

```js
import { MongoClient } from 'mongodb'

import { LockManager } from 'mongo-locks'

  

import { config } from '~/src/config/index.js'

  

/**

* @satisfies { import('@hapi/hapi').ServerRegisterPluginObject<*> }

*/

export const mongoDb = {

plugin: {

name: 'mongodb',

version: '1.0.0',

/**

*

* @param { import('@hapi/hapi').Server } server

* @param {{mongoUrl: string, databaseName: string, retryWrites: boolean, readPreference: string}} options

* @returns {Promise<void>}

*/

register: async function (server, options) {

server.logger.info('Setting up MongoDb')

  

const client = await MongoClient.connect(options.mongoUrl, {

retryWrites: options.retryWrites,

readPreference: options.readPreference,

...(server.secureContext && { secureContext: server.secureContext })

})

  

const databaseName = options.databaseName

const db = client.db(databaseName)

const locker = new LockManager(db.collection('mongo-locks'))

  

await createIndexes(db)

await createSchemaValidations(db)

  

server.logger.info(`MongoDb connected to ${databaseName}`)

  

server.decorate('server', 'mongoClient', client)

server.decorate('server', 'db', db)

server.decorate('server', 'locker', locker)

server.decorate('request', 'db', () => db, { apply: true })

server.decorate('request', 'locker', () => locker, { apply: true })

  

// eslint-disable-next-line @typescript-eslint/no-misused-promises

server.events.on('stop', async () => {

server.logger.info('Closing Mongo client')

await client.close(true)

})

}

},

options: {

mongoUrl: config.get('mongoUri'),

databaseName: config.get('mongoDatabase'),

retryWrites: false,

readPreference: 'secondary'

}

}

  

/**

* @param {import('mongodb').Db} db

* @returns {Promise<void>}

*/

async function createIndexes(db) {

await db.collection('mongo-locks').createIndex({ id: 1 })

  

// Create indexes for governance templates

await db.collection('governanceTemplates').createIndex({ name: 1 })

await db.collection('governanceTemplates').createIndex({ version: 1 })

await db

.collection('governanceTemplates')

.createIndex({ name: 1, version: 1 }, { unique: true })

  

// Create indexes for workflow templates

await db

.collection('workflowTemplates')

.createIndex({ governanceTemplateId: 1 })

await db.collection('workflowTemplates').createIndex({ name: 1 })

await db

.collection('workflowTemplates')

.createIndex({ governanceTemplateId: 1, name: 1 }, { unique: true })

  

// Create indexes for checklist item templates

await db

.collection('checklistItemTemplates')

.createIndex({ workflowTemplateId: 1 })

}

  

async function createSchemaValidations(db) {

const validations = [

{

collection: 'governanceTemplates',

schema: {

bsonType: 'object',

required: ['name', 'version', 'createdAt', 'updatedAt'],

additionalProperties: false,

properties: {

_id: { bsonType: 'objectId' },

name: { bsonType: 'string' },

version: { bsonType: 'string' },

description: { bsonType: 'string', pattern: '^.*$' },

createdAt: { bsonType: 'date' },

updatedAt: { bsonType: 'date' }

}

}

},

{

collection: 'projects',

schema: {

bsonType: 'object',

required: [

'name',

'governanceTemplateId',

'selectedWorkflowTemplateIds',

'createdAt',

'updatedAt'

],

additionalProperties: false,

properties: {

_id: { bsonType: 'objectId' },

name: { bsonType: 'string' },

description: { bsonType: 'string' },

governanceTemplateId: { bsonType: 'objectId' },

selectedWorkflowTemplateIds: {

bsonType: 'array',

items: { bsonType: 'objectId' }

},

metadata: { bsonType: 'object' },

createdAt: { bsonType: 'date' },

updatedAt: { bsonType: 'date' }

}

}

},

{

collection: 'workflowTemplates',

schema: {

bsonType: 'object',

required: ['governanceTemplateId', 'name', 'createdAt', 'updatedAt'],

additionalProperties: false,

properties: {

_id: { bsonType: 'objectId' },

governanceTemplateId: { bsonType: 'objectId' },

name: { bsonType: 'string' },

description: { bsonType: 'string', pattern: '^.*$' },

metadata: { bsonType: 'object' },

createdAt: { bsonType: 'date' },

updatedAt: { bsonType: 'date' }

}

}

},

{

collection: 'workflowInstances',

schema: {

bsonType: 'object',

required: [

'projectId',

'workflowTemplateId',

'name',

'status',

'createdAt',

'updatedAt'

],

additionalProperties: false,

properties: {

_id: { bsonType: 'objectId' },

projectId: { bsonType: 'objectId' },

workflowTemplateId: { bsonType: 'objectId' },

name: { bsonType: 'string' },

description: { bsonType: 'string', pattern: '^.*$' },

metadata: { bsonType: 'object' },

status: { bsonType: 'string', enum: ['active', 'completed'] },

createdAt: { bsonType: 'date' },

updatedAt: { bsonType: 'date' }

}

}

},

{

collection: 'checklistItemTemplates',

schema: {

bsonType: 'object',

required: [

'workflowTemplateId',

'name',

'type',

'createdAt',

'updatedAt'

],

additionalProperties: false,

properties: {

_id: { bsonType: 'objectId' },

workflowTemplateId: { bsonType: 'objectId' },

name: { bsonType: 'string' },

description: { bsonType: 'string', pattern: '^.*$' },

type: { bsonType: 'string' },

dependencies_requires: {

bsonType: 'array',

items: { bsonType: 'objectId' }

},

metadata: { bsonType: 'object' },

createdAt: { bsonType: 'date' },

updatedAt: { bsonType: 'date' }

}

}

},

{

collection: 'checklistItemInstances',

schema: {

bsonType: 'object',

required: [

'workflowInstanceId',

'checklistItemTemplateId',

'name',

'type',

'status',

'createdAt',

'updatedAt'

],

additionalProperties: false,

properties: {

_id: { bsonType: 'objectId' },

workflowInstanceId: { bsonType: 'objectId' },

checklistItemTemplateId: { bsonType: 'objectId' },

name: { bsonType: 'string' },

description: { bsonType: 'string', pattern: '^.*$' },

type: { bsonType: 'string' },

status: {

bsonType: 'string',

enum: ['incomplete', 'complete', 'not_required']

},

dependencies_requires: {

bsonType: 'array',

items: { bsonType: 'objectId' }

},

metadata: { bsonType: 'object' },

createdAt: { bsonType: 'date' },

updatedAt: { bsonType: 'date' }

}

}

}

]

  

for (const { collection, schema } of validations) {

try {

await db.command({

collMod: collection,

validator: { $jsonSchema: schema },

validationLevel: 'strict',

validationAction: 'error'

})

} catch (error) {

if (error.codeName === 'NamespaceNotFound') {

await db.createCollection(collection, {

validator: { $jsonSchema: schema },

validationLevel: 'strict',

validationAction: 'error'

})

} else {

throw error

}

}

}

}

  

/**

* To be mixed in with Request|Server to provide the db decorator

* @typedef {{db: import('mongodb').Db, locker: import('mongo-locks').LockManager }} MongoDBPlugin

*/
```

---

## Feature 1: Governance Template Management (✅ Feature Completed)

**Feature Description:**  
Allow users to create, view, update, and delete governance process blueprints (GovernanceTemplates). These templates include version, name, and description and are used as snapshots for instantiating projects.

> **Note:** DELETE requests for GovernanceTemplates will cascade delete any child WorkflowTemplates and associated ChecklistItemTemplates.

### Story 1.1: Backend API for Governance Template CRUD (✅ Story Completed)
- **As a** backend developer, **I want** to implement CRUD operations for GovernanceTemplate using the `/api/v1/governance-templates` endpoints, **so that** users can manage governance blueprints.
- **Design / UX Consideration:**  
  Validate required fields (e.g., version, name, description) and return appropriate HTTP status codes and error messages.
- **Acceptance Criteria:**
  - **Given** a valid POST payload containing `version`, `name`, and `description`,  
    **When** a POST request is sent to `/api/v1/governance-templates`,  
    **Then** a new GovernanceTemplate is created and returned with a unique `_id`.
  - **Given** an existing GovernanceTemplate,  
    **When** a GET request is made to `/api/v1/governance-templates/{id}`,  
    **Then** the corresponding GovernanceTemplate object is returned.
  - **Given** an existing GovernanceTemplate,  
    **When** a PUT request is made to `/api/v1/governance-templates/{id}` with updated fields,  
    **Then** the GovernanceTemplate is updated and the new version is returned.
  - **Given** an existing GovernanceTemplate,  
    **When** a DELETE request is made to `/api/v1/governance-templates/{id}`,  
    **Then** the GovernanceTemplate is deleted, cascading the deletion to all child WorkflowTemplates and ChecklistItemTemplates, and a success message is returned.
- **Detailed Architecture Design Notes:**
  - Use an Hapi.js (or similar) router to define the CRUD endpoints.
  - Integrate with a document database (e.g., MongoDB) for persistence.
  - Implement input validation and error handling middleware.
  - Optionally, record state changes in the AuditLog if required.

---

### Story 1.2: Frontend UI for Governance Template Listing and Detail Pages (✅ Story Completed)
- **As a** user, **I want** to view a list of all GovernanceTemplates with their names and versions, **so that** I can select a template to inspect its details.
- **Design / UX Consideration:**  
  Use GDS "Page heading" components, lists or tables, and clearly labeled links that navigate to the detail page.
- **Acceptance Criteria:**
  - **Given** that one or more GovernanceTemplates exist,  
    **When** the user navigates to `/governance-templates`,  
    **Then** a list of templates (displaying `name (version)`) is shown, with each item linking to its detail page.
  - **Given** a template link is clicked,  
    **When** the detail page loads,  
    **Then** the page displays the template's `name`, `version`, `description`, and a list of its associated WorkflowTemplates.
- **Detailed Architecture Design Notes:**
  - Consume the `/api/v1/governance-templates` endpoint to retrieve the list.
  - Implement routing so that clicking a template navigates to `/governance-templates/{id}`.
  - Use Node.js + Hapi.js, govuk-frontend npm library, and Nunjucks templates with progressive enhancement.

---

### Story 1.3: Frontend UI for New Governance Template Creation (✅ Story Completed)
- **As a** user, **I want** to create a new GovernanceTemplate by entering a version, name, and description, **so that** I can define a new governance process blueprint.
- **Design / UX Consideration:**  
  Use GDS text input components with inline validation and error messaging. Ensure that the form is clear and accessible.
- **Acceptance Criteria:**
  - **Given** the user navigates to `/governance-templates/new`,  
    **When** the form is rendered,  
    **Then** fields for version, name, and description are clearly displayed.
  - **Given** the user submits the form with valid data,  
    **When** a POST request is made to `/api/v1/governance-templates`,  
    **Then** the new GovernanceTemplate is created and the user is redirected to its detail page.
- **Detailed Architecture Design Notes:**
  - Manage form state and validations on the client side.
  - Use Fetch to call the API endpoint.
  - Redirect upon successful creation using the router.

---

## Feature 2: Workflow Template Management (✅ Feature Completed)

**Feature Description:**  
Enable users to manage WorkflowTemplates—workflows that are part of a GovernanceTemplate. Users can create new workflows, update existing ones, and view workflow details.

> **Note:** DELETE requests for WorkflowTemplates will cascade delete any child ChecklistItemTemplates.

### Story 2.1: Backend API for Workflow Template CRUD (✅ Story Completed)
- **As a** backend developer, **I want** to implement CRUD operations for WorkflowTemplate using the `/api/v1/workflow-templates` endpoints, **so that** workflows can be managed and linked to their parent GovernanceTemplate.
- **Design / UX Consideration:**  
  Ensure that every WorkflowTemplate POST request includes a valid `governanceTemplateId`.
- **Acceptance Criteria:**
  - **Given** a valid POST payload containing `governanceTemplateId`, `name`, `description`, and `metadata`,  
    **When** a POST request is sent to `/api/v1/workflow-templates`,  
    **Then** a new WorkflowTemplate is created and returned.
  - **Given** an existing WorkflowTemplate,  
    **When** a GET request is made to `/api/v1/workflow-templates/{id}`,  
    **Then** the WorkflowTemplate is returned.
  - **Given** an existing WorkflowTemplate,  
    **When** a PUT request is made to `/api/v1/workflow-templates/{id}`,  
    **Then** the WorkflowTemplate is updated.
  - **Given** an existing WorkflowTemplate,  
    **When** a DELETE request is made to `/api/v1/workflow-templates/{id}`,  
    **Then** the WorkflowTemplate is deleted, cascading the deletion to all child ChecklistItemTemplates, and a success message is returned.
- **Detailed Architecture Design Notes:**
  - Validate the existence of `governanceTemplateId` (using a lookup to the GovernanceTemplate collection).
  - Log changes if necessary using AuditLog integration.

---

### Story 2.2: Frontend UI for Workflow Template Detail and Creation (✅ Story Completed)
- **As a** user, **I want** to view a WorkflowTemplate's details and add new workflows under a specific GovernanceTemplate, **so that** I can manage and expand the governance process.
- **Design / UX Consideration:**  
  Use GDS form components for the creation page and clearly display the workflow's metadata on the detail page. Ensure that the creation page is nested under the GovernanceTemplate detail view.
- **Acceptance Criteria:**
  - **Given** the user is on a GovernanceTemplate detail page,  
    **When** the "Add New Workflow" button is clicked,  
    **Then** the user is navigated to `/governance-templates/{id}/workflows/new` where a form for entering workflow details is shown.
  - **Given** valid workflow details are submitted,  
    **When** a POST request is made to `/api/v1/workflow-templates`,  
    **Then** the new WorkflowTemplate is created and the user is redirected to its detail page.
  - **Given** a WorkflowTemplate exists,  
    **When** its detail page is loaded,  
    **Then** it displays its name, description, and metadata.
- **Detailed Architecture Design Notes:**
  - Use routing to maintain the relationship between GovernanceTemplates and WorkflowTemplates.
  - Use form state management and API integration for creation.

---

## Feature 3: Checklist Item Template Management (✅ Feature Completed)

**Feature Description:**  
Enable management of ChecklistItemTemplates that define the tasks within a workflow. These templates include configuration details, type definitions, and dependency references.  
**Notes:**
- The legacy `itemKey` and `dependencies` fields have been removed.
- Dependencies are now tracked via the `dependencies_requires` array (an array of ObjectId strings).
- **New:** All GET endpoints for ChecklistItemTemplates will return a computed field, `dependencies_requiredBy`, which contains all ChecklistItemTemplate objects that include the current object in their `dependencies_requires` array.

### Story 3.1: Backend API for Checklist Item Template CRUD  (✅ Story Completed)
- **As a** backend developer, **I want** to implement CRUD operations for ChecklistItemTemplate using the `/api/v1/checklist-item-templates` endpoints, **so that** each workflow can have its own defined tasks.
- **Design / UX Consideration:**  
  Validate that each POST includes required fields such as `workflowTemplateId`, `name`, and `type` (along with `createdAt` and `updatedAt`).  
  The optional `dependencies_requires` field must be an array of objectId strings. Do not accept whole objects in this field.
- **Acceptance Criteria:**
  - **Given** a valid POST payload (with `workflowTemplateId`, `name`, `type`, and optionally `dependencies_requires` as an array of objectId strings),
    **When** a POST request is made to `/api/v1/checklist-item-templates`,
    **Then** a new ChecklistItemTemplate is created and returned.
  - **Given** an existing ChecklistItemTemplate,
    **When** a GET request is made to `/api/v1/checklist-item-templates/{id}`,
    **Then** the ChecklistItemTemplate is returned with the populated `dependencies_requires` **and** the computed `dependencies_requiredBy` field.
    - The `dependencies_requiredBy` field includes all ChecklistItemTemplate objects that list the current template in their `dependencies_requires` array.
  - **Given** an existing ChecklistItemTemplate,
    **When** a PUT request is made,
    **Then** the template is updated accordingly.
  - **Given** an existing ChecklistItemTemplate,
    **When** a DELETE request is made,
    **Then** the template is deleted.
- **Detailed Architecture Design Notes:**
  - Remove the legacy `itemKey` and `dependencies` object.
  - Validate that each objectId in `dependencies_requires` references an existing checklist item template.
  - Ensure that on GET requests, the API populates both `dependencies_requires` and computes `dependencies_requiredBy`.

---

### Story 3.2: Frontend UI for Checklist Item Template Creation and Listing  (✅ Story Completed)
- **As a** user, **I want** to create new ChecklistItemTemplates and see a list of existing ones under a WorkflowTemplate, **so that** I can define and manage the tasks for that workflow.
- **Design / UX Consideration:**  
  Use GDS form inputs for fields such as name, type, and configuration.  
  For dependencies, provide a multi-select input that collects objectIds referencing existing checklist item templates.  
  The API accepts a list of objectId strings for the `dependencies_requires` field and returns fully populated dependency objects on GET (including the computed `dependencies_requiredBy`).
- **Acceptance Criteria:**
  - **Given** the user navigates to `/governance-templates/{govTemplateId}/workflows/{workflowTemplateId}/checklist-items/new`,
    **When** the page loads,
    **Then** a form with fields for name, type, configuration, and dependencies is displayed.
  - **Given** valid data is entered and the form is submitted,
    **When** a POST request is made to `/api/v1/checklist-item-templates`,
    **Then** the new checklist item is created and the user is redirected (or the list is updated) on the WorkflowTemplate detail page.
  - **Given** the user is on the WorkflowTemplate detail page,
    **When** the checklist items are rendered,
    **Then** all ChecklistItemTemplates for that workflow are displayed with both the populated `dependencies_requires` and computed `dependencies_requiredBy` fields.
- **Detailed Architecture Design Notes:**
  - Ensure that dependency selection correlates with existing checklist items in the workflow.
  - Validate form inputs before submission.

---

## Feature 4: Project Management (Instantiation of Templates) (✅ Feature Completed)

**Feature Description:**  
Manage projects that instantiate a selected, version‑controlled GovernanceTemplate along with chosen WorkflowTemplates. This feature creates live WorkflowInstances and ChecklistItemInstances to track progress.

### Story 4.1: Backend API for Project CRUD Operations (✅ Story Completed)
- **As a** backend developer, **I want** to implement CRUD operations for Projects using the `/api/v1/projects` endpoints, **so that** projects can be created with a selected GovernanceTemplate and associated workflows.
- **Design / UX Consideration:**  
  Validate that the provided `governanceTemplateId` exists and that each workflow in `selectedWorkflowTemplateIds` is valid. On creation, trigger the instantiation (snapshot) of WorkflowInstances and ChecklistItemInstances.
- **Acceptance Criteria:**
  - **Given** a valid POST payload containing `name`, `description`, `governanceTemplateId`, and `selectedWorkflowTemplateIds`,  
    **When** a POST request is sent to `/api/v1/projects`,  
    **Then** a new Project is created with the specified governance template and workflows.
  - **Given** an existing project,  
    **When** a GET request is made to `/api/v1/projects/{id}`,  
    **Then** the project details are returned.
  - **Given** an existing project,  
    **When** a PUT request is made,  
    **Then** the project is updated accordingly.
  - **Given** an existing project,  
    **When** a DELETE request is made,  
    **Then** the project is deleted.
- **Detailed Architecture Design Notes:**
  - Upon project creation, instantiate WorkflowInstance records (and corresponding ChecklistItemInstances) as a snapshot of the selected templates.
  - Ensure database consistency and reference integrity.

---

### Story 4.2: Frontend UI for Project Listing Page (✅ Story Completed)
- **As a** user, **I want** to view a list of all projects, **so that** I can navigate to a project's detail page.
- **Design / UX Consideration:**  
  Use GDS components (page headings, tables or lists, and links) to display project names that link to their detail pages.
- **Acceptance Criteria:**
  - **Given** that one or more projects exist,  
    **When** the user navigates to `/projects`,  
    **Then** a list of project names is displayed, with each linking to `/projects/{projectId}`.
- **Detailed Architecture Design Notes:**
  - Consume the `/api/v1/projects` endpoint to fetch projects.
  - Implement responsive design and clear navigation.

---

### Story 4.3: Frontend UI for New Project Creation (✅ Story Completed)
- **As a** user, **I want** to create a new project by entering a project name, selecting a GovernanceTemplate, and choosing applicable WorkflowTemplates, **so that** I can instantiate a project with the correct governance process.
- **Design / UX Consideration:**  
  Use a GDS dropdown for selecting GovernanceTemplates, dynamic checkboxes for available WorkflowTemplates (which load when a GovernanceTemplate is selected), and proper form validation.
- **Acceptance Criteria:**
  - **Given** the user navigates to `/projects/new`,  
    **When** the page loads,  
    **Then** a form with fields for project name, a dropdown listing GovernanceTemplates, and checkboxes for the corresponding WorkflowTemplates is displayed.
  - **Given** the user selects a GovernanceTemplate and the corresponding workflows load,  
    **When** the form is submitted with valid input,  
    **Then** a POST request is sent to `/api/v1/projects` and a new project is created.
  - **Given** a project is successfully created,  
    **When** the response is received,  
    **Then** the user is redirected to the new project's detail page.
- **Detailed Architecture Design Notes:**
  - Dynamically load workflows using the endpoint `/api/v1/workflow-templates?governanceTemplateId=...` upon GovernanceTemplate selection.
  - Manage form state and error handling.

---

### Story 4.4: Frontend UI for Project Detail with Workflow and Checklist Display (✅ Story Completed)
- **As a** user, **I want** to view a project's details—including its instantiated WorkflowInstances and ChecklistItemInstances—**so that** I can track progress and take action on individual tasks.
- **Design / UX Consideration:**  
  Use GDS headings and components to display each WorkflowInstance and nested ChecklistItemInstances. Provide actionable elements (e.g., checkboxes, buttons) for updating item states.
- **Acceptance Criteria:**
  - **Given** a project has been instantiated with WorkflowInstances,  
    **When** the user navigates to `/projects/{projectId}`,  
    **Then** the page displays the project's name, the GovernanceTemplate details (name and version), and a list of WorkflowInstances.
  - **Given** each WorkflowInstance is displayed,  
    **When** its ChecklistItemInstances are rendered,  
    **Then** each item shows its current state (e.g., "incomplete", "complete", or "not required") along with options for state updates or file uploads.
- **Detailed Architecture Design Notes:**
  - Fetch data from `/api/v1/workflow-instances?projectId={projectId}` and `/api/v1/checklist-item-instances?workflowInstanceId=...`.
  - Enable state updates via PUT requests to `/api/v1/checklist-item-instances/{id}`.

---

Below is the revised **Feature 5** that reflects the updated behavior of instantiating workflow and checklist item instances during project creation, as well as cleaning up dependency references for checklist items that aren’t included in the new project structure.

---

## Feature 5: Automated Workflow and Checklist Item Instance Management on Project Creation  (✅ Feature Completed)

**Feature Description:**  
When a new project is created via a POST to `/api/v1/projects`, the backend will automatically instantiate live copies of workflows and checklist items based on the selected workflow template IDs (provided in the `selectedWorkflowTemplateIds` field). Specifically, for each workflow template ID provided:

- **WorkflowInstances:**  
    A WorkflowInstance is created as a snapshot of the corresponding WorkflowTemplate, capturing the version and other relevant metadata at the time of project creation.
    
- **ChecklistItemInstances:**  
    For every ChecklistItemTemplate associated with each selected workflow, a ChecklistItemInstance is created with an initial default state (e.g., "incomplete"). Each instance copies over dependency definitions from its template.
    
- **Dependency Cleanup:**  
    As checklist item instances are created, their `dependencies_requires` arrays will be examined. Any dependency reference that points to a checklist item template not included in the instantiated set (because its parent workflow was not selected) will be removed. This cleanup ensures that each ChecklistItemInstance only retains dependency references to other checklist items that are actually part of the new project setup.
    

---

### Revised Story 5.1: Automatic Workflow Instance Creation on Project Creation  (✅ Story Completed)

- **As a** backend developer, **I want** the backend to automatically create WorkflowInstance records for each workflow template specified in `selectedWorkflowTemplateIds` when a project is created, **so that** the project has a live snapshot of the selected workflows.
- **Design / UX Consideration:**
    - Upon receiving a valid project creation request (with a `governanceTemplateId` and an array of `selectedWorkflowTemplateIds`), the system validates that the IDs exist.
    - For each selected workflow template, a WorkflowInstance is created, capturing the version and other snapshot details from the original WorkflowTemplate.
- **Acceptance Criteria:**
    - **Given** a valid project creation request,  
        **When** the POST `/api/v1/projects` endpoint is called,  
        **Then** a new Project is created and a WorkflowInstance is created for each workflow template ID provided in `selectedWorkflowTemplateIds`.

---

### Revised Story 5.2: Automatic Checklist Item Instance Creation on Project Creation  (✅ Story Completed)

- **As a** backend developer, **I want** the backend to automatically create ChecklistItemInstance records for every ChecklistItemTemplate associated with each selected workflow during project creation, **so that** every task is tracked within the project context.
- **Design / UX Consideration:**
    - For each WorkflowInstance created, the system fetches all associated ChecklistItemTemplates.
    - A ChecklistItemInstance is created for each template, copying fields such as name, type, and the original `dependencies_requires` array.
    - **Dependency Cleanup:**
        - Before finalizing the ChecklistItemInstance, its `dependencies_requires` array is checked.
        - Any dependency reference that points to a ChecklistItemTemplate that is not included in the instantiated workflows is removed.
- **Acceptance Criteria:**
    - **Given** a valid project creation request with selected workflows,  
        **When** ChecklistItemInstances are created for each associated ChecklistItemTemplate,  
        **Then** each instance is initialized with the default state (e.g., "incomplete") and its `dependencies_requires` array only includes references to checklist items that were instantiated as part of the project.

---

### Revised Story 5.3: Managing Checklist Item Instance States (✅ Story Completed)

- **As a** user, **I want** to update the state of checklist items (e.g., mark them as complete or not required) within the project, **so that** I can track progress accurately.
- **Design / UX Consideration:**
    - The UI will present each ChecklistItemInstance with options (e.g., checkboxes, radio buttons, or toggles) to update its state.
    - If an item has unmet dependencies (based on its cleaned-up `dependencies_requires`), the UI should disable interaction or provide inline guidance.
- **Acceptance Criteria:**
    - **Given** a checklist item instance is displayed on the project detail page,  
        **When** the user updates its state,  
        **Then** a PUT request is sent to update the ChecklistItemInstance and the UI reflects the new state.
    - **Given** a checklist item instance has dependencies that are not yet met,  
        **When** it is rendered,  
        **Then** it either appears disabled or includes guidance indicating which dependencies need to be completed.

---

**Detailed Architecture Design Notes:**

- **Transactional Instantiation:**  
    The project creation process (triggered by POST `/api/v1/projects`) should handle instantiation of WorkflowInstances and ChecklistItemInstances within a transaction. This ensures that if any part of the instantiation fails, the entire operation can be rolled back.
    
- **Snapshot Integrity:**  
    The instantiation process should capture the current version and metadata of the GovernanceTemplate, WorkflowTemplates, and ChecklistItemTemplates at the time of project creation, ensuring a consistent snapshot is maintained even if the templates change later.
    
- **Dependency Validation:**  
    As ChecklistItemInstances are created, their dependency references (`dependencies_requires`) must be validated against the set of ChecklistItemTemplates that have been instantiated for the project. Any reference to a non-instantiated checklist item should be removed.
    
- **Audit Logging (Optional):**  
    Consider logging the instantiation process (or any cleanup actions) via the Audit Log endpoints to provide traceability for the creation and modification of instance records.
    

---

This revised feature ensures that when a project is created, all related workflows and tasks are automatically instantiated, and their dependency structures are consistent with the selected project setup.
---

## Feature 6: Document Upload Management

**Feature Description:**  
Allow users to upload and manage documents (e.g., approval evidence) associated with specific ChecklistItemInstances. Each checklist item may have multiple document uploads.

### Story 6.1: Backend API for Document Upload CRUD Operations
- **As a** backend developer, **I want** to implement CRUD operations for DocumentUpload records using the `/api/v1/document-uploads` endpoints, **so that** supporting documents are properly stored and linked to checklist items.
- **Design / UX Consideration:**  
  Validate all required fields (such as `checklistItemInstanceId`, `uploadType`, `fileName`, `url`, `mimeType`, and `uploadedAt`) and store additional metadata.
- **Acceptance Criteria:**
  - **Given** a valid POST payload,  
    **When** a POST request is sent to `/api/v1/document-uploads`,  
    **Then** a new DocumentUpload record is created and returned.
  - **Given** an existing DocumentUpload record,  
    **When** a GET request is made to `/api/v1/document-uploads/{id}`,  
    **Then** the record is returned.
  - **Given** an existing DocumentUpload record,  
    **When** a PUT request is made,  
    **Then** the record is updated accordingly.
  - **Given** an existing DocumentUpload record,  
    **When** a DELETE request is made,  
    **Then** the record is deleted.
- **Detailed Architecture Design Notes:**
  - Ensure that file metadata and uploader information are stored correctly.

---

### Story 6.2: Frontend UI for Document Uploads on Checklist Items
- **As a** user, **I want** to upload supporting documents for checklist items that require evidence, **so that** I can fulfill process requirements (e.g., for approvals).
- **Design / UX Consideration:**  
  Use the GDS file upload component and provide clear status indicators (e.g., progress, success, error messages). Allow for multiple uploads per checklist item.
- **Acceptance Criteria:**
  - **Given** a checklist item instance that requires a document upload,  
    **When** the user interacts with the file upload element,  
    **Then** they can select a file and it is uploaded via a POST request to `/api/v1/document-uploads`.
  - **Given** a successful file upload,  
    **When** the upload completes,  
    **Then** the UI displays the file name and upload status, and the document is associated with the checklist item.
- **Detailed Architecture Design Notes:**
  - Integrate the file picker with the API using Fetch.
  - Allow for PUT operations to update the file metadata if needed.

---

## Feature 7: Audit Log Integration

**Feature Description:**  
Record and display audit logs for significant events (such as checklist item state changes and dependency overrides) to ensure full traceability and compliance. Audit logs are immutable and only support creation and retrieval.

### Story 7.1: Backend API for Audit Log Recording
- **As a** backend developer, **I want** to record audit log entries using the `/api/v1/audit-logs` endpoints, **so that** all important events are traceable and immutable.
- **Design / UX Consideration:**  
  AuditLog entries should be created with all relevant details (eventType, objectType, objectId, changedAt, changedBy, and details) and must not support updates or deletions.
- **Acceptance Criteria:**
  - **Given** an event (such as a checklist item state change) occurs,  
    **When** a POST request is made to `/api/v1/audit-logs`,  
    **Then** an audit log entry is created with the correct event details.
  - **Given** existing audit log entries,  
    **When** a GET request is made to `/api/v1/audit-logs`,  
    **Then** a list of audit log entries is returned.
- **Detailed Architecture Design Notes:**
  - Ensure that audit logs are written asynchronously if needed to avoid performance impacts.
- **Dependencies:**  
  Must be integrated with state change operations in Features 5 and 6.

---

### Story 7.2: Frontend UI for Viewing Audit Logs (Read-Only)
- **As a** user, **I want** to view a read‑only list of audit log entries, **so that** I can review the history of changes and ensure process compliance.
- **Design / UX Consideration:**  
  Use a GDS table component or similar to display audit logs in a clear, non‑editable format.
- **Acceptance Criteria:**
  - **Given** that audit logs exist,  
    **When** the user navigates to the audit log view,  
    **Then** a list of audit log entries (with eventType, objectType, timestamp, etc.) is displayed.
- **Detailed Architecture Design Notes:**
  - Call the GET `/api/v1/audit-logs` endpoint and render the results.

---

## Feature 8: Global Frontend Navigation and Routing (✅ Feature Completed)

**Feature Description:**  
Implement a consistent global navigation structure and routing across the application to enable users to easily access Projects, GovernanceTemplates, WorkflowTemplates, and ChecklistItemTemplates.

### Story 8.1: Frontend Global Navigation and Routing  (✅ Story Completed)
- **As a** user, **I want** a persistent navigation bar with links to "Projects" and "Governance Templates", **so that** I can easily switch between sections of the application.
- **Design / UX Consideration:**  
  Use GDS navigation components; ensure the design is responsive and accessible.
- **Acceptance Criteria:**
  - **Given** the application loads,  
    **When** the global navigation is rendered,  
    **Then** it includes clearly labeled links to `/projects` and `/governance-templates`.
  - **Given** a user clicks one of the navigation links,  
    **When** the target page loads,  
    **Then** the corresponding listing (e.g., Project Listing or Governance Template Listing) is displayed.
- **Detailed Architecture Design Notes:**
  - Implement routing using Node.js + Hapi.js, govuk-frontend npm library, and Nunjucks templates.
  - Ensure that navigation state is maintained between page transitions.

---

### Story 8.2: Frontend Back Link and Page Hierarchy Implementation
- **As a** user, **I want** clear back links on detail pages (such as Project Detail or Governance Template Detail), **so that** I can easily navigate back to the previous list or parent page.
- **Design / UX Consideration:**  
  Use the GDS "Back link" component placed prominently above the page heading.
- **Acceptance Criteria:**
  - **Given** the user is on a detail page (e.g., a Project Detail page),  
    **When** the page renders,  
    **Then** a back link is visible that, when clicked, navigates back to the corresponding listing page (e.g., `/projects`).
- **Detailed Architecture Design Notes:**
  - Dynamically generate the back link based on the current route and navigation history.

---

## Feature 9: Deep Duplication Endpoints (Backend)

**Feature Description:**  
Implement new backend endpoints that support "deep" duplication of objects. The duplication process should recursively duplicate an object along with its child objects. All duplicated objects should have unique names generated by appending "copy" with a timestamp to the original name.

### Story 9.1: Backend API for Deep Duplication of GovernanceTemplates
- **As a** backend developer, **I want** to create an endpoint (e.g., POST `/api/v1/governance-templates/{id}/duplicate`) that duplicates a GovernanceTemplate along with all its child WorkflowTemplates and ChecklistItemTemplates, **so that** users can quickly create a new version based on an existing template.
- **Acceptance Criteria:**
  - **Given** a valid GovernanceTemplate ID,  
    **When** a POST request is made to the duplication endpoint,  
    **Then** the entire GovernanceTemplate and its associated child objects are deep duplicated.
  - **And** each duplicated object's name is suffixed with `" copy <timestamp>"` to ensure uniqueness.
  - **And** a complete object tree reflecting the original structure is returned.

---

### Story 9.2: Backend API for Deep Duplication of WorkflowTemplates
- **As a** backend developer, **I want** to create an endpoint (e.g., POST `/api/v1/workflow-templates/{id}/duplicate`) that duplicates a WorkflowTemplate and all its child ChecklistItemTemplates, **so that** users can reuse existing workflows with modifications.
- **Acceptance Criteria:**
  - **Given** a valid WorkflowTemplate ID,  
    **When** a POST request is made to the duplication endpoint,  
    **Then** the WorkflowTemplate and its child ChecklistItemTemplates are deep duplicated.
  - **And** each duplicated object's name is updated by appending `" copy <timestamp>"` to the original name.
  - **And** the new WorkflowTemplate maintains all relationships with its duplicated ChecklistItemTemplates.

---

### Story 9.3: Backend API for Deep Duplication of ChecklistItemTemplates
- **As a** backend developer, **I want** to create an endpoint (e.g., POST `/api/v1/checklist-item-templates/{id}/duplicate`) that duplicates a ChecklistItemTemplate, **so that** users can quickly create variants of checklist items.
- **Acceptance Criteria:**
  - **Given** a valid ChecklistItemTemplate ID,  
    **When** a POST request is made to the duplication endpoint,  
    **Then** the ChecklistItemTemplate is duplicated.
  - **And** the duplicated checklist item's name is modified with `" copy <timestamp>"` to ensure it is unique.
  - **And** any nested or related child objects (if applicable) are also deep duplicated accordingly.

---

## Feature 10: Frontend Duplication Actions

**Feature Description:**  
Add duplication functionality to the frontend by incorporating a duplication button in the list rows for GovernanceTemplates, WorkflowTemplates, and ChecklistItemTemplates. The action must require user confirmation using a GOV.UK compliant confirmation dialog, and upon successful duplication, the page reloads to reflect the changes.

### Story 10.1: Frontend UI for Duplicating GovernanceTemplates
- **As a** user, **I want** a duplication button on each GovernanceTemplate row in the listing, **so that** I can duplicate a template.
- **Design / UX Consideration:**  
  The duplication action should prompt the user with a GOV.UK compliant confirmation modal/dialog before proceeding.
- **Acceptance Criteria:**
  - **Given** a GovernanceTemplate is listed,  
    **When** the user clicks the duplication button,  
    **Then** a confirmation dialog is presented.
  - **And** upon confirming the action, a POST request is sent to `/api/v1/governance-templates/{id}/duplicate`.
  - **And** the page reloads once the duplication is successful.

---

### Story 10.2: Frontend UI for Duplicating WorkflowTemplates
- **As a** user, **I want** a duplication button on each WorkflowTemplate row in the listing, **so that** I can duplicate a workflow template along with its child checklist items.
- **Design / UX Consideration:**  
  Ensure that the confirmation modal follows GOV.UK design guidelines.
- **Acceptance Criteria:**
  - **Given** a WorkflowTemplate is listed,  
    **When** the duplication button is clicked,  
    **Then** the user is prompted to confirm the action.
  - **And** on confirmation, a POST request is made to `/api/v1/workflow-templates/{id}/duplicate`.
  - **And** the page reloads after the duplication completes.

---

### Story 10.3: Frontend UI for Duplicating ChecklistItemTemplates
- **As a** user, **I want** a duplication button on each ChecklistItemTemplate row in the listing, **so that** I can duplicate a checklist item template.
- **Design / UX Consideration:**  
  The confirmation prompt should be consistent with GOV.UK standards.
- **Acceptance Criteria:**
  - **Given** a ChecklistItemTemplate is listed,  
    **When** the duplication button is clicked,  
    **Then** a confirmation dialog is shown.
  - **And** upon confirmation, a POST request is sent to `/api/v1/checklist-item-templates/{id}/duplicate`.
  - **And** the page reloads once the duplication is successful.

---

## Feature 11: Frontend Delete Button Enhancements  (✅ Feature Completed)

**Feature Description:**  
Enhance frontend tables by adding a delete button to each row, enabling users to remove GovernanceTemplates, WorkflowTemplates, and ChecklistItemTemplates directly from the listing. The deletion will invoke the existing backend DELETE endpoints and then reload the page to reflect changes.

### Story 11.1: Frontend UI for Deleting GovernanceTemplates  (✅ Story Completed)
- **As a** user, **I want** a delete button on each GovernanceTemplate row, **so that** I can remove a template.
- **Design / UX Consideration:**  
  Ensure the delete button is clearly visible and styled according to GOV.UK guidelines.
- **Acceptance Criteria:**
  - **Given** a GovernanceTemplate is listed,  
    **When** the delete button is clicked,  
    **Then** a DELETE request is sent to `/api/v1/governance-templates/{id}`.
  - **And** the page reloads to reflect the deletion.

---

### Story 11.2: Frontend UI for Deleting WorkflowTemplates  (✅ Story Completed)
- **As a** user, **I want** a delete button on each WorkflowTemplate row, **so that** I can remove a workflow template.
- **Design / UX Consideration:**  
  The delete button should be integrated seamlessly into the table layout with GOV.UK compliant styling.
- **Acceptance Criteria:**
  - **Given** a WorkflowTemplate is listed,  
    **When** the delete button is clicked,  
    **Then** a DELETE request is sent to `/api/v1/workflow-templates/{id}`.
  - **And** the page reloads upon successful deletion.

---

### Story 11.3: Frontend UI for Deleting ChecklistItemTemplates  (✅ Story Completed)
- **As a** user, **I want** a delete button on each ChecklistItemTemplate row, **so that** I can remove a checklist item template.
- **Design / UX Consideration:**  
  The delete button must be clearly accessible and follow the design standards set by GOV.UK.
- **Acceptance Criteria:**
  - **Given** a ChecklistItemTemplate is listed,  
    **When** the delete button is clicked,  
    **Then** a DELETE request is sent to `/api/v1/checklist-item-templates/{id}`.
  - **And** the page reloads after the deletion is complete.