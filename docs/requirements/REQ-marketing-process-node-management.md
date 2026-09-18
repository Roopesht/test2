# Marketing Process Node Management

## Summary

Users can document a marketing process as a chain of connected **nodes**,
where each node represents one meaningful step in the process (e.g.
"Identify Course" → "Define Campaign" → "Reach Audience" → "Capture Leads"
→ "Nurture Leads" → "Measure Interest" → "Offer Course" → "Follow Up").
Nodes can be added, removed, reordered, or split as the process evolves,
without needing to redesign the process from scratch.

This requirement covers the data model and editing capability described in
the project blueprint (`docs/project/blueprint.cleaned.html`, sections 2–7):
process structure, node definition, Level 1 / Level 2 documentation, and
tool/URL references. Automated *execution* of automation (actually running
triggers/actions) and future code generation (blueprint section 9) are out
of scope for this requirement — only structured *documentation* of them is
required here.

## Process structure

- A marketing process is a sequence of connected nodes: `Node → Node → Node`.
- Each node links to one or more `next_nodes`, allowing branching if needed.
- Nodes may be added, removed, reordered, or split at any time as the
  process evolves. Editing the process must not require re-authoring
  unaffected nodes.

## Node definition

Each node supports the following fields (only relevant fields need to be
populated):

- `id`, `name`, `description`
- `input`, `output`
- `implementation`: `description`, `manual`, `automated`
- `tools`: a list of `{ name, url }` references
- `automation`: `description`, `trigger`, `conditions`, `actions`
- `next_nodes`: ids of the node(s) that follow
- `notes`

## Level 1 vs Level 2 documentation

Each node separates two layers of documentation, which can change
independently of one another:

- **Level 1 — what must happen.** Independent of specific tools, software,
  people, APIs, or implementation details.
- **Level 2 — how the node is currently implemented.** Manual steps,
  automated steps, tools used, and relevant URLs.

Editing Level 2 for a node (e.g. swapping which tool is used) must not
require changing that node's Level 1 description, and vice versa.

## Automation documentation

Each node can document its automation state as one of: **none**,
**partially automated**, **fully automated**, or **planned**, along with:

- A trigger (what starts the automation)
- Conditions (what must be true for it to run)
- A numbered list of actions it performs

This is documentation of automation, not an automation execution engine.

## Tools and URL references

Any node may reference external tools, applications, documents, forms,
APIs, dashboards, or internal systems, each as a name + URL pair, so
readers can jump directly to where a step is currently performed.

## Acceptance criteria

- A user can create a new marketing process and add nodes to it in sequence.
- A user can edit a node's `next_nodes` to reorder, branch, or splice in a
  new node without recreating the rest of the process.
- A user can split one node into two connected nodes, preserving existing
  Level 1/Level 2 content on the original node.
- A user can independently edit a node's Level 1 and Level 2 content.
- A user can record a node's automation status (none / partial / full /
  planned) with trigger, conditions, and actions.
- A user can attach one or more named tool/URL references to a node.
