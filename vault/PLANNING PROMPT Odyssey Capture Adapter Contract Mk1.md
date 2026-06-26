# PLANNING PROMPT Odyssey Capture Adapter Contract Mk1

Repository:

```text
odyssey-chatgpt-capture
```

Read the following documents in the exact order listed below:

```text
1. DESIGN. Odyssey Storage Array Mk1 Baseline.md
2. DESIGN. Odyssey Storage Service Mk1.md
```

These documents are frozen architectural references.

The Storage Array Baseline defines:

```text
WHAT exists
```

The Storage Service defines:

```text
HOW captured reality is preserved
```

Your task is to define:

```text
How Reality Capture Adapters interact with Storage Service.
```

---

# Objective

Create:

```text
docs/DESIGN. Odyssey Capture Adapter Contract Mk1.md
```

Documentation only.

Do not modify code.

Do not modify extension files.

Do not modify API files.

Do not create package files.

Do not install dependencies.

Do not create Docker files.

Do not modify existing design documents.

Create only the requested document.

---

# Architectural Alignment

The document must align with the following Odyssey model.

```text
Adapters capture fragments of Reality.

Storage Service preserves captured fragments.

Spark assembles (sews) fragments of Reality into Pieces of Reality.

Spirit observes a Piece of Reality to derive meaning, make sense of it, and augment Spirit knowledge and experience.
```

The document must preserve this separation of responsibilities.

Adapters do not perform Spark functions.

Adapters do not perform Spirit functions.

Adapters do not perform Storage Service functions.

---

# Core Principle

Define:

```text
Capture = Discover + Normalize + Submit

Preserve = Validate + Resolve Identity + Enforce Policy + Persist + Index + Link
```

Explain:

```text
Adapters capture Reality Fragments.

Storage Service preserves Reality Fragments.
```

---

# Architectural Position

Include:

```text
Reality Vector
        ↓
Capture Adapter
        ↓
Normalized Capture Package
        ↓
Storage Service
        ↓
Storage Array
        ↓
Reality Candidate
        ↓
Spark Handoff Package
        ↓
00-Spark
        ↓
Piece of Reality
        ↓
01-Spirit
```

---

# Adapter Definition

Define a Capture Adapter as:

```text
A component responsible for capturing Reality Fragments from a Reality Vector and submitting normalized capture packages to the Storage Service.
```

Examples:

```text
ChatGPT Capture Adapter
WhatsApp Capture Adapter
YouTube Capture Adapter
TikTok Capture Adapter
X Capture Adapter
Email Capture Adapter
Web Capture Adapter
Manual Import Adapter
```

The document must remain adapter-agnostic.

---

# Adapter Responsibilities

Document that adapters may:

```text
Observe
Capture
Discover
Normalize
Submit
Collect Evidence
Report Errors
```

Explain:

```text
Capture
    =
Discover
    +
Normalize
    +
Submit
```

---

# Adapter Restrictions

Document that adapters must not:

```text
Persist directly to MongoDB
Persist directly to Google Drive
Enforce Capture Policies
Resolve Identity
Create Reality Candidates
Perform Spark functions
Perform Spirit functions
Interpret Reality
Summarize Reality
Perform Semantic Analysis
Create Ontologies
```

---

# Clarification About Storage Writes

The document must explicitly state:

```text
Adapters do not write directly to the Storage Array.
```

and also:

```text
Adapters do write indirectly to the Storage Array through the Storage Service.
```

Explain:

```text
Reality Vector
    ↓
Adapter
        captures fragments
        discovers entities
        normalizes payload
        submits package
    ↓
Storage Service
        validates
        resolves identity
        enforces policy
        persists to Storage Array
    ↓
Storage Array
```

---

# Ownership Boundaries

Adapters own:

```text
Discovery
Normalization
Evidence Collection
Submission
```

Storage Service owns:

```text
Identity Resolution
Capture Policy Enforcement
Persistence
Versioning
Relationship Creation
MongoDB Synchronization
Google Drive Synchronization
Capture Run Management
```

Spark owns:

```text
Reality Assembly
Reality Sewing
Spark Handoff Production
```

Spirit owns:

```text
Observation
Meaning Derivation
Knowledge Augmentation
Experience Augmentation
```

---

# Supported Entity Discovery

Document that adapters may discover descriptions of:

```text
Capture Source
Project
Project Source
Capture Stream
Capture Fragment
Artifact
Payload
Storage Object
Actor
```

Adapters do not create these entities directly.

Storage Service creates and persists them.

---

# Adapter Lifecycle

Provide lifecycle diagram:

```text
Observe Source
        ↓
Capture Reality Fragments
        ↓
Discover Entities
        ↓
Normalize Data
        ↓
Build Capture Package
        ↓
Submit To Storage Service
        ↓
Receive Result
```

---

# Normalized Capture Package

Describe the package concept.

Use the same package shape already defined in Storage Service.

Maintain complete alignment with existing design documents.

Do not redefine the package.

---

# Error Handling

Define:

```text
Discovery Errors
Normalization Errors
Submission Errors
```

Adapters report errors.

Storage Service decides persistence outcomes.

---

# Adapter Types

Describe:

```text
Browser Extension Adapter
API Connector Adapter
Webhook Adapter
CLI Import Adapter
Batch Import Adapter
Scheduled Adapter
File Import Adapter
```

No implementation details.

No technology selections.

---

# Non-Goals

Adapters do not perform:

```text
Reality Interpretation
Semantic Analysis
AI Summarization
Ontology Creation
Storage Management
Candidate Assembly
Spark Sewing
Spirit Observation
```

---

# Core Formula

End with:

```text
Observe
        ↓
Capture
        ↓
Discover
        ↓
Normalize
        ↓
Submit
        ↓
Preserve
```

---

# Deliverable

Create only:

```text
docs/DESIGN. Odyssey Capture Adapter Contract Mk1.md
```

At completion provide:

```text
File created

Summary of sections included

Verification that no code files were modified

Verification that only documentation files were modified
```
