# DocSpec v2.0 — Documentation Specification Standard

**Version:** 2.0  
**Status:** Normative  
**Format:** Markdown (.md)  
**Scope:** All project documentation subject to CI validation

---

# 1. Overview

This specification defines the structure, vocabulary, syntax, and validation rules for all documentation files governed by DocSpec v2.0.

Compliance is mandatory. Any document that violates a rule defined herein MUST be rejected by the CI validation system. Rejected documents MUST NOT be merged, published, or distributed.

This specification is designed to be:

- Deterministic across all LLM systems (GPT, Claude, Grok, and equivalents)
- Machine-checkable by external CI tooling
- Human-readable and consistently followable
- Modular and backward-compatible across future versions

All normative requirements use the keywords MUST, MUST NOT, SHALL, SHALL NOT, SHOULD, and SHOULD NOT as defined in RFC 2119.

---

# 2. Document Types Definition

Each governed document MUST declare exactly one of the following types in its front matter block.

| Type Token        | Purpose                                                  |
|-------------------|----------------------------------------------------------|
| `API_REFERENCE`   | Describes API endpoints, parameters, and responses       |
| `INTEGRATION`     | Describes integration setup between systems              |
| `RUNBOOK`         | Describes operational procedures and incident response   |
| `TEST_SUITE`      | Contains Test Blocks validated by CI                     |
| `ARCHITECTURE`    | Describes system design and component relationships      |
| `CHANGELOG`       | Records version history and modifications                |

**Rules:**

- Every document MUST begin with a front matter block using the format below.
- The `type` field MUST contain exactly one Type Token from the table above.
- The `type` field MUST NOT contain freeform text.
- The `docspec` field MUST read `"2.0"` for compliance with this version.

**Required Front Matter Format:**

```yaml
---
docspec: "2.0"
type: TEST_SUITE
title: "Human-readable document title"
version: "1.0.0"
status: "draft | review | approved"
---
```

- The `status` field MUST be exactly one of: `draft`, `review`, or `approved`.
- The `title` field MUST NOT be empty.
- The `version` field MUST follow semantic versioning (`MAJOR.MINOR.PATCH`).

---

# 3. Markdown Structure Rules

## 3.1 Heading Hierarchy

- The document MUST contain exactly one `# ` (H1) heading, which MUST be the document title.
- Headings MUST follow strict sequential order: H1 → H2 → H3 → H4.
- Heading levels MUST NOT be skipped. An H4 MUST NOT appear unless an H3 precedes it in the same section.
- Headings MUST NOT be formatted with bold, italic, or inline code markers.
- Heading text MUST NOT end with punctuation marks (`.`, `!`, `?`).

## 3.2 Section Ordering

- Sections MUST appear in the order defined by this specification or by the document type's schema.
- No section defined as required MUST be omitted.
- Optional sections MUST NOT appear before required sections.

## 3.3 Lists

- Unordered lists MUST use `-` as the bullet marker.
- Ordered lists MUST use `1.`, `2.`, `3.` numeric markers.
- List items MUST NOT be empty.
- Nested lists MUST be indented with exactly two spaces per level.

## 3.4 Code Blocks

