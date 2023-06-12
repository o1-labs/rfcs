# O(1) Labs Request for Comments (RFC) repository

RFCs facilitate rapid review and approval from O(1) Labs architects. RFCs are an effective way to manage planned changes and provide a contextual historical artifact. An open and honest dialog in a Request for Comments (RFC) workflow is visible and provides historical context.

## Motivation

The RFC workflow ensures that all important stakeholders participate in the RFC review process. 

RFCs provide context on why the decisions were made and describe the technical climate at the time the idea was proposed. RFCs use existing git collaboration tools to create, edit, maintain, and build consensus on code ideas that impact all of engineering. 

## RFCs at O(1) Labs

Each RFC captures technical details for a specific change within a limited system context at a given moment in time. 

- The rapid review and approval cycle prevents lengthy opportunities for public input. 
- In contrast to timeless product specifications, RFCs are more ephemeral and benefit from the narrower focus of a delta.
- In many software projects, a typical RFC invites public discourse. In contrast, an RFC at O(1) Labs is an internal design document to ensure that all architects participate in design decisions. 
- RFCs happen downstream from product requirements doc (PRDs).

## Workflow

1. Product teams define product requirements in the product requirements doc (PRD).
2. An internal O(1) Labs team member creates a pull request using the [0000-template.md](0001-template.md).
3. The RFC covers the changes in enough detail to inform the decision.
4. Discussions occur in the RFC pull request within a rapid defined time period.
5. If architecture changes proposed in the initial RFC are significantly changed during the discussion, the submitter updates the RFC in a separate pull request.
6. All architects must approve the RFC pull request within two business days.
7. RFC approval by all architects is required before implementation decisions.

## Qualities of effective RFCs

- Prefer timely over perfect
- Enable collaborative code comments, questions, and responses that capture the relevant RFC discussion
- Provide context for changes that impact different product areas

## Who?

Who can submit an RFC? Anyone.
Who should read RFCs? Everyone.
Who must approve RFCs? All architects.



