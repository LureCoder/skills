---
name: "feature-analyzer"
description: "Analyzes feature documentation and extracts structured testing requirements. Invoke after reading Confluence pages to parse acceptance criteria, business rules, and test conditions."
---

# Feature Analyzer Skill

This skill analyzes feature documentation from Confluence and extracts structured testing requirements, business rules, and scenarios for downstream test generation.

## Input

The input comes from the **confluence-reader** skill in markdown format:

```markdown
## Feature: User Login
**Page ID**: 123456

### Description
Users can log in using email and password...

### Acceptance Criteria
- AC1: User can log in with valid credentials
- AC2: User sees error on invalid password
- AC3: Account locks after 5 failed attempts

### Business Rules
- Password must be 8-50 characters
- Account locks for 30 minutes
```

## Analysis Pipeline

### Phase 1: Feature Identification

Extract core feature metadata:

```yaml
feature:
  name: "User Login"
  pageId: "123456"
  
  # Extracted from headings, tables, and structured content
  type: "Functional"          # functional / non-functional / technical
  module: "Authentication"    # Business module
  priority: "High"           # From labels or explicit marking
  sprint: "Sprint-24"        # From labels or metadata
```

### Phase 2: Scenario Classification

Classify content into scenario categories:

```
INPUT CONTENT                    CLASSIFIED AS
──────────────────────────────────────────────────
"User can log in with valid"  →  POSITIVE (Happy Path)
"User sees error on invalid"  →  NEGATIVE (Error Path)
"Account locks after 5"       →  BOUNDARY (Threshold)
"Password must be 8-50"       →  VALIDATION (Constraints)
"System responds within 2s"   →  PERFORMANCE (NFR)
"Only admin can reset"        →  SECURITY (Permission)
"Works on mobile and web"     →  COMPATIBILITY (Cross-platform)
```

### Phase 3: Business Rule Extraction

Tables and lists containing rules are normalized:

```yaml
business_rules:
  - id: "BR-001"
    description: "Password length validation"
    condition: "Password length"
    constraint: "8 to 50 characters"
    source: "Acceptance Criteria table"
    
  - id: "BR-002"
    description: "Account lockout threshold"
    condition: "Failed login attempts"
    constraint: "Maximum 5 attempts"
    source: "Business Rules section"
    
  - id: "BR-003"
    description: "Account lockout duration"
    condition: "After account lock"
    constraint: "30 minutes auto-unlock"
    source: "Business Rules section"
```

### Phase 4: Test Condition Generation

From each piece of analyzed content, generate test conditions:

```yaml
test_conditions:
  - id: "TCOND-001"
    derived_from: "AC1 - Valid credentials login"
    type: "POSITIVE"
    
    conditions:
      - "Valid email format (user@example.com)"
      - "Password meets complexity requirements"
      - "Account is active (not locked)"
      - "Account is not expired"
    
  - id: "TCOND-002"
    derived_from: "AC3 - Account lockout"
    type: "BOUNDARY"
    
    conditions:
      - "4 failed attempts → still allowed"
      - "5th failed attempt → account locked"
      - "After 30 minutes → account unlocked"
      - "During lockout → all attempts rejected"
```

### Phase 5: Dependency Mapping

Identify relationships and dependencies:

```yaml
dependencies:
  external_systems:
    - "Authentication Service (LDAP)"
    - "Email Notification Service"
    - "Audit Logging Service"
    
  preconditions:
    - "User must be registered"
    - "Login page must be accessible"
    
  data_requirements:
    - "Registered user accounts with various states"
    - "Password policy configuration"
    
  test_data_templates:
    valid_user:
      email: "testuser@example.com"
      password: "ValidPass123!"
      status: "ACTIVE"
    
    locked_user:
      email: "locked@example.com"
      password: "AnyPass123!"
      status: "LOCKED"
    
    expired_user:
      email: "expired@example.com"
      password: "AnyPass123!"
      status: "EXPIRED"
```

## Analysis Techniques

### Technique 1: Table-Based Analysis

Confluence tables → structured rules:

```markdown
| Input Field   | Validation Rule        | Error Message          |
|---------------|------------------------|------------------------|
| Email         | Required, valid format | "Invalid email"        |
| Password      | Required, 8-50 chars   | "Invalid password"     |
```

Extracted:
```yaml
fields:
  - name: "Email"
    rules:
      - type: "required"
      - type: "format"
        pattern: "email"
    error_message: "Invalid email"
    
  - name: "Password"
    rules:
      - type: "required"
      - type: "length"
        min: 8
        max: 50
    error_message: "Invalid password"
```

### Technique 2: Acceptance Criteria Parsing

Parse structured ACs:

```markdown
AC1: User can log in with valid email and password
     → GIVEN a registered user with valid credentials
       WHEN they enter correct email and password
       THEN they are successfully logged in
       AND redirected to the dashboard

AC2: User sees error message with invalid password
     → GIVEN a registered user
       WHEN they enter incorrect password
       THEN an error message is displayed
       AND they remain on the login page
```

### Technique 3: Diagram/Flow Analysis

If the Confluence page contains flow diagrams (ASCII art or text-based):

```markdown
[Login Page] → [Validate Credentials] → [Dashboard]
     │                │
     ▼                ▼
[Error Message]   [Account Locked]
```

Extracted flows:
```yaml
flows:
  - id: "FLOW-001"
    name: "Happy Path Login"
    steps:
      - "Navigate to login page"
      - "Enter valid credentials"
      - "System validates credentials"
      - "Redirect to dashboard"
      
  - id: "FLOW-002"
    name: "Failed Login"
    steps:
      - "Navigate to login page"
      - "Enter invalid credentials"
      - "System validates credentials"
      - "Show error message"
      - "Stay on login page"
```

## Output Format

The complete analysis output passed to **test-scenario-writer**:

```yaml
feature:
  name: "User Login"
  module: "Authentication"
  type: "Functional"
  
scenarios:
  positive:
    - "Successful login with valid credentials"
    - "Login with remember-me enabled"
    
  negative:
    - "Login with invalid password"
    - "Login with unregistered email"
    
  boundary:
    - "Login at max password length (50 chars)"
    - "Login attempt #5 (account lock threshold)"
    
validation_rules:
  - field: "email"
    rules: ["required", "format:email", "max_length:100"]
  - field: "password"
    rules: ["required", "min_length:8", "max_length:50"]
    
dependencies:
  services: ["Auth Service", "User Service"]
  test_data: { valid_user: "...", locked_user: "..." }
  
risk_areas:
  - "Account lockout race condition"
  - "Concurrent login from multiple devices"
```

## Best Practices

1. **Be conservative** - Only extract what is explicitly stated or clearly implied
2. **Flag ambiguities** - Mark unclear requirements for human review
3. **Preserve source references** - Each extracted item links back to source line
4. **Handle incomplete info** - Note missing acceptance criteria or business rules
5. **Separate concerns** - Functional vs technical vs non-functional requirements
6. **Cross-reference** - Link related rules/conditions to avoid duplication

## Error Handling

| Issue | Action |
|-------|--------|
| No ACs found | Flag warning, extract from descriptions |
| Ambiguous rules | Note ambiguity, suggest clarification |
| Contradictory statements | Flag as risk item for human review |
| Missing test data | Generate placeholder templates |
