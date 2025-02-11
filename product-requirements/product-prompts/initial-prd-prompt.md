please create a PRD out of the following requirements. create it in a downloadable markdown format in a single file. use best practices standards for a GDS-compliant application:

# Context
We are creating an application that allows users to use a checklist format for managing project governance so we can better track governance tasks that are complete and evidence them.

The app consists of a node.js frontend that uses GOV.UK style and formatting standards and a node.js backend api with a mongoDB database, that also uses best practices standards.

There is currently a template for the frontend and backend application, however, there is no functionality currently implemented regarding the governance checklist application we want to build.

# New features to implement

Given the attached data model, we would like to implement a basic flow for administering new governance checklists, then displaying the checklist items to a user.

The data model uses a template structure to create versions of template governance models that can then be applied to a project.  When applied to a project, the different template workflows are activated based on the type of project it is.  For the activated workflows, the corresponding template checklist items are also included.

At the point of project setup, the template governance version will be selected and applied.  This will copy over the activated workflows and workflow checklist items to the WorflowInstance and ChecklistItemInstance tables as a snapshot in time.  The project will then use these instance items to manage the project governance.

## Data model

In the backend, we should create clean schema files for the attached data model. these schemas and their corresponding node.js data model files should be used as a pattern throughout the application during development.  Any data interaction should go via this pattern.

## API Endpoints

All endpoints follow RESTful conventions. Below is a complete list of endpoints modeled after the sample `/api/v1/governance-template` endpoint, now including PUT requests for updating resources by ID. The Audit Log endpoints allow creation (POST) and retrieval (GET) but do not permit updating or deleting entries.

### Governance Template Endpoints

#### **Endpoint:** `/api/v1/governance-template`
- **POST request:**  
  Creates a new governance template object with all the fields passed in the body; returns the newly created object from the database.
- **GET request:**  
  Returns a list of all the governance template objects stored in the database.

#### **Endpoint:** `/api/v1/governance-template/{id}`
- **GET request:**  
  Returns the governance template object for the given id.
- **PUT request:**  
  Updates the governance template object for the given id with the fields provided in the request body; returns the updated object.
- **DELETE request:**  
  Deletes the governance template object for the given id and returns a success message with status code 200.

---

Workflow Template Endpoints

#### **Endpoint:** `/api/v1/workflow-template`
- **POST request:**  
  Creates a new workflow template object with the provided fields; returns the newly created object.
- **GET request:**  
  Returns a list of all workflow template objects stored in the database.

#### **Endpoint:** `/api/v1/workflow-template/{id}`
- **GET request:**  
  Returns the workflow template object for the specified id.
- **PUT request:**  
  Updates the workflow template object for the specified id with the provided fields; returns the updated object.
- **DELETE request:**  
  Deletes the workflow template object for the specified id and returns a success message with status code 200.

---

### Checklist Item Template Endpoints

#### **Endpoint:** `/api/v1/checklist-item-template`
- **POST request:**  
  Creates a new checklist item template object with the provided fields; returns the newly created object.
- **GET request:**  
  Returns a list of all checklist item template objects stored in the database.

#### **Endpoint:** `/api/v1/checklist-item-template/{id}`
- **GET request:**  
  Returns the checklist item template object for the specified id.
- **PUT request:**  
  Updates the checklist item template object for the specified id with the provided fields; returns the updated object.
- **DELETE request:**  
  Deletes the checklist item template object for the specified id and returns a success message with status code 200.

---

Project Endpoints

#### **Endpoint:** `/api/v1/project`
- **POST request:**  
  Creates a new project using the provided fields (including selected governance template version and workflow templates); returns the newly created project object.
- **GET request:**  
  Returns a list of all project objects stored in the database.

#### **Endpoint:** `/api/v1/project/{id}`
- **GET request:**  
  Returns the project object for the specified id.
- **PUT request:**  
  Updates the project object for the specified id with the provided fields; returns the updated object.
- **DELETE request:**  
  Deletes the project object for the specified id and returns a success message with status code 200.

---

Workflow Instance Endpoints

#### **Endpoint:** `/api/v1/workflow-instance`
- **POST request:**  
  Creates a new workflow instance by snapshotting the selected workflow template for a given project; returns the newly created workflow instance.
- **GET request:**  
  Returns a list of all workflow instance objects stored in the database.

#### **Endpoint:** `/api/v1/workflow-instance/{id}`
- **GET request:**  
  Returns the workflow instance object for the specified id.
- **PUT request:**  
  Updates the workflow instance object for the specified id with the provided fields; returns the updated object.
- **DELETE request:**  
  Deletes the workflow instance object for the specified id and returns a success message with status code 200.

---

Checklist Item Instance Endpoints

#### **Endpoint:** `/api/v1/checklist-item-instance`
- **POST request:**  
  Creates a new checklist item instance from a checklist item template for a given workflow instance; returns the newly created checklist item instance.
- **GET request:**  
  Returns a list of all checklist item instance objects stored in the database.

#### **Endpoint:** `/api/v1/checklist-item-instance/{id}`
- **GET request:**  
  Returns the checklist item instance object for the specified id.
- **PUT request:**  
  Updates the checklist item instance object for the specified id with the provided fields; returns the updated object.
- **DELETE request:**  
  Deletes the checklist item instance object for the specified id and returns a success message with status code 200.

---

### Document Upload Endpoints

#### **Endpoint:** `/api/v1/document-upload`
- **POST request:**  
  Creates a new document upload record associated with a checklist item instance, with the provided file metadata; returns the newly created document upload record.
- **GET request:**  
  Returns a list of all document upload objects stored in the database.

#### **Endpoint:** `/api/v1/document-upload/{id}`
- **GET request:**  
  Returns the document upload object for the specified id.
- **PUT request:**  
  Updates the document upload object for the specified id with the provided fields; returns the updated object.
- **DELETE request:**  
  Deletes the document upload object for the specified id and returns a success message with status code 200.

---

### Audit Log Endpoints

> **Note:** Audit logs are append-only. You may create (POST) new audit log entries and retrieve (GET) existing entries; updating (PUT) or deleting (DELETE) audit log records is not permitted.

#### **Endpoint:** `/api/v1/audit-log`
- **POST request:**  
  Creates a new audit log record with the provided fields; returns the newly created audit log record.
- **GET request:**  
  Returns a list of all audit log records stored in the database.

#### **Endpoint:** `/api/v1/audit-log/{id}`
- **GET request:**  
  Returns the audit log record for the specified id.


