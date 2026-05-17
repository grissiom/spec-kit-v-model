---
description: Generate a comprehensive software architecture design from requirements alone, synthesizing IEEE 1016 design entities within IEEE 42010 views. Replaces system-design + architecture-design (Path B).
handoffs:
  - label: Generate Integration Tests
    agent: speckit.v-model.integration-test
    prompt: Generate the integration test plan for this software architecture design
    send: true
  - label: Back to Requirements
    agent: speckit.v-model.requirements
    prompt: Review or update requirements
scripts:
  sh: scripts/bash/setup-v-model.sh --json --require-reqs
  ps: scripts/powershell/setup-v-model.ps1 -Json -RequireReqs
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Generate a comprehensive software architecture design derived from `requirements.md` alone, synthesizing **IEEE 1016:2009** (Software Design Description — design entity model) within **IEEE 42010:2011** (Architecture Description — viewpoint framework). This single artifact replaces the two-step Path A chain (`system-design` → `architecture-design`).

The output must support:
- `REQ-NNN` → `ARCH-NNN` traceability (no intermediate `SYS-NNN` layer)
- four architecture views synthesizing IEEE 1016 design entities within IEEE 42010 viewpoints:
  - **Logical View** ← IEEE 1016 Decomposition (§5.1) + Dependency (§5.2) within IEEE 42010 Logical View table
  - **Process View** ← IEEE 42010 / Kruchten 4+1 (no IEEE 1016 counterpart — synthesis differentiator)
  - **Interface View** ← IEEE 1016 Interface Identification (§5.3) + IEEE 42010 protocol bindings, with external/internal distinction
  - **Data Flow View** ← IEEE 1016 Data Design (§5.4) + IEEE 42010 pipeline semantics
- ASPICE SWE.2 process guidance (only when `domain: iso_26262` in `v-model-config.yml`)
- ISO/IEC 42030:2019 architecture evaluation + ISO/IEC 25010:2023 quality attribute cross-check
- explicit handling of derived requirements, derived modules, and cross-cutting elements
- **architecture governance**: Phase -1 architecture framing (intent, core tensions, stable boundaries, change axes, invariants, anti-patterns), per-view gap tracking, and architecture gate verification (6 hard ERROR gates)

## Execution Steps

### 1. Setup

Run `{SCRIPT}` from the repository root and parse the JSON output.

The script returns JSON with these keys:
- `VMODEL_DIR`: Path to `specs/{feature}/v-model/` directory
- `FEATURE_DIR`: Path to `specs/{feature}/` directory
- `BRANCH`: Current branch name
- `REQUIREMENTS`: Path to `requirements.md` (MUST exist — script uses `--require-reqs`)
- `AVAILABLE_DOCS`: Array of documents that currently exist

For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

### 2. Load Context

1. **Load the template**: Read `templates/software-architecture-design-template.md` from the extension directory to understand the required output structure.

2. **Load requirements**: Read `requirements.md` from the `REQUIREMENTS` path. This is the **sole source of truth** for what the system must do.
   - Extract all `REQ-NNN` identifiers (all categories: functional, non-functional, interface, constraint)
   - Note the total count — every REQ must appear as a parent in at least one ARCH element

3. **Load spec.md** (if `AVAILABLE_DOCS` contains `"spec.md"`): Read for supplementary context (user stories, acceptance scenarios, edge cases). This provides architectural insight but does NOT override requirements.

4. **Load domain config**: Read `v-model-config.yml` if it exists at the repository root. Note the `domain` value.

5. **Check Path A coexistence**: If `AVAILABLE_DOCS` contains `"architecture-design.md"`, note that both Path A and Path B artifacts will coexist. Emit a warning but proceed. `integration-test` already prefers `software-architecture-design.md`.

### 3. Domain Configuration

Load `v-model-config.yml` if it exists at the repository root.

**If `domain` is set** (e.g., `iso_26262`, `do_178c`, `iec_62304`):
1. Read the command overlay: `commands/overlays/{domain}/software-architecture-design.md`
   - If it exists: note the domain-specific design sections (e.g., SWE.2 BP1–BP9 for `iso_26262`, DAL allocation for `do_178c`, safety class allocation for `iec_62304`)
   - If it does not exist: this domain does not extend this command — proceed with base only
