# Security Controls Table

A per-endpoint-pair breakdown of what an RFC or design doc actually says about authentication, authorization, sanitization, data protection, and logging — generated automatically from the document's own text by the `security-design-review-controls-table` skill.

See [`examples/profile-service-example.md`](examples/profile-service-example.md) for a worked example, generated from [`../../examples/profile-service-rfc.md`](../../examples/profile-service-rfc.md).

## What this table actually is

One row per connection in the system (`Web user → API`, `API → Database`, `Service A → Service B`), one column per control category. Every cell is either:

- **A direct extraction** — the RFC states a mechanism, and the cell names it (`mTLS`, `Session cookie`, `Ownership-based access`).
- **A quoted gap** — the RFC requires something but doesn't name how (`Encrypted; protocol not named ("MUST be transport-encrypted")`).
- **An honest "Not specified by the RFC"** — the document says nothing at all for that cell.

Nothing in the table is inferred, assumed, or filled in from "how these systems usually work." If it's not in the source document, it's not in the table.

## Why this matters for a security review

**It turns "read the whole RFC for security-relevant details" into a five-minute scan.** RFCs bury their security posture across an architecture section, an API reference, and a paragraph or two of prose near the end. This table pulls all of that into one place, organized the way a reviewer actually thinks — by connection, not by document section.

**Every gap is a finding, not a guess.** A cell reading "Not specified by the RFC" isn't a placeholder — it's the table telling you exactly what to ask the RFC's author before sign-off. Because nothing is invented, you can trust that a filled-in cell reflects a real decision the design made, and an empty one reflects a real decision it didn't.

**It's exhaustive by construction.** The extraction step walks every actor, every API, every datastore, and every edge between them — so a connection can't quietly fall through the cracks the way it might in a prose-only review, especially on a system with more than three or four moving parts.

**It's a ready-made review artifact.** Paste the table into the RFC itself, a design-review doc, or a ticket, and it becomes the concrete checklist a review meeting works through — "here are the six edges, here's what's covered, here are the four gaps to resolve" — instead of a free-form discussion that may or may not touch every hop.

**It scales the same way across every RFC you review.** Same columns, same conventions, every time — which means you can compare two RFCs' security posture at a glance, or spot that a team's designs *consistently* leave logging unspecified, a pattern that's much harder to notice reading prose one document at a time.

**It pairs with the dataflow diagram.** The sibling `security-design-review-dataflow-diagram` skill extracts the same endpoints and edges into a visual layout — useful in the same review for showing *where* the gaps in this table actually sit in the system's shape, and (in its current version) marking which edges carry sensitive data or an unencrypted stated protocol.

## What it deliberately won't do

It won't tell you a control is *good enough* — only whether the RFC names one. A cell reading "TLS" doesn't mean the review is over; it means that's the fact on record for you to evaluate. And it won't fill a real gap with a plausible guess, even when you're confident you know the answer — a table with speculative content stops being trustworthy the moment one cell in it might be a guess instead of a fact, since a reader can no longer tell which cells are which.
