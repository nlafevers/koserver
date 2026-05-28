# Implementation Roadmap or Checklist Annotation Format

This project uses a strict annotation format to track the state of implementation steps. AI Agents and human developers must adhere to this format to ensure state-tracking remains accurate and parseable.

## State Annotations

- `[ ]` **Uncompleted**: Work has not yet started.
- `[-]` **Work in Progress (WIP)**: Implementation is currently underway. If an agent encounters this state upon startup, it indicates a previously interrupted session.
- `[x]` **Completed**: Work is finished, verified, and committed.

## Formatting Rules

1. **Unique Identifiers**: Every actionable step MUST begin with a unique identifier (e.g., `TS-01`, `AUTH-102`). This prevents search-and-replace errors when automated tools update the file.
2. **Phase Hierarchy**: Steps should be grouped under Phase headings (`### Phase X`) to allow for batched execution.
3. **Audit Trails**: When marking a step as `[x]` Completed, agents should append the resulting Git Commit hash to the end of the line for traceability.

## Example Roadmap Structure

```markdown
### Phase 1: Foundation
- [x] **INIT-01**: Project Initialization (Commit: a1b2c3d)
- [-] **INIT-02**: Database Schema Initialization
- [ ] **INIT-03**: Configure CI/CD Pipeline

### Phase 2: Core Features
- [ ] **FEAT-01**: Implement User Authentication
- [ ] **FEAT-02**: Create Dashboard Layout