2. Where the base command references "domain overlay sections", use the overlay's guidance in preference

**If `domain` is empty or absent:**
- Produce clean, industry-neutral output
- Do NOT include any domain-specific regulatory references
- SWE.2 sections are NOT generated

### 4. Architecture Framing (Phase -1)

Before decomposing requirements into architecture elements, establish the architecture judgment this pass must preserve. This framing constrains all four views and the architecture evaluation. Do not create an additional file — apply this framing as the decision filter for every later step.

**Identify the following** and use them to constrain all subsequent view generation:

1. **Architecture Intent**: What this architecture pass is trying to make stable or explicit. What boundaries, tradeoffs, and constraints must future implementation preserve?
2. **Core Tensions**: The main design forces in conflict (e.g., modularity vs performance, simplicity vs extensibility), and the current tradeoff direction for each. For each tension, identify which views are affected and the architectural consequence.
3. **Stable Boundaries**: Responsibilities or authority lines that should remain stable across iterations. For each boundary, identify an explicit **forbidden crossing** — a connection or dependency that must not be introduced.
4. **Change Axes**: Business, workflow, operational, or integration areas expected to vary and therefore needing isolation. For each axis, identify how it is isolated (which ARCH element or pattern absorbs the change).
5. **Responsibility Collision Risks**: Responsibilities that agents or teams are likely to merge incorrectly. Explicitly separate them.
6. **Invariants**: Architecture rules later iterations must not violate. Each invariant must be grounded in a requirement from `requirements.md`, a scenario from `spec.md`, or an explicit architectural constraint.
7. **Non-goals / Anti-patterns**: Concerns this architecture pass does not solve, plus designs that would drift from the intent (e.g., "black-box ARCH elements without interface contracts", "implementation details leaking into architecture views").
8. **Implementation Details to Exclude**: Concrete classes, files, API endpoints, DTO fields, database tables, framework selections, infrastructure manifests — anything that belongs in module design or implementation, not architecture.

**Quality rules for framing**:
- Every non-placeholder conclusion must be grounded in a requirement, scenario, or stated architectural constraint.
- If evidence is insufficient to support a claim, record a specific gap instead of inventing facts.
- Use stable names consistently across all four views and the architecture evaluation.
- Remove generic statements ("scalable", "secure", "modular") unless they name owner, affected view, scope, and architecture consequence.

**Output**: Record the framing decisions in the `## Architecture Framing` section of the generated artifact. The framing section appears after `## Overview` and before `## ID Schema` in the output template.

### 5. Lifecycle Rules (When Evolving Existing Artifacts)

When an existing `software-architecture-design.md` is loaded, apply these rules before generating new content:

1. **Never delete an ID** — mark as `[DEPRECATED]`
2. **Deprecation types:**
   - `[DEPRECATED — Superseded by ARCH-NNN]`: Replaced by a new element
   - `[DEPRECATED — Withdrawn: <reason>]`: Removed entirely with justification
3. **Suspect detection from parent REQ:** If a parent REQ (in `requirements.md`) is deprecated or modified, mark each ARCH that traces to it as `[SUSPECT — Parent REQ-NNN {deprecated|modified}]`.
4. **Suspect resolution:** For each suspect ARCH:
   - **Re-parent** to the superseding REQ (if capability continues under a new requirement)
   - **Deprecate** (if the requirement is withdrawn — cascade to downstream MOD, ITP)
   - **Confirm active** (if still valid despite the parent change — remove the SUSPECT tag)
5. **Modified elements:** Update content in-place, preserve the original ARCH ID. Downstream artifacts (MOD, ITP) tracing to this ARCH become suspect.

If no existing `software-architecture-design.md` is found, skip this step — all elements are new.

### 6. Decompose Requirements into Architecture Elements

Follow the **strict translator constraint**: You are decomposing requirements into architecture elements. You must NOT invent capabilities not present in `requirements.md`.

#### 6.1 Decomposition by Requirement Category

