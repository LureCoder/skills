# Feature Analysis Template
# Used by feature-analyzer skill to structure analysis output

feature:
  name: ""
  pageId: ""
  space: ""
  module: ""
  type: ""  # Functional / Non-functional / Technical
  priority: ""

description: ""

acceptance_criteria:
  - id: "AC-001"
    description: ""
    type: "positive"  # positive / negative / boundary

business_rules:
  - id: "BR-001"
    description: ""
    condition: ""
    constraint: ""
    source: ""

validation_rules:
  - field: ""
    rules:
      - type: "required"
      - type: "format"
        value: ""
      - type: "length"
        min: 0
        max: 0
    error_message: ""

scenarios:
  positive:
    - ""
  negative:
    - ""
  boundary:
    - ""
  security:
    - ""
  performance:
    - ""

dependencies:
  external_systems:
    - ""
  preconditions:
    - ""
  data_requirements:
    - ""

test_data_templates:
  scenario_name:
    field: "value"

risk_areas:
  - ""