- All code blocks MUST declare a language identifier (e.g., ` ```bash `, ` ```powershell `, ` ```json `).
- Code blocks MUST NOT use the generic ` ``` ` identifier without a language tag.
- Code blocks MUST NOT be nested inside one another.

## 3.5 Tables

- Tables MUST include a header row and a separator row.
- All columns MUST be aligned using pipe (`|`) characters.
- Tables MUST NOT be used to represent Test Blocks.

## 3.6 Line Length and Whitespace

- Trailing whitespace MUST NOT appear on any line.
- Documents MUST end with exactly one newline character.
- Blank lines MUST NOT appear consecutively more than twice.

---

# 4. Keyword Lock System

The following keywords are locked. They define the canonical vocabulary of DocSpec v2.0.

## 4.1 Locked Keyword Registry

| Locked Keyword    | Permitted Context                        |
|-------------------|------------------------------------------|
| `Command`         | Test Block field label                   |
| `Expected Result` | Test Block field label                   |
| `Failure Case`    | Test Block field label                   |
| `Test`            | Section prefix in `### Test:` headings   |

## 4.2 Keyword Rules

- Locked keywords MUST NOT be renamed.
- Locked keywords MUST NOT be abbreviated (e.g., `Cmd`, `Exp Result`, `Fail` are forbidden).
- Locked keywords MUST NOT be replaced by synonyms (e.g., `Command` MUST NOT be replaced with `Step`, `Action`, `Input`, `Instruction`, or any equivalent).
- Locked keywords MUST appear with identical casing as listed in Section 4.1.
- Locked keywords MUST NOT be translated into other languages.
- Locked keywords MUST NOT be modified across DocSpec versions without a major version increment.

---

# 5. Test Block DSL

A Test Block is the core unit of documentation in `TEST_SUITE` documents. Test Blocks MAY also appear in `RUNBOOK` and `INTEGRATION` documents.

## 5.1 Test Block Declaration

Every Test Block MUST be declared using the following heading syntax:

```
### Test: <Test Name>
```

- `Test:` MUST appear exactly as written, including the colon.
- `<Test Name>` MUST be a concise, unique, human-readable identifier within the document.
- Test Names MUST NOT contain special characters other than hyphens (`-`) and spaces.
- Two Test Blocks within the same document MUST NOT share the same Test Name.

## 5.2 Required Test Block Fields

Every Test Block MUST contain all five of the following fields, in the following order:

1. **Purpose**
2. **Command**
3. **Expected Result**
4. **Failure Case**

The field label MUST appear as a bold inline marker followed by a colon (e.g., `**Purpose**:`).

No field MUST be omitted. A Test Block missing any required field MUST be rejected by CI.

## 5.3 Purpose Field

- MUST be a single paragraph of plain prose.
- MUST describe what the test validates and why it exists.
- MUST NOT contain code blocks, lists, or tables.

## 5.4 Command Field

- MUST contain exactly two labeled platform subsections: Linux/macOS and Windows.
- Each subsection MUST use the labels defined in Section 6.
- Each subsection MUST contain exactly one fenced code block with the correct language identifier.
- Commands MUST NOT be mixed across platform code blocks.
- Placeholder values MUST be denoted using angle brackets (e.g., `<your-api-key>`).

**Required Command Structure:**

```
**Command**:

_Linux/macOS:_

```bash
<bash command here>
```

_Windows:_

```powershell
<powershell command here>
```
```

## 5.5 Expected Result Field

The Expected Result field MUST include all three of the following sub-elements when applicable:

- **HTTP Status Code** — MUST be present if the Command targets an HTTP endpoint. Format: `HTTP <code> <reason>` (e.g., `HTTP 200 OK`).
- **Example Response Payload** — MUST be a fenced code block with an appropriate language tag (e.g., ` ```json `).
- **Success Condition** — MUST be a plain-prose statement describing the condition that constitutes a passing result.

If the Command does not target an HTTP endpoint, the HTTP Status Code sub-element MUST be replaced with the label `N/A — Non-HTTP command` on its own line.

## 5.6 Failure Case Field

The Failure Case field MUST contain:

- A bullet-point list of discrete failure conditions.
- Each bullet MUST describe one specific failure mode.
- Common error codes MUST be included where applicable, using the format: `<CODE>: <description>` (e.g., `401: Unauthorized — invalid or missing API key`).
- The list MUST contain at least one item.
- Vague entries such as "Something went wrong" or "Unknown error" MUST NOT appear.

---

# 6. Multi-Platform Rules

All Command fields MUST provide coverage for both of the following platforms.

| Platform      | Label (exact)   | Code Block Language Tag |
|---------------|-----------------|-------------------------|
| Linux/macOS   | `_Linux/macOS:_`  | `bash`                  |
| Windows       | `_Windows:_`      | `powershell`            |

- Platform labels MUST appear in italic using underscore syntax (`_label_`).
- Platform labels MUST match the Label column in the table above exactly.
- The Linux/macOS section MUST appear before the Windows section.
- Each platform MUST have its own independent code block.
- A single code block covering multiple platforms MUST NOT be used.
- Platform-specific behavior differences MUST be documented inline as comments within the respective code block, using the native comment syntax for that platform (`#` for bash, `#` for PowerShell).

---

# 7. Validation Rules (CI Assumptions)

This section defines the rules enforced by the external CI validation system. The CI system is authoritative. Document authors MUST NOT assume any rule listed here will be waived.

## 7.1 Structural Validation

| Rule ID | Rule Description                                                              |
|---------|-------------------------------------------------------------------------------|
| S-01    | Document MUST contain a valid front matter block as defined in Section 2      |
| S-02    | `docspec` field MUST equal `"2.0"`                                            |
| S-03    | `type` field MUST contain a valid Type Token from Section 2                   |
| S-04    | `status` field MUST be one of: `draft`, `review`, `approved`                 |
| S-05    | Heading levels MUST NOT be skipped                                            |
| S-06    | Document MUST end with exactly one newline character                          |
| S-07    | No trailing whitespace MUST be present on any line                            |
| S-08    | All code blocks MUST declare a language identifier                            |

## 7.2 Test Block Validation

| Rule ID | Rule Description                                                              |
|---------|-------------------------------------------------------------------------------|
| T-01    | Every `### Test:` heading MUST be followed by all four required fields        |
| T-02    | Fields MUST appear in the order: Purpose, Command, Expected Result, Failure Case |
| T-03    | Every Command field MUST contain both Linux/macOS and Windows subsections     |
| T-04    | Platform subsections MUST use exact labels from Section 6                     |
| T-05    | Platform subsections MUST NOT share a single code block                       |
| T-06    | Failure Case MUST contain at least one bullet-point item                      |
| T-07    | Test Names MUST be unique within a document                                   |
| T-08    | Expected Result MUST include HTTP status or `N/A` declaration                 |

## 7.3 Keyword Validation

| Rule ID | Rule Description                                                              |
|---------|-------------------------------------------------------------------------------|
| K-01    | Locked keywords MUST appear with exact casing as defined in Section 4.1      |
| K-02    | No synonym or abbreviation of a locked keyword MUST be present                |
| K-03    | `### Test:` prefix MUST NOT be altered                                        |

## 7.4 Violation Handling

- The CI system MUST reject any document that violates one or more rules in this section.
- The CI system MUST output the Rule ID(s) violated in its rejection message.
- The CI system MUST NOT auto-correct or partially accept invalid documents.
- Document authors MUST address all reported violations and resubmit.
- Regeneration of AI-produced documents MUST be triggered externally by the author or pipeline operator.
- The CI system MUST NOT merge, publish, or distribute rejected documents under any circumstance.

---

# 8. Valid Example

```yaml
---
docspec: "2.0"
type: TEST_SUITE
title: "Authentication API Test Suite"
version: "1.0.0"
status: "approved"
---
```

# Authentication API Test Suite

## Login Endpoint Tests

### Test: Valid Credential Login

**Purpose**: Validates that a registered user can authenticate using correct credentials and receives a valid session token in response.

**Command**:

_Linux/macOS:_

```bash
curl -X POST https://api.example.com/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "<your-password>"}'
```

_Windows:_

```powershell
Invoke-RestMethod -Method POST -Uri "https://api.example.com/auth/login" `
  -ContentType "application/json" `
  -Body '{"username": "testuser", "password": "<your-password>"}'
```

**Expected Result**:

HTTP 200 OK

```json
{
  "status": "success",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600
}
```

Success Condition: The response body contains a non-empty `token` field and `status` equals `"success"`.

**Failure Case**:

- `401`: Unauthorized — credentials do not match any registered user
- `400`: Bad Request — malformed JSON body or missing required fields
- `429`: Too Many Requests — rate limit exceeded; retry after the duration specified in the `Retry-After` header
- `500`: Internal Server Error — contact the platform team if this persists

---

# 9. Invalid Example (with Explanation)

The following document fragment is invalid under DocSpec v2.0. The CI system MUST reject it.

```markdown
### Check: Valid Credential Login

**Goal**: Makes sure login works.

**Cmd**:

```
curl -X POST https://api.example.com/auth/login
```

**Result**:

Returns a token.

**Errors**:

- Something went wrong
```

**Violations Detected:**

| Rule ID | Violation Description                                                                            |
|---------|--------------------------------------------------------------------------------------------------|
| K-03    | `### Check:` MUST be `### Test:` — the heading prefix is a locked keyword and MUST NOT be altered |
| K-01    | `**Goal**` MUST be `**Purpose**` — `Goal` is not a permitted locked keyword label               |
| K-01    | `**Cmd**` MUST be `**Command**` — abbreviations of locked keywords are forbidden                 |
| S-08    | Code block has no language identifier — every code block MUST declare a language tag             |
| T-05    | Command contains only one platform block — both Linux/macOS and Windows MUST be present          |
| T-08    | Expected Result contains no HTTP status code or `N/A` declaration                               |
| K-01    | `**Result**` MUST be `**Expected Result**` — partial keyword is not permitted                    |
| K-01    | `**Errors**` MUST be `**Failure Case**` — synonyms of locked keywords are forbidden             |
| T-06    | Failure Case entry "Something went wrong" is vague and MUST NOT appear                          |

This document MUST NOT be merged. The author MUST correct all listed violations and resubmit for CI validation.

---

# 10. Extensibility Rules

## 10.1 Adding New Document Types

- New Type Tokens MAY be added to the registry in Section 2.
- Each new Type Token MUST be accompanied by a schema definition specifying its required sections.
- New Type Tokens MUST be uppercase, underscore-delimited strings (e.g., `DEPLOYMENT_GUIDE`).
- Existing Type Tokens MUST NOT be removed or renamed without a major version increment.

## 10.2 Adding New Sections to the Specification

- New sections MUST be appended after Section 10 or inserted into an appropriate numbered subsection.
- Heading levels of new sections MUST conform to the hierarchy defined in Section 3.1.
- New sections MUST NOT reuse heading text already present in the document.
- New sections MUST be assigned a unique section number.

## 10.3 Modifying Test Block Structure

- The five required Test Block fields (Purpose, Command, Expected Result, Failure Case, and Test Name) MUST NOT be removed.
- Additional optional fields MAY be appended after the Failure Case field.
- Optional fields MUST be documented in this specification before use.
- Optional fields MUST NOT use names that conflict with locked keywords.

## 10.4 Keyword Lock Immutability

- Locked keywords defined in Section 4.1 MUST NOT be changed in any minor or patch version.
- Changes to locked keywords require a major version increment of this specification.
- All documents governed by a prior major version remain valid under that version only.
- Migration guides MUST be published when locked keywords are changed between major versions.

## 10.5 Backward Compatibility

- Documents valid under DocSpec v2.x MUST remain valid under all subsequent v2.x releases.
- A DocSpec minor version increment MUST NOT introduce breaking changes to existing validation rules.
- A DocSpec patch version increment MUST be limited to clarifications and non-normative corrections.
- Breaking changes MUST only be introduced in a new major version (e.g., DocSpec v3.0).

## 10.6 Version Declaration on Extension

- Any document adopting extended or custom sections MUST still declare `docspec: "2.0"` in its front matter.
- Extension schemas MUST NOT override the front matter format defined in Section 2.
- Extension schemas MUST be maintained in a separate, versioned schema registry file referenced by the implementing team.
