# spec-os

A structured specification system for defining software, AI behavior, and system architecture in a deterministic and machine-enforceable way.

---

## 🧭 Overview

`spec-os` is not a documentation repository.

It is a **specification operating system** that defines how systems should be:

- Designed
- Written
- Validated
- Interpreted by both humans and AI

It serves as a **single source of truth for system structure and behavior**, covering:

- Software architecture
- API and code contracts
- AI / LLM interaction rules
- Prompt engineering standards
- Project-level design constraints

---

## 🎯 Core Philosophy

Modern systems fail not because of implementation complexity, but because of **specification inconsistency**.

`spec-os` solves this by enforcing:

- 📌 Determinism → same input, same interpretation
- 📌 Structure → everything has a defined schema
- 📌 Verifiability → specs can be validated by CI
- 📌 Machine readability → AI can interpret and follow rules
- 📌 Human clarity → engineers can reason about systems precisely

---

## 🧱 System Layers

`spec-os` is organized into three conceptual layers:

### 1. Language Layer (How to write specs)

Defines the structure and syntax of specifications.

Examples:
- DocSpec (documentation rules)
- CodeSpec (code structure rules)
- PromptSpec (AI prompt standards)
- ProjectSpec (system design rules)

---

### 2. Schema Layer (How to validate specs)

Defines machine-readable constraints:

- JSON Schema definitions
- YAML / DSL structures
- AST-like representations
- Validation rules

This layer ensures all specs are **checkable and enforceable**.

---

### 3. Runtime Layer (How specs are executed)

Defines tooling built on top of specs:

- CI validators
- Lint systems for AI output
- Spec compliance checkers
- Code / prompt generators

This layer turns specifications into **active system behavior enforcement**.

---

## 📁 Repository Structure

```
spec-os/
├── README.md
├── specs/
│   ├── docspec/
│   ├── devspec/
│   ├── promptspec/
│   ├── codespec/
│   ├── projectspec/
│   └── agentspec/
│
├── schemas/
│   ├── docspec.schema.json
│   ├── codespec.schema.json
│   └── apispec.schema.json
│
├── rules/
│   ├── naming.rules.md
│   ├── structure.rules.md
│   ├── validation.rules.md
│
├── examples/
│   ├── valid/
│   └── invalid/
│
├── tools/
│   ├── validator/
│   ├── linter/
│   ├── parser/
│
└── versioning.md
```

## 🤖 AI Integration Principle

`spec-os` assumes AI is a first-class participant in system design.

Therefore:

- AI outputs must conform to schemas
- Prompts must follow PromptSpec
- Code generation must follow CodeSpec
- Agent behavior must follow AgentSpec

This enables:

> AI systems that are constrained, predictable, and testable

---

## 🔒 Key Design Rules
- Everything must be specifiable
- Everything specifiable must be validate-able
- No implicit system behavior
- No undocumented assumptions
- All AI outputs are structured artifacts

---

🚀 Goals
- Create a universal specification standard for AI-native systems
- Enable CI-verifiable prompt and code correctness
- Reduce ambiguity in system design
- Bridge human intent and machine execution

---

🧪 Future Extensions

Planned directions:

- Visual spec editor (UI DSL builder)
- AI-powered spec validator
- Auto-generation of services from specs
- Cross-model compatibility layer (GPT / Claude / local LLMs)
- Spec-to-code compiler

---

📌 Status

Early-stage framework.

Core philosophy and structure are being actively designed and iterated.

---

🧠 Summary

> `spec-os` defines how systems should be written so that both humans and AI cannot misunderstand them.