- **Functional requirements** (`REQ-NNN`): Each maps to one or more dedicated architecture elements. Group related capabilities into cohesive components.
- **Non-functional requirements** (`REQ-NF-NNN`): Map to cross-cutting elements (e.g., logging, caching, error handling) or as additional parents on existing elements that implement the quality attribute.
- **Interface requirements** (`REQ-IF-NNN`): Map to elements that own the interface contract. These will have detailed entries in the Interface View.
- **Constraint requirements** (`REQ-CN-NNN`): Map to elements that enforce the constraint (e.g., rate limiter, schema validator) or annotate existing elements with constraint notes.

#### 6.2 IEEE 1016 Design Entity Attributes

Every architecture element must document these IEEE 1016 attributes:
- **Identifier**: `ARCH-NNN` (sequential, starting at ARCH-001)
- **Purpose**: Short name (the "Name" column in the Logical View)
- **Function**: What it does (the "Description (Purpose)" column)
- **Subordinates**: None at this level (dependencies are to REQ, not to other ARCH — Path B does not have a hierarchical SYS middle layer)
- **Dependencies**: Parent `REQ-NNN` identifiers from `requirements.md`

#### 6.3 Many-to-Many Mapping

- A single REQ may be a parent of multiple ARCH elements (many-to-many)
- An ARCH element may have multiple parent REQs (many-to-many)
- The Logical View's "Parent Requirements (Dependencies)" column is the single source of truth for this mapping

#### 6.4 Element Types

Assign each ARCH element a type:
- **Component**: Primary functional unit
- **Service**: Background or infrastructure service
- **Library**: Shared utility code
- **Utility**: Stateless helper
- **Adapter**: Protocol or format bridge

#### 6.5 Cross-Cutting Tag

For infrastructure/utility elements not traceable to a specific REQ (e.g., Logger, Thread Pool, Config Manager):
- Set the "Parent Requirements (Dependencies)" column to `[CROSS-CUTTING] — <rationale>`
- Cross-cutting elements MUST still have interface contracts in the Interface View
- Cross-cutting elements MUST still have at least one parent `REQ-NNN` (per REQ-022 from `requirements.md`)
- Do NOT abuse this tag — if a capability can be tied to a non-functional or constraint REQ, use that REQ instead

#### 6.6 Derived Elements

When a capability is architecturally necessary but has no corresponding `REQ-NNN`, flag it as `[DERIVED REQUIREMENT: description of the capability and why it is architecturally necessary]`
- The human must resolve each derived item before proceeding to integration test generation

#### 6.7 Anti-Pattern Guard (per IEEE 42010)

- Reject "black box" descriptions: every `ARCH-NNN` MUST have an explicit interface contract (inputs, outputs, exceptions) in the Interface View
- If an element description is too vague to derive contracts, emit a warning and refine until concrete

### 7. Build the Four Synthesized Architecture Views

#### 7.1 Logical View (IEEE 1016 Decomposition §5.1 + Dependency §5.2 within IEEE 42010)

The primary view. Fill the Logical View table with all ARCH elements:

| ARCH ID | Name | Description (Purpose) | Parent Requirements (Dependencies) | Type |
|---------|------|----------------------|-----------------------------------|------|

**Rules**:
- Every `REQ-NNN` from `requirements.md` must appear in at least one row's "Parent Requirements" column
- Use comma-separated `REQ-NNN` list for many-to-many relationships
- Cross-cutting elements appear with `[CROSS-CUTTING] — rationale` in the Parent Requirements column
- No ARCH element may have an empty Parent Requirements field (unless `[CROSS-CUTTING]`)
- The Description column serves as the IEEE 1016 "purpose" and "function" attributes

**Gap tracking**: After the Logical View table, populate the Logical View Gaps table. Record any logical concern from requirements that cannot be mapped to an ARCH element or where uncertainty prevents complete decomposition. Every gap must name the affected element/capability and why it matters.

#### 7.2 Process View (IEEE 42010 / Kruchten 4+1 — no IEEE 1016 counterpart)

Document runtime interactions using Mermaid sequence diagrams:

1. For each critical interaction path, generate a `sequenceDiagram` with ARCH-NNN as participants
2. Document concurrency model (pipeline, event loop, actor model, etc.)
3. Show synchronization points, decision branches, and execution order constraints
4. Include domain branching logic (e.g., SWE.2 generation gated by domain)

