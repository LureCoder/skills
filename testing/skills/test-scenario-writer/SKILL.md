---
name: "test-scenario-writer"
description: "Generates structured test scenarios from feature analysis. Invoke after feature analysis is complete to produce test cases in various formats (Markdown, Cucumber, JSON)."
---

# Test Scenario Writer Skill

This skill generates structured, comprehensive test scenarios from the feature analysis output. It produces standardized test cases suitable for manual testing, automated testing, and test management tools.

## Input

The input comes from the **feature-analyzer** skill:

```yaml
feature:
  name: "User Login"
  module: "Authentication"
  
scenarios:
  positive:
    - "Successful login with valid credentials"
  negative:
    - "Login with invalid password"
  boundary:
    - "Login attempt #5 (account lock threshold)"
    
validation_rules:
  - field: "email"
    rules: ["required", "format:email"]
    
dependencies:
  services: ["Auth Service"]
  test_data:
    valid_user: { email: "test@example.com", password: "Pass123!" }
```

## Scenario Generation Patterns

### Pattern 1: Happy Path (Positive)

For each positive scenario, generate a standard test case:

```markdown
### TC-[MODULE]-[SEQ]: Successful Login with Valid Credentials

| Field | Description |
|-------|-------------|
| **ID** | TC-AUTH-001 |
| **Title** | Successful Login with Valid Credentials |
| **Priority** | P0 |
| **Severity** | Critical |
| **Type** | Functional / Positive |
| **Feature** | User Login |
| **Source** | AC1 - Valid credentials login |

**Preconditions:**
- User is registered with email and password
- User account is ACTIVE
- Login page is accessible

**Test Data:**
| Field | Value |
|-------|-------|
| Email | test@example.com |
| Password | Pass123! |

**Test Steps:**
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to login page | Login page is displayed |
| 2 | Enter valid email in email field | Email is displayed in field |
| 3 | Enter valid password in password field | Password is masked |
| 4 | Click "Login" button | Loading indicator appears |

**Expected Results:**
- User is successfully logged in
- User is redirected to dashboard
- User's name appears in the header
- Session cookie/token is created

**Postconditions:**
- User session is active
- Login audit log entry is created
```

### Pattern 2: Negative/Error Path

```markdown
### TC-AUTH-002: Login with Invalid Password

| Field | Description |
|-------|-------------|
| **ID** | TC-AUTH-002 |
| **Title** | Login with Invalid Password |
| **Priority** | P0 |
| **Type** | Functional / Negative |

**Preconditions:**
- User is registered
- User knows the registered email

**Test Data:**
| Field | Value |
|-------|-------|
| Email | test@example.com |
| Password | WrongPassword999! |

**Test Steps:**
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to login page | Login page is displayed |
| 2 | Enter valid email | Email is displayed |
| 3 | Enter invalid password | Password is masked |
| 4 | Click "Login" button | Error message appears |

**Expected Results:**
- Error message: "Invalid email or password"
- User remains on login page
- Password field is NOT cleared
- Failed attempt counter increments
```

### Pattern 3: Boundary/Edge Case

```markdown
### TC-AUTH-003: Account Lockout on 5th Failed Attempt

| Field | Description |
|-------|-------------|
| **ID** | TC-AUTH-003 |
| **Title** | Account Lockout After Maximum Failed Attempts |
| **Priority** | P1 |
| **Type** | Boundary / Security |

**Preconditions:**
- User is registered
- User has made 4 failed login attempts

**Test Data:**
| Attempt | Email | Password | Expected Result |
|---------|-------|----------|-----------------|
| 1-4 | user@test.com | WrongPass* | "Invalid credentials" |
| 5 | user@test.com | WrongPass* | "Account locked" |

**Test Steps:**
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1-4 | Attempt login 4 times with wrong password | Error shown each time |
| 5 | Attempt login 5th time with wrong password | "Account locked" message |
| 6 | Attempt login with correct password | "Account locked" message |
| 7 | Wait 30 minutes | — |
| 8 | Attempt login with correct password | Login successful |

**Expected Results:**
- After 5th failure: Account locked notification
- During lockout: All attempts rejected
- After 30 min: Account auto-unlocked
- Login successful with correct credentials
```

### Pattern 4: Combinatorial (Multiple Variations)

For features with multiple input variations:

```markdown
### TC-AUTH-004: Email Field Validation

| ID | Email | Expected | Priority |
|----|-------|----------|----------|
| TC-AUTH-004.1 | (empty) | "Email is required" | P0 |
| TC-AUTH-004.2 | invalid-email | "Invalid email format" | P1 |
| TC-AUTH-004.3 | user@ | "Invalid email format" | P1 |
| TC-AUTH-004.4 | @domain.com | "Invalid email format" | P1 |
| TC-AUTH-004.5 | a@b.c | "Invalid email format" | P2 |
| TC-AUTH-004.6 | (50 chars) | Valid | P2 |
| TC-AUTH-004.7 | (101 chars) | "Email too long" | P1 |
| TC-AUTH-004.8 | valid@email.com | Valid | P0 |
```

