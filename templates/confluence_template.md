| **Name of Feature** | {TICKET_ID} {FEATURE_TITLE} |
| --- | --- |
| **NPLAN & ENG Epic** | {NPLAN_ENG_EPIC} |
| **QE Epic Link** | {QE_EPIC_LINK} |
| **Product requirement doc** | {PRODUCT_REQUIREMENT_DOC} |
| **Feature design doc** | {FEATURE_DESIGN_DOC} |
| **Release Schedule** | {RELEASE_SCHEDULE} |
| **QE Members** | {QE_MEMBERS} |
| **Dev Members** | {DEV_MEMBERS} |
| **TestPlan Review by QE & Dev & PM** | TBD |
| **Test Rail Link** | {TEST_RAIL_LINK} |
| **Testing Estimates ( in days or weeks )**  \n**Manual + Automation** | {TEST_ESTIMATE} |
| **Feature TOI link** | {FEATURE_TOI_LINK} |
| **Version \[Version to be updated each time Testplan is updated \]** | {VERSION} |

## Test Scope

The scope is to give the complete test strategy for the delivery of a feature. All project members will have a clear understanding about what is tested and what is not.

{SCOPE_NARRATIVE}

| **In test scope** | **Not in test scope** |
| --- | --- |
| {IN_SCOPE_BULLETS} | {NOT_IN_SCOPE_BULLETS} |

## Setup diagram

{SETUP_DIAGRAM}

## Test Requirements

{STACK_ACCESS_NOTES}

| **Requirements** | **Availability Status** |
| --- | --- |
{TEST_REQUIREMENTS_TABLE_ROWS}

## Feature Flags

List control flags, feature flags, group_vars, provisioner flags, USRA flags if applicable

| **Type** | **Flag Name** | **Default Status** |
| --- | --- | --- |
{FEATURE_FLAGS_TABLE_ROWS}

## Dependency features and Regression Impact

{DEPENDENCY_FEATURES_AND_REGRESSION_IMPACT}

## Acceptance Criteria

{ACCEPTANCE_CRITERIA_BULLETS}

{RESILIENCY_SECTION}

## Test Cases

Full detail (Steps + Expected Results) is in the TestRail CSV: `{TICKET_ID}_{FEATURE_NAME}_testrail.csv`

| **S. No** | **Section** | **Test Categories(Type)** | **Service/Component** | **Test Summary** | **Steps** | **Expected Result** | **Priority(P0/P1/P2/P3)** | **Automatable (Yes/No)** | **Automated (Yes/No)** | **UI Case (Yes/No)** | **Arrived by QE (Yes/No)** | **Suggested by Dev (Yes/No)** | **Derived by AI (Yes/No)** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
{TEST_CASES_TABLE_ROWS}

## Detailed Information about Test Categories

{DETAILED_TEST_CATEGORY_NOTES}

## Actionable Items

{ACTIONABLE_ITEMS_CHECKLIST}

**TOI Links**

{TOI_LINKS}

**Key Bugs/Tasks which needs special attention**

{KEY_BUGS_TASKS}
