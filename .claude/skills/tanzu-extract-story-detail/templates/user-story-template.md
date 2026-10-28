# {US-ID}: {Title}

## User Story

**As a** {actor/role},  
**I want to** {action/capability},  
**So that** {business benefit/value}.

## Acceptance Criteria

Number each criterion (AC-1, AC-2, …) so test cases can reference them via the **AC Ref** column.

- [ ] **AC-1** {Criterion 1 - testable condition}
- [ ] **AC-2** {Criterion 2 - testable condition}
- [ ] **AC-3** {Criterion 3 - testable condition}

## Technical Context

> One user story = one service request = one endpoint + one primary operation. Every endpoint that reads or writes data must have a corresponding Service Operation.

### Endpoint

```yaml
ServiceEndpoint:
  path: "{endpoint_path}"
  method: {HTTP_METHOD}
  path_variables:
    - name: {var_name}
      type: {var_type}
      description: "{description}"
  query_parameters:
    - name: {param_name}
      type: {param_type}
      required: {true|false}
  request_body: {DataStructure_name}
  response_type: {JSON|XML|VIEW|REDIRECT|FILE|VOID}
  response_body: {DataStructure_name}
  authentication: {ANONYMOUS|AUTHENTICATED|API_KEY}
  authorization: ["{ROLE_1}", "{ROLE_2}"]
  error_responses:
    - status_code: {code}
      condition: "{when this error occurs}"
```

### Service Operation

```yaml
ServiceOperation:
  name: "{operation_name}"
  description: "{business purpose}"
  type: {QUERY|COMMAND|EVENT_HANDLER|SCHEDULED|BATCH_STEP}
  inputs:
    - name: {param_name}
      type: {param_type}
      required: {true|false}
      validation: ["{validation_rules}"]
  output: {return_type}
  transactional: {true|false}
  transaction_propagation: {REQUIRED|REQUIRES_NEW|...}
  read_only: {true|false}
  preconditions:   # business rules; do not repeat in steps
    - id: "{precondition_id}"
      description: "{human-readable rule}"
      condition: "{optional formal condition}"
      inputs_used: ["{input_name}"]
      failure_exception: {ExceptionClass}
      failure_message: "{optional message}"
  steps:   # use $name for variables
    - "Get {Entity} as ${var} (by {input})"
    - "Update ${var}'s ... or ..."
    # For QUERY operations: one step per output property
    # - "Set output.field = $source to load the data"
  structural_events:   # single source of what changes
    - kind: {ENTITY_CREATED|ENTITY_UPDATED|ENTITY_DELETED|MESSAGE_SENT}
      entity: "{DataStructure_name}"
      instance_variable: "${var}"
      attributes: ["{prop1}", "{prop2}"]
      value_source: "{input.path}"
      description: "{one-line summary}"
  exceptions:
    - type: {ExceptionClass}
      condition: "{when thrown}"
```

### Data Structures Referenced

- `{EntityName}` - {brief description}
- `{DTOName}` - {brief description}

### Integrations

- `{integration_name}` - {brief description}

### Source Files (Legacy)

- `{package/path/ClassName.java}` - Lines {start}-{end}

## UI Specification

> Include this section only for endpoints with `response_type: VIEW`.

### Page: {Page Title}

**URL:** `{url_path}`  
**View Name:** `{view_name}`  
**Source File:** `{path/to/view.jsp}`

### Wireframe

```
┌─────────────────────────────────────────┐
│  [Header]                               │
├─────────────────────────────────────────┤
│  Page content layout                    │
└─────────────────────────────────────────┘
```

### Form Fields (if applicable)

| Field | Type | Required | Validation |
|-------|------|----------|------------|

### User Interactions

| Trigger | Action | Result |
|---------|--------|--------|

## Test Scenarios

> Use Given-When-Then format with concrete values. Test cases must cover happy path, errors, validation, edge cases, and UI (if applicable).

### Happy Path Tests

| ID | Scenario | Preconditions (Given) | Input (When) | Expected Result (Then) | AC Ref | Validates with | Source |
|----|----------|----------------------|--------------|----------------------|--------|----------------|--------|
| TC-{NNN}-01 | {successful_scenario} | {concrete_state} | {concrete_input} | {concrete_outcome} | AC-1 | {operation(params)} | {test_file} |

### Error/Validation Tests

| ID | Scenario | Input (When) | Expected Error (Then) | AC Ref | Validates with | Source |
|----|----------|--------------|---------------------|--------|----------------|--------|
| TC-{NNN}-02 | {validation_failure} | {invalid_input} | {error_message} | AC-x | {operation(params)} | |

### Edge Cases

| ID | Scenario | Input (When) | Expected Result (Then) | AC Ref | Validates with | Source |
|----|----------|--------------|----------------------|--------|----------------|--------|
| TC-{NNN}-05 | {boundary_condition} | {edge_value} | {expected_behavior} | AC-x | {operation(params)} | |

### UI Tests (for VIEW endpoints)

| ID | Scenario | User Action (When) | Expected Visual Result (Then) | AC Ref | Validates with |
|----|----------|--------------------|----------------------------|--------|----------------|
| TC-{NNN}-UI-01 | {ui_scenario} | {click_or_action} | {what_user_sees} | AC-x | Manual (UI) |

### Test case sources

| Source file | TC-* IDs |
|-------------|----------|
| {test_file_path} | TC-001, TC-002 |

### Strategic questions for test cases requiring verification

| TC ID | Question or decision | Resolution |
|-------|---------------------|------------|
| TC-{NNN}-01 | {verification_question} | **Keep** / **Update** / **Remove** |

### Test Data Examples

```yaml
valid_input:
  field1: "example_value"
  field2: 123

invalid_inputs:
  - scenario: "missing required field"
    data: { field2: 123 }
```

## Strategic Questions

### Business Rules

| ID | Question | Why It Matters | Answer |
|----|----------|----------------|--------|
| Q-{NNN}-01 | {question_text} | {impact on extraction} | _To be answered_ |

### Data Integrity / Schema

| ID | Question | Why It Matters | Answer |
|----|----------|----------------|--------|
| Q-{NNN}-02 | {question_text} | {impact on entity design} | _To be answered_ |

### Integration / Third-Party

| ID | Question | Why It Matters | Answer |
|----|----------|----------------|--------|
| Q-{NNN}-04 | {question_text} | {impact on integration} | _To be answered_ |

### Test Case Verification

| ID | Question | Why It Matters | Answer |
|----|----------|----------------|--------|
| Q-{NNN}-05 | {question_text} | {what needs verification} | _To be answered_ |

## Priority

**{HIGH|MEDIUM|LOW}** - {Brief justification}

## Complexity

**{LOW|MEDIUM|HIGH|VERY_HIGH}** - {Brief justification}

## Migration phase

**{1|2|3|4}** - {Brief rationale based on complexity and dependencies}

## Reviewer Validation

### Business Accuracy
- [ ] User story reflects actual business need
- [ ] Acceptance criteria are complete and testable
- [ ] Priority assignment is correct

### Technical Completeness
- [ ] All endpoints are documented correctly
- [ ] Data structures match actual schema
- [ ] Integrations are identified
- [ ] Migration phase assignment correct

### Test Coverage
- [ ] Happy path tests are realistic
- [ ] Error cases cover actual failure modes
- [ ] Edge cases are identified
- [ ] Test cases without existing implementation verified (Keep/Update/Remove documented)

### Questions Resolved
- [ ] All strategic questions have answers
- [ ] No blocking questions remain

**Reviewed by:** _________________ **Date:** _________
