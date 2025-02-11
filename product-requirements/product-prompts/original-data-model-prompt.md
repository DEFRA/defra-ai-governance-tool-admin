```
I'm writing a governance checklist app.  I'm trying to work out the data model for it and I'd like you to help me work it out and ask questions so we get it right.  Here are the features of the app:

- This too allows projects to track their governance process through a unified checklist, composed of multiple governance workflows, each with their own checklist items.
- There can be multiple governance workflows of checklist items, such as a main governance flow, then sub-flows depending on what part of the business you're delivering for.
- the checklist items are general, but should be able to support these types - labels are examples: 'approval' (where the   checklist item requires somebody to upload a document as evidence of the approval and state the person, date/time of the approval when checking off), 'document' (where the user must upload the latest version of the document before checking off).  I guess 'approval' items also have 'document' item aspects to it, so it's likely they are just two additional types of a normal checklist item (without any states), 'event' could be another type.
- a check list item should have states (example labels): 'incomplete', 'complete', 'not required'. 
- if there is a state change, then we need to have an audio log of when the state was changed and by whom
- checklist items can have dependencies, whereby, there can be zero or more dependencies against other checklist items.  Therefore, an item with incomplete dependency checklist items cannot be marked complete until all the dependencies are marked complete first.
- if something is marked 'not required' then we need to note who marked it as such with an explanation.
- for the different workflows, a checklist item may have dependencies that cross a workflow.  These dependencies force a sort of timeline sequence of checklist items.
- some workflow processes could happen in parallel.  If there are no dependencies impeding a checklist item, then it should be actionable.
- If a checklist item is an event, then we should track when it was scheduled and completed
- The different workflows need to have some metadata about how they apply to different projects so that we can ask the user to assess which workflows are needed for their project given their delivery circumstances.
- Because workflows may, or may not, be available to a given project, we need to be able to setup the dependencies structure in the backend, but when it comes to tracking the checklist items for a project, we need to be able to ignore any dependencies for workflow checklist items that may not exist for that project.  There needs to be a concept of 'active' dependencies vs 'inactive' ones.
- I want the overall workflow template, along with all the sub workflows and checklist items to be under a version control system, so that new project can follow an updated version of the model, without affecting legacy projects.
- The whole system needs to have granular audit logging for audit purposes.

Think about the above requirements and work out a simplified data model suitable for use with mongoDB.  Also produce a Mermaid diagram so it is understandable in a relational format.  Feel free to change labels to be more clear if you need to if they are more logical, but make sure the end model covers all the requirements before returning a solution
```