## Output Formats

### Format 1: Structured Markdown (Default)

The default format, as shown in the patterns above.

### Format 2: Cucumber/Gherkin (BDD)

```gherkin
@functional @login @P0
Feature: User Login
  As a registered user
  I want to log into the system
  So that I can access my account

  Background:
    Given the login page is accessible
    And a registered user exists with email "test@example.com"

  @positive @happy-path
  Scenario: Successful login with valid credentials
    Given I am on the login page
    When I enter "test@example.com" in the email field
    And I enter "Pass123!" in the password field
    And I click the "Login" button
    Then I should be redirected to the dashboard
    And I should see my profile name in the header

  @negative @invalid-input
  Scenario: Login with invalid password
    Given I am on the login page
    When I enter "test@example.com" in the email field
    And I enter "WrongPassword999!" in the password field
    And I click the "Login" button
    Then I should see the error message "Invalid email or password"
    And I should remain on the login page
```

### Format 3: JSON (For Automation/API)

```json
{
  "testCases": [
    {
      "id": "TC-AUTH-001",
      "title": "Successful Login with Valid Credentials",
      "priority": "P0",
      "type": "positive",
      "preconditions": [
        "User is registered with email and password",
        "User account is ACTIVE"
      ],
      "testData": {
        "email": "test@example.com",
        "password": "Pass123!"
      },
      "steps": [
        {
          "step": 1,
          "action": "Navigate to login page",
          "expected": "Login page is displayed"
        },
        {
          "step": 2,
          "action": "Enter valid email",
          "expected": "Email is displayed in field"
        },
        {
          "step": 3,
          "action": "Enter valid password",
          "expected": "Password is masked"
        },
        {
          "step": 4,
          "action": "Click Login button",
          "expected": "User is redirected to dashboard"
        }
      ],
      "tags": ["functional", "login", "P0"]
    }
  ]
}
```

### Format 4: XLSX (Test Management Tools)

In XLSX format, the following columns are generated:

| Column | Description | Example |
|--------|-------------|---------|
| Test Case ID | Unique identifier | TC-AUTH-001 |
| Title | Test case title | Successful Login |
| Feature | Feature name | User Login |
| Module | Module name | Authentication |
| Priority | P0-P3 | P0 |
| Type | Test type | Positive |
| Precondition | Setup steps | User is registered |
| Test Data | Input data | email: test@example.com |
| Step | Step number | 1 |
| Action | Action to perform | Enter email |
| Expected Result | Expected outcome | Email displayed |
| Status | Pass/Fail/Skip/Blocked | (empty) |
| Notes | Additional notes | (empty) |

## Priority Assignment Rules

| Priority | Criteria |
|----------|----------|
| **P0** | Core functionality, security, data loss prevention |
| **P1** | Major feature, common user scenarios, error handling |
| **P2** | Minor features, edge cases, boundary conditions |
| **P3** | UI polish, rare edge cases, cosmetic issues |

## Test Case ID Convention

```
TC-{MODULE_CODE}-{SEQUENCE_NUMBER}
    │            │
    │            └─── Sequential number (001, 002, ...)
    │
    └─── Module abbreviation (AUTH, PAY, ORDER, ...)
```

## Generation Rules

### Rule 1: Coverage Completeness
For each analyzed scenario type, generate:
- At least 1 positive test per AC
- All negative conditions from rules
- Boundary tests for each constraint
- At least 1 security test for auth-related features

### Rule 2: Traceability
Each test case must reference its source:
```
Source: AC1 - Valid credentials login
Source: BR-002 - Account lockout threshold
```

### Rule 3: Deterministic Data
All test data must be concrete, not abstract:
```yaml
# Correct
test_data:
  email: "testuser-001@example.com"
  password: "ValidPass123!"

# Incorrect
test_data:
  email: "valid_email"  # Too abstract
  password: "valid_password"
```

### Rule 4: Atomic Steps
Each step must be a single, verifiable action:
```yaml
# Correct
- Step 1: Enter email
- Step 2: Enter password
- Step 3: Click Login

# Incorrect
- Step 1: Fill in credentials and click login  # Multiple actions
```

## Output Selection

Based on user preference, determine output format:

| User Says | Format |
|-----------|--------|
| "Generate test cases" | Markdown (default) |
| "Export for automation" | JSON |
| "Create BDD scenarios" | Gherkin/Cucumber |
| "Export to Excel" | XLSX |
| "All formats" | Generate all 4 formats |

## Best Practices

1. **One assertion per step** - Each step verifies one thing
2. **Independent test cases** - No inter-case dependencies
3. **Self-describing titles** - Title explains the scenario
4. **Explicit test data** - No ambiguous values
5. **Negative testing** - Always include error scenarios
6. **Boundary coverage** - Test at, below, and above limits
7. **Idempotent tests** - Same data, same result every time
8. **Human-readable** - Non-technical stakeholders can understand
