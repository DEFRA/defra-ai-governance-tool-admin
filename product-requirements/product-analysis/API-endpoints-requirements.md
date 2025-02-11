	# API Documentation
	
	This document provides a complete set of API endpoint requirements for each major resource in the data model. All endpoints follow RESTful conventions, use JSON payloads, and (except for the Audit Log) support full CRUD (Create, Read, Update, Delete) operations. For each resource, example request objects are provided to illustrate the expected payloads.
	
	## 1. Governance Template Endpoints
	
	### Endpoint: `/api/v1/governance-template`
	- **POST request:**  
	  Creates a new **GovernanceTemplate** using the fields provided in the request body. Returns the newly created object from the database.
	  
	- **GET request:**  
	  Returns a list of all **GovernanceTemplate** objects stored in the database.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "version": "1.0.0",
	  "name": "Default Governance Process",
	  "description": "The baseline process for governance and compliance."
	}
	```
	
	### Endpoint: `/api/v1/governance-template/{id}`
	- **GET request:**  
	  Retrieves the **GovernanceTemplate** object with the specified `{id}`.
	  
	- **PUT request:**  
	  Updates the **GovernanceTemplate** object with the given `{id}` using the provided fields; returns the updated object.
	  
	- **DELETE request:**  
	  Deletes the **GovernanceTemplate** object for the given `{id}` and returns a success message with status code 200.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "version": "1.0.1",
	  "name": "Updated Governance Process",
	  "description": "An updated description reflecting process improvements."
	}
	```
	
	## 2. Workflow Template Endpoints
	
	### Endpoint: `/api/v1/workflow-template`
	- **POST request:**  
	  Creates a new **WorkflowTemplate**. The request body must include a reference to the parent GovernanceTemplate (`governanceTemplateId`), a name, description, and a `metadata` JSON object.
	  
	- **GET request:**  
	  Returns a list of all **WorkflowTemplate** objects.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "governanceTemplateId": "60d21baee3d5d533d9fc1e4b",
	  "name": "Approval Workflow",
	  "description": "A workflow to handle approvals.",
	  "metadata": {
	    "department": "Finance",
	    "priority": "High"
	  }
	}
	```
	
	### Endpoint: `/api/v1/workflow-template/{id}`
	- **GET request:**  
	  Retrieves the **WorkflowTemplate** object with the specified `{id}`.
	  
	- **PUT request:**  
	  Updates the **WorkflowTemplate** object with the given `{id}` using provided fields.
	  
	- **DELETE request:**  
	  Deletes the **WorkflowTemplate** object for the given `{id}` and returns a success message.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "name": "Revised Approval Workflow",
	  "description": "Updated workflow for processing approvals.",
	  "metadata": {
	    "department": "Finance",
	    "priority": "Medium"
	  }
	}
	```
	
	## 3. Checklist Item Template Endpoints
	
	### Endpoint: `/api/v1/checklist-item-template`
	- **POST request:**  
	  Creates a new **ChecklistItemTemplate**. The required fields include `workflowTemplateId`, `itemKey`, `name`, `description`, `type`, `config`, and an array for `dependencies`.
	  
	- **GET request:**  
	  Returns a list of all **ChecklistItemTemplate** objects.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "workflowTemplateId": "60d21bbfe3d5d533d9fc1e4c",
	  "itemKey": "approvalDoc",
	  "name": "Upload Approval Document",
	  "description": "Checklist item for uploading an approval document.",
	  "type": "document",
	  "config": {
	    "required": true
	  },
	  "dependencies": []
	}
	```
	
	### Endpoint: `/api/v1/checklist-item-template/{id}`
	- **GET request:**  
	  Retrieves the **ChecklistItemTemplate** with the specified `{id}`.
	  
	- **PUT request:**  
	  Updates the **ChecklistItemTemplate** object with the given `{id}`.
	  
	- **DELETE request:**  
	  Deletes the **ChecklistItemTemplate** object with the specified `{id}` and returns a success message.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "name": "Upload Signed Approval Document",
	  "description": "Updated checklist item requiring a signed document.",
	  "config": {
	    "required": true,
	    "fileFormat": "PDF"
	  }
	}
	```
	
	## 4. Project Endpoints
	
	### Endpoint: `/api/v1/project`
	- **POST request:**  
	  Creates a new **Project**. The body should include the project's `name`, `description`, a reference to the chosen `governanceTemplateId`, and an array of `selectedWorkflowTemplateIds` indicating which workflows to apply.
	  
	- **GET request:**  
	  Returns a list of all **Project** objects.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "name": "Project Alpha",
	  "description": "A new project following the default governance process.",
	  "governanceTemplateId": "60d21baee3d5d533d9fc1e4b",
	  "selectedWorkflowTemplateIds": [
	    "60d21bbfe3d5d533d9fc1e4c",
	    "60d21bcfe3d5d533d9fc1e4d"
	  ]
	}
	```
	
	### Endpoint: `/api/v1/project/{id}`
	- **GET request:**  
	  Retrieves the **Project** object for the specified `{id}`.
	  
	- **PUT request:**  
	  Updates the **Project** object with the provided fields.
	  
	- **DELETE request:**  
	  Deletes the **Project** object for the given `{id}` and returns a success message.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "name": "Project Alpha - Phase 2",
	  "description": "Updated project description after initial review."
	}
	```
	
	## 5. Workflow Instance Endpoints
	
	### Endpoint: `/api/v1/workflow-instance`
	- **POST request:**  
	  Creates a new **WorkflowInstance**. Required fields include the parent `projectId`, the source `workflowTemplateId`, and a `version` (taken from the GovernanceTemplate/WorkflowTemplate snapshot).
	  
	- **GET request:**  
	  Returns a list of all **WorkflowInstance** objects.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "projectId": "60d21c1ae3d5d533d9fc1e4e",
	  "workflowTemplateId": "60d21bbfe3d5d533d9fc1e4c",
	  "version": "1.0.0"
	}
	```
	
	### Endpoint: `/api/v1/workflow-instance/{id}`
	- **GET request:**  
	  Retrieves the **WorkflowInstance** with the specified `{id}`.
	  
	- **PUT request:**  
	  Updates the **WorkflowInstance** object with the provided fields.
	  
	- **DELETE request:**  
	  Deletes the **WorkflowInstance** object with the specified `{id}` and returns a success message.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "version": "1.0.1"
	}
	```
	
	## 6. Checklist Item Instance Endpoints
	
	### Endpoint: `/api/v1/checklist-item-instance`
	- **POST request:**  
	  Creates a new **ChecklistItemInstance**. The body should include the `workflowInstanceId`, `checklistItemTemplateId`, an initial `state` (such as `"incomplete"`), an optional `typeData` JSON object, and an array for `dependencies` (which may be empty if there are no dependencies).
	  
	- **GET request:**  
	  Returns a list of all **ChecklistItemInstance** objects.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "workflowInstanceId": "60d21c5ae3d5d533d9fc1e4f",
	  "checklistItemTemplateId": "60d21bbfe3d5d533d9fc1e4c",
	  "state": "incomplete",
	  "typeData": {},
	  "dependencies": []
	}
	```
	
	### Endpoint: `/api/v1/checklist-item-instance/{id}`
	- **GET request:**  
	  Retrieves the **ChecklistItemInstance** for the given `{id}`.
	  
	- **PUT request:**  
	  Updates the **ChecklistItemInstance** object with the specified `{id}`.
	  
	- **DELETE request:**  
	  Deletes the **ChecklistItemInstance** object with the given `{id}` and returns a success message.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "state": "complete",
	  "typeData": {
	    "approvedBy": "60d21d2be3d5d533d9fc1e50",
	    "approvedAt": "2025-02-11T10:15:00Z"
	  }
	}
	```
	
	## 7. Document Upload Endpoints
	
	### Endpoint: `/api/v1/document-upload`
	- **POST request:**  
	  Creates a new **DocumentUpload** record. The request body must include a reference to the associated `checklistItemInstanceId`, the `uploadType` (e.g., `"approvalEvidence"`), `fileName`, `url`, `mimeType`, `uploadedAt` (timestamp), `uploadedBy` (user reference), and optional `metadata`.
	  
	- **GET request:**  
	  Returns a list of all **DocumentUpload** objects.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "checklistItemInstanceId": "60d21c5ae3d5d533d9fc1e4f",
	  "uploadType": "approvalEvidence",
	  "fileName": "approval.pdf",
	  "url": "https://s3.amazonaws.com/bucket/approval.pdf",
	  "mimeType": "application/pdf",
	  "uploadedAt": "2025-02-11T10:00:00Z",
	  "uploadedBy": "60d21d2be3d5d533d9fc1e50",
	  "metadata": {
	    "notes": "Signed and approved document"
	  }
	}
	```
	
	### Endpoint: `/api/v1/document-upload/{id}`
	- **GET request:**  
	  Retrieves the **DocumentUpload** object with the specified `{id}`.
	  
	- **PUT request:**  
	  Updates the **DocumentUpload** record with the provided fields.
	  
	- **DELETE request:**  
	  Deletes the **DocumentUpload** object with the given `{id}` and returns a success message.
	
	#### Example PUT Request Payload:
	
	```json
	{
	  "fileName": "updated_approval.pdf",
	  "metadata": {
	    "notes": "Replaced with updated version"
	  }
	}
	```
	
	## 8. Audit Log Endpoints
	
	*Audit logs are immutable. Endpoints only allow creation (POST) and retrieval (GET); updates and deletions are not permitted.*
	
	### Endpoint: `/api/v1/audit-log`
	- **POST request:**  
	  Creates a new **AuditLog** entry. The request body must include the `eventType`, `objectType`, `objectId`, `changedAt` (timestamp), `changedBy` (user reference), and a `details` JSON object with additional information.
	  
	- **GET request:**  
	  Returns a list of all **AuditLog** entries.
	
	#### Example POST Request Payload:
	
	```json
	{
	  "eventType": "stateChange",
	  "objectType": "ChecklistItemInstance",
	  "objectId": "60d21c5ae3d5d533d9fc1e4f",
	  "changedAt": "2025-02-11T10:05:00Z",
	  "changedBy": "60d21d2be3d5d533d9fc1e50",
	  "details": {
	    "previousState": "incomplete",
	    "newState": "complete",
	    "note": "Checklist item approved and completed."
	  }
	}
	```
	
	### Endpoint: `/api/v1/audit-log/{id}`
	- **GET request:**  
	  Retrieves the **AuditLog** entry for the specified `{id}`.
	
	## Summary
	
	### CRUD Operations
	All endpoints (except for Audit Log, which is write-once/read-only) support standard operations:
	- **POST:** Create a new resource
	- **GET:** Retrieve one or more resources
	- **PUT:** Update an existing resource by its ID
	- **DELETE:** Remove a resource by its ID
	
	### Payload Examples
	Each resource has an example JSON object provided for POST (and PUT where applicable), ensuring clarity on required fields and data formats.
	
	This structured API design supports creating and managing version-controlled governance templates, workflows, checklist items, project instances, file uploads, and audit logs, ensuring a robust and traceable governance management application.