**Rules**:
- Use Mermaid `sequenceDiagram` syntax — diagrams MUST be syntactically valid
- Reference `ARCH-NNN` IDs as participants
- Model interaction flows from requirements analysis (not from a system design Dependency View — there is none in Path B)
- This view directly feeds **Concurrency & Race Condition Testing** in integration test

**Gap tracking**: After the Process View, populate the Process View Gaps table. Record gaps in runtime behavior understanding. Every gap must name the affected runtime link or scenario and why it matters.

#### 7.3 Interface View (IEEE 1016 §5.3 Interface Identification + IEEE 42010)

Define interface contracts for **every** ARCH-NNN element, with explicit external/internal distinction:

**External Interfaces** (CLI args, file I/O, user-facing boundaries):
| ARCH ID | Interface Name | Direction | Protocol | Input | Output | Error Handling |

**Internal Interfaces** (element-to-element communication):
| Source ARCH | Target ARCH | Interface Name | Protocol | Data Format | Error Handling |

**Rules**:
- MUST distinguish between external and internal interfaces — separate tables (per IEEE 1016 §5.3)
- External interfaces focus on protocol compliance, input validation, and error responses
- Internal interfaces focus on contract adherence, data format correctness, and failure propagation
- No "black box" elements — every ARCH module MUST have contract entries
- Cross-cutting elements MUST also have contracts defined
- This view directly feeds **Interface Contract Testing** and **Interface Fault Injection** in integration test

**Gap tracking**: After the Interface View, populate the Interface View Gaps table. Record gaps in interface contracts. Every gap must name the affected interface/contract and why it matters.

#### 7.4 Data Flow View (IEEE 1016 §5.4 Data Design + IEEE 42010)

Trace data through the architecture pipeline and document data structures:

| Stage | Module | Input Format | Transformation | Output Format |

**Data Design supplement** (per IEEE 1016 §5.4):
| Data Entity | Owning ARCH | Storage | Protection | Lifecycle |

**Rules**:
- Show intermediate data formats at each pipeline stage
- Document data protection measures (at rest, in transit) where applicable
- Each flow traces input → transformation → output with intermediate formats
- Data flows directly drive **Data Flow Testing** in integration test

**Gap tracking**: After the Data Flow View, populate the Data Flow View Gaps table. Record gaps in data flow understanding. Every gap must name the affected data flow/entity and why it matters.

### 8. Architecture Evaluation (ISO/IEC 42030:2019 / ISO/IEC 25010:2023)

After generating the four views, perform a scenario-based fitness-for-purpose evaluation. This completes the IEEE 42010 "describe" → ISO 42030 "evaluate" cycle.

#### 8.1 Quality Attribute Cross-Check (ISO/IEC 25010:2023)

For each characteristic implied or explicitly stated in `requirements.md`, confirm at least one ARCH element or view decision covers it:

| Quality Characteristic | ISO/IEC 25010 Ref | Design Evidence Required |
|------------------------|-------------------|--------------------------|
| Functional Suitability (completeness, correctness) | §4.2.1 | Every `REQ-NNN` maps to at least one `ARCH-NNN` |
| Reliability (availability, fault tolerance, recoverability) | §4.2.2 | Interface View documents error handling and failure propagation |
| Performance Efficiency (time behaviour, resource utilisation) | §4.2.3 | Interface View or Data Flow View specifies performance constraints |
| Security (confidentiality, integrity, authenticity) | §4.2.5 | Data Flow View documents protection at rest/in transit for sensitive data |
| Maintainability (modularity, reusability, testability) | §4.2.7 | Logical View separates concerns; every interface is explicitly contracted |
| Safety (if applicable per domain) | §4.2.9 | ARCH elements link to domain-specific safety sections |

**Action on gaps**: If a characteristic implied by the requirements is NOT addressed by any ARCH element or view decision, flag it as `[QUALITY GAP: ISO 25010 §X.X — <characteristic> not explicitly addressed]`.

#### 8.2 Quality Attribute Justification (ISO/IEC 42030:2019)

For each significant architectural decision (one that affects more than one view or introduces a cross-cutting element), document its quality attribute rationale:

