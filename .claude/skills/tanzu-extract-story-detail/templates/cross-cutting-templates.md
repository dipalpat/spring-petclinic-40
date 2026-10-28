# Cross-Cutting Templates

## Security Requirements Template

```markdown
# Security Requirements

## Overview

Cross-cutting security concerns that apply across all business domains.

## Authentication

### US-SEC-001: {Authentication Story Title}

#### User Story

**As a** {actor},  
**I want to** {authentication capability},  
**So that** {security benefit}.

#### Acceptance Criteria

- [ ] {criterion}

#### Technical Context

```yaml
SecurityRequirement:
  authentication_method: {FORM|BASIC|BEARER_TOKEN|OAUTH2|SAML|LDAP|CERTIFICATE}
  session_management: {STATEFUL|STATELESS|HYBRID}
  protected_endpoints:
    - pattern: "{url_pattern}"
      access: "{access_rule}"
  legacy_config:
    source: "{configuration_source}"
```

## Authorization

### US-SEC-002: {Authorization Story Title}

{... follow same pattern ...}

## Role Definitions

| Role | Description | Permissions |
|------|-------------|-------------|
| {ROLE_NAME} | {description} | {list of permissions} |
```

## External Integrations Template

```markdown
# External Integrations

## Overview

External systems and services the application integrates with.

---

## US-INT-001: {Integration Story Title}

### User Story

**As a** {actor},  
**I want** {integration capability},  
**So that** {business benefit}.

### Acceptance Criteria

- [ ] {criterion}

### Technical Context

```yaml
IntegrationPoint:
  name: "{integration_name}"
  type: {DATABASE|MESSAGE_QUEUE|REST_API|SOAP_API|FILE_SYSTEM|FTP|EMAIL|LDAP|CACHE}
  direction: {INBOUND|OUTBOUND|BIDIRECTIONAL}
  protocol: "{protocol}"
  endpoint: "{endpoint_url_or_name}"
  authentication:
    type: {NONE|BASIC|BEARER_TOKEN|API_KEY|OAUTH2|CERTIFICATE}
    credentials_source: "{source}"
  data_format: {JSON|XML|CSV|BINARY|FIXED_WIDTH}
  retry_policy:
    max_attempts: {number}
    backoff_ms: {milliseconds}
  circuit_breaker: {true|false}
  timeout_ms: {milliseconds}
```

### Affected Domains

- {Domain 1}
- {Domain 2}
```

## Batch Processes Template

```markdown
# Batch Processes & Scheduled Tasks

## Overview

Background jobs and scheduled tasks that run outside of user requests.

---

## US-BATCH-001: {Batch Job Title}

### User Story

**As a** {actor},  
**I want** {scheduled/batch capability},  
**So that** {business benefit}.

### Acceptance Criteria

- [ ] {criterion}

### Technical Context

```yaml
ServiceTask:
  name: "{task_name}"
  description: "{purpose}"
  schedule_type: {CRON|FIXED_RATE|FIXED_DELAY}
  cron_expression: "{cron_expr}"
  concurrency_policy: {SKIP|QUEUE|PARALLEL}
  failure_handling: "{handling_description}"
```

Or for batch processes:

```yaml
ServiceBatchProcess:
  name: "{job_name}"
  description: "{purpose}"
  trigger: {SCHEDULED|EVENT|MANUAL|FILE_ARRIVAL}
  steps:
    - name: "{step_name}"
      type: {CHUNK|TASKLET|FLOW|DECISION}
      reader: "{reader_description}"
      processor: "{processor_description}"
      writer: "{writer_description}"
  chunk_size: {number}
  restart_policy: {ALWAYS|NEVER|ON_FAILURE}
  error_handling:
    skip_limit: {number}
    retry_limit: {number}
```
```
