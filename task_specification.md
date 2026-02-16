Purpose
This document defines the standard structure for coding tasks.
It is intended to be reusable for future tasks and compatible with AI-assisted task generation.

1. TASK OVERVIEW

Provide a concise description of what the task is intended to accomplish and why it exists.

This section should answer:
1) What is being built, modified, or investigated?
2) What larger system, workflow, or business goal does this support?

2. REPOSITORIES AND BRANCHING

Specify where the work should occur and how it should be managed.

Include:
1) Repository name(s)
2) Branching requirements (e.g., feature branch required). Base branch should default to "your latest branch" unless explicitly specified.
3) Any restrictions (e.g., do not commit directly to "main")

3. TASK SCOPE

Define exactly what is included in this task and what must be produced.

Specify:
1) Data sources, systems, or services involved
2) Work to be completed (requirements and constraints)

Avoid describing how the work should be implemented unless constraints are required.
Do not include in/scope out of scope lists.

4. INPUTS / ACCESS PROVIDED

List any access, credentials, or materials already provided to complete the task.

Examples:
1) UI access
2) API tokens
3) Repositories or shared folders
4) Reference documents or examples

5. TECHNICAL REQUIREMENTS

Describe technical and procedural expectations.

Include as applicable:
1) Programming language(s)
2) Libraries or frameworks
3) Coding standards or methodology to follow
4) Modularity and documentation expectations
5) Local vs remote execution assumptions
6) Time tracking requirements (e.g., Asana actual time entry on the day work is performed)

6. ENVIRONMENT CONFIGURATION

Specify environment and configuration requirements.

Include:
1) Required environment variables
2) ".env" and ".env.example" expectations
3) Git ignore requirements

7. ADMIN

Use this section for administrative requirements tied to the task.

Include as applicable:
1) Time tracking requirements (e.g., Asana actual time entry on the day work is performed)
2) Reporting cadence or status update expectations
3) Communication channels or escalation contacts
4) Acknowledgement requirements (e.g., confirm receipt)
5) Estimation or schedule expectations (e.g., hours estimate, expected completion date)

8. DEFAULTS AND OPERATING CONTEXT

Capture default assumptions that should be applied when a task does not explicitly specify them.
This section prevents reliance on prior task memory and keeps AI-generated tasks consistent.
Do not include this section in the task output. Instead, fold these defaults into the relevant sections when generating a task.

Defaults (apply unless explicitly overridden by the task):
1) Repositories and branching:
- Primary repo: "onewaypath.com-public"
- Base branch: "your latest branch"
- Create a new feature branch from the base
- Do not commit directly to "main"
2) If backend/API/database work is required, also include:
- Repository: "serverJS"
- Base branch: "main"
- Create a new feature branch from "main"
- Do not commit directly to "main"
3) Coding policy: follow the latest OWP Coding Policy in "methodology/owpc-coding-policy-v*.md" and repository coding conventions/formatting.
4) Time tracking: log time in Asana on the day work is performed, in the Actual Time field.
5) Status updates: none unless explicitly specified (e.g., a Slack channel/cadence).
6) Admin expectations: confirm receipt, provide hours estimate, provide expected completion date, and log time on the day of work.
7) Deliverables: include an "Expected Outputs / Deliverables" list with concrete bullets.

If a task explicitly overrides any default, the task's explicit instruction wins.

9. NOTES FOR AI-GENERATED TASKS

When using AI to generate a task based on this specification:
1) Populate each section explicitly.
2) Output plain text only. Do not use Markdown, bold, or code blocks.
3) Use all-caps section headings (e.g., "TASK OVERVIEW").
4) Do not introduce additional sections or Agile terminology beyond this specification.
5) Keep language neutral, professional, and execution-focused.
4) Assume the reader is a technical contributor with repository access.
5) Do not include boilerplate like "Brief run instructions if new steps are required."
6) For Figma-only tasks, sections 2 (Repositories and Branching), 4 (Inputs / Access Provided), and 6 (Environment Configuration) are not required.
7) Do not include the "TECHNICAL TASK SPECIFICATION" header at the top of task files.
8) Do not include an "Objective" subheading under Task Overview.
9) Do not include this "NOTES FOR AI-GENERATED TASKS" section in the task output.