| Architecture Decision | Quality Characteristic (ISO 25010) | Trade-off Accepted |
|----------------------|------------------------------------|--------------------|
| e.g., Pipeline architecture | Maintainability §4.2.7 ↑, Performance §4.2.3 ↓ | Sequential dependency accepted for modularity and testability |

#### 8.3 Sensitivity and Trade-off Points

List any **sensitivity points** (where a small change significantly affects quality) and **trade-off points** (where improving one characteristic degrades another). Include these in the Architecture Overview.

### 9. Architecture Gates

After all views are generated and evaluated, verify the architecture artifact against these hard gates before writing output. Each gate is an **ERROR** if failed — do not proceed with failures.

1. **No Implementation Leakage (ERROR)**: Verify that no view contains concrete classes, file paths, function names, DTO fields, database tables, framework selections, or deployment manifests. Architecture views must stay at design-entity level per IEEE 1016 (§4.1). Architecture-level type descriptions (e.g., "String", "Array<REQ>", "Dict[str, int]") are permitted in the Interface View — these describe contract types, not implementation DTOs. Prohibited items are language-specific or framework-specific constructs (e.g., "java.util.List<CustomerDTO>", "React.useState<OrderForm>").

2. **Boundary Completeness (ERROR)**: Verify every stable boundary identified in the Architecture Framing (Step 4) has an explicit **forbidden crossing** documented in the output's Architecture Framing section. If a boundary has responsibilities but no forbidden crossing, add it now.

3. **Decision Traceability (ERROR)**: Verify every major architectural decision (one that affects more than one view or introduces a cross-cutting element) is tied to a REQ, a quality characteristic (ISO 25010), or an explicit tradeoff. If a decision lacks consequence, affected view, or justification, document the missing information.

4. **Invariant Grounding (ERROR)**: Verify every invariant listed in the Architecture Framing is tied to a requirement from `requirements.md`, a scenario from `spec.md`, or an explicit architectural constraint. If an invariant cannot be grounded, remove it or flag it as `[UNGROUNDED]` for human review.

5. **No Invented Capabilities (ERROR)**: Verify all ARCH elements trace to REQ-NNN from `requirements.md` or are explicitly flagged as `[DERIVED REQUIREMENT]` / `[DERIVED MODULE]` with rationale in the Derived Requirements section. Remove any ARCH element not grounded in a requirement and not explicitly flagged as derived.

6. **Gap Explicitness (ERROR)**: Verify all uncertain areas are recorded as specific gaps in the appropriate per-view Gaps table with affected element/link/interface/flow and why it matters. Generic "TBD" markers without scope are not acceptable. If uncertainty exists but no gap is recorded, add the gap entry now.

**Output**: Record the gate results in the `## Architecture Gates` section of the generated artifact. Include each gate's status (✅ Passed / ❌ Failed) and a brief evidence note.

### 10. Write Output

Write the final artifact to `{VMODEL_DIR}/software-architecture-design.md` using the template structure. Include:
- header metadata (feature, branch, date)
- source references for `requirements.md`
- **Architecture Framing** (intent, core tensions, stable boundaries, change axes, invariants, anti-patterns — from Step 4)
- IEEE 1016/42010 synthesized architecture views (Logical, Process, Interface, Data Flow)
- **Per-view gap tables** (Logical View Gaps, Process View Gaps, Interface View Gaps, Data Flow View Gaps — from Step 7)
- Architecture evaluation (ISO 42030 + ISO 25010)
- **Architecture Gates** verification results (6 gates with status and evidence — from Step 9)
- traceability summary and coverage metrics
- derived requirement/module records

### 11. Finish

If the output is complete and valid, conclude with a summary that confirms:
- `REQ-NNN` → `ARCH-NNN` coverage (forward and backward)
- all four views populated with non-placeholder content (SC-002a)
- architecture evaluation performed (ISO 42030 + ISO 25010)
- no silent derived artifacts — all flagged for human review
- Path A coexistence warning (if `architecture-design.md` was detected)
- **Architecture gates**: all 6 passed, or explicit failures with required remediation
- **Gaps recorded**: N gaps across views (with affected-view breakdown)
