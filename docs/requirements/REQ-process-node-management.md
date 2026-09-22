# Process Node Management

## Summary

Users can create, edit, and remove **process nodes**, where each node
represents one individual marketing activity (e.g. "Identify Course",
"Reach Audience", "Capture Leads"). Each node carries its own
documentation, which can be edited independently of every other node.

This requirement defines a standalone node entity, distinct from — and
coexisting with — the process-chain concept in
`REQ-marketing-process-node-management` (which documents nodes as a linked
sequence with automation, tooling, and `next_nodes` fields). This
requirement covers only the node itself: its identity and its own
documentation. **Connecting nodes into an ordered sequence or flow is out
of scope here** and is left to a separate, future requirement.

## Node definition

Each node supports:

- `id`, `name` — the node's identity.
- **Level 1 — what must happen.** A description of the activity itself,
  independent of who performs it or which tools are used.
- **Level 2 — how it is currently done.** Implementation notes: manual
  steps, tools, or other practical detail of carrying the activity out
  today.

Level 1 and Level 2 are edited and stored independently of one another —
editing one must never require touching or overwriting the other.

## Acceptance criteria

- A user can create a new process node with a name and, optionally,
  Level 1 and/or Level 2 content.
- A user can edit an existing node's name.
- A user can edit a node's Level 1 content without altering its Level 2
  content, and vice versa.
- A user can remove a node.
- Nodes are managed independently of one another — creating, editing, or
  removing one node does not require touching any other node.

## Out of scope

- Ordering, chaining, or branching nodes into a process flow
  (`next_nodes`) — tracked separately.
- Automation status/trigger/conditions/actions and tool/URL references —
  these belong to the existing chain-based node model in
  `REQ-marketing-process-node-management`, not this requirement.
