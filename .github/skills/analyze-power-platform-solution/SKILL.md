---
name: analyze-power-platform-solution
description: Use when analyzing an exported Power Platform solution folder for maintainability issues mapped to the 10 SIG guidelines from the "Beyond Shipping" conference session.
---

# Analyze Power Platform Solution

## Overview

Walks through an unpacked Power Platform solution folder and checks it against the 10 SIG maintainability guidelines translated to Power Platform. Produces a structured per-guideline findings report with ratings and actionable recommendations.

---

## Export and Unpack Steps

The agent MUST:
- Export the Power Platform solution as the single source of truth
- Use the `solution_export` tool from the `power-platform` MCP server
- Use `Expand-Archive` in PowerShell to extract the solution zip file to `/extracted-[solution-name]/`
- Analyze extracted files Power Platform solution by reading the extract XML, JSON and YAML files.
- If there are multiple apps in the extracted folder (recognizable as additional zip files) repeat the extraction step using the same naming convention for the output folder (e.g. `/[app-name]/`) and analyze each app separately.

---

## Expected Folder Structure

```
<SolutionRoot>/
  solution.xml                          — component manifest
  CanvasApps/
    <AppName>/src/
      App.fx.yaml                       — named formulas, app-level settings
      <ScreenName>.fx.yaml              — per-screen controls and formulas
  Workflows/
    <FlowName>-<GUID>.json              — cloud flow definitions
  Entities/
    <TableName>/
      Entity.xml
      BusinessRules/                    — Dataverse business rule XML
  connectionreferences/                 — connector references
  environmentvariabledefinitions/       — env var definitions
  environmentvariablevalues/
```

---

## Analysis: 10 Guidelines

### G1 — Keep Formulas Short

Files: `CanvasApps/*/src/*.fx.yaml`

What to check:
- Read every control property value (OnSelect, OnChange, OnVisible, Items, etc.)
- Flag formulas exceeding 15 lines or 800 characters
- Count semicolons in OnSelect — flag if > 5 (multiple concerns)
- Grep `App.fx.yaml` for `App.Formulas:` block — presence = positive signal
- Grep for `Function(` or UDF definitions — presence = positive signal

Red flags: OnSelect with 5+ semicolon-separated statements, no named formulas in a large app, single property formula > 30 lines.

---

### G2 — Flatten Logic

Files: `CanvasApps/*/src/*.fx.yaml`, `Workflows/*.json`

What to check:
- Canvas: grep for `If(` and count nesting (If inside If) — flag depth > 2
- Canvas: grep for `Switch(` — presence = positive signal
- Flows: parse `actions` JSON structure, look for `"type": "If"` inside another `"type": "If"` — flag depth > 2
- Flows: look for `"type": "Switch"` — positive signal
- Flows: look for `"type": "Scope"` — positive signal (try/catch pattern)

Red flags: nested If depth > 2 in canvas or flows, zero Switch actions in flows with multi-path logic, no Scope actions for error handling.

---

### G3 — Write Code Once (DRY)

Files: `solution.xml`, `Workflows/*.json`, `CanvasApps/*/src/*.fx.yaml`, `environmentvariabledefinitions/`

What to check:
- `solution.xml`: grep for `ComponentType="ComponentLibrary"` — positive signal
- `solution.xml`: grep for `ComponentType="CanvasApp"` and count
- Flows: grep for child flow calls — look for flows calling another flow's HTTP trigger with a solution-aware connection — positive signal
- `environmentvariabledefinitions/`: count files — positive signal if > 0
- Canvas .fx.yaml: grep for `https://` or `http://` hardcoded strings — flag these
- Canvas .fx.yaml: look for identical formula fragments repeated > 3 times across screens

Red flags: no component library, no child flows, no environment variables, hardcoded URLs/GUIDs in flows or canvas, repeated formula blocks across screens.

---

### G4 — Keep Interfaces Small

Files: `CanvasApps/*/src/*.fx.yaml` (component files), `Workflows/*.json`

What to check:
- Canvas component files: grep for `CustomProperties:` and count entries per component — flag > 5
- Flows: find flows with `"type": "Request"` (manual trigger), count `properties.schema.properties` input fields — flag > 5
- Flows: find `"Respond_to_a_PowerApp_or_flow"` action, count output fields — flag > 5

Red flags: component with > 5 custom properties, flow trigger with > 5 input fields, flow response with > 5 output fields.

---

### G5 — Separate Concerns

Files: `solution.xml`, `CanvasApps/*/src/*.fx.yaml`, `Entities/*/BusinessRules/`

What to check:
- `solution.xml`: count `ComponentType="CanvasApp"` vs `ComponentType="AppModule"` (model-driven)
- Per canvas app: count `.fx.yaml` files in `src/` (= screen count) — flag if > 15
- Canvas .fx.yaml: grep for `Launch(` — positive signal (multi-app architecture)
- Canvas .fx.yaml: grep for `Param(` — positive signal (receives context from another app)
- `Entities/*/BusinessRules/`: count XML files — positive signal
- Canvas .fx.yaml: grep for `Patch(` combined with `If(IsBlank(` in same OnSelect — flag (validation in UI layer)

Red flags: single canvas app > 15 screens, no model-driven apps for data-heavy solution, no Launch/Param usage, no Dataverse business rules, validation logic embedded in canvas Patch calls.

---

### G6 — Loose Coupling

Files: `Workflows/*.json`, `CanvasApps/*/src/*.fx.yaml`, `solution.xml`

What to check:
- Flows: categorise trigger types from JSON:
  - `"OpenApiConnectionWebhook"` with Dataverse = event-driven (positive)
  - `"Request"` HTTP trigger = potential tight coupling
  - `"Recurrence"` = scheduled (neutral)
  - `"ApiConnectionNotification"` = connector-based event (positive)
- Canvas .fx.yaml: grep for `http` inside `Office365Outlook`, direct HTTP connector calls, or hardcoded flow trigger URLs
- `solution.xml`: grep for `ComponentType="CustomConnector"` — positive signal

Red flags: all flows are HTTP-triggered (no event-driven), canvas app contains hardcoded flow URL calls, no Dataverse-triggered flows in a Dataverse solution.

---

### G7 — Balanced Architecture

Files: `solution.xml`, `CanvasApps/*/src/`, `Workflows/*.json`

What to check:
- Canvas: count `.fx.yaml` screen files per app, compute min/max/average — flag if max > 3× average
- Flows: parse each flow JSON, count total actions — flag if one flow has 5× more actions than average
- `solution.xml`: list component types and counts — flag extreme skew
- Check if all components are in a single solution file (no segmentation)

Red flags: one app with 3× more screens than others, one flow with 5× more actions than others, all components in one monolithic solution.

---

### G8 — Keep Codebase Small

Files: `solution.xml`, `CanvasApps/*/src/*.fx.yaml`, `Entities/*/BusinessRules/`, `connectionreferences/`

What to check:
- `solution.xml`: count `AppModule` (model-driven app) components — absence in data-heavy solution = flag
- `Entities/*/BusinessRules/`: count XML files — positive signal
- Canvas .fx.yaml: count `Patch(` occurrences — high count in absence of model-driven apps = flag
- `connectionreferences/`: identify custom vs. prebuilt connectors
- Canvas .fx.yaml: total line count across all screen files — proxy for custom code volume

Red flags: zero model-driven apps for a data management solution, many Patch() screens that could be model-driven views/forms, custom connectors where prebuilt equivalents exist.

---

### G9 — Automate Tests

Files: solution root (recursive search), `Workflows/*.json`

What to check:
- Search solution root for `*.testplan.yaml`, `testplan.yaml`, or any YAML with `testSuite:` key — positive signal
- Search for `azure-pipelines.yml` or `.github/workflows/` — positive signal
- Flows: count flows with at least one `"type": "Scope"` action (error handling)
- Flows: look for monitoring/alerting flows (triggered on `Failed` run of another flow)
- Look for solution checker result files (`*.sarif`, `*-checker-results.*`)

Red flags: no test plan YAML files, no CI/CD pipeline config, flows with zero Scope/error-handling actions, no solution checker results.

---

### G10 — Clean Code

Files: `CanvasApps/*/src/*.fx.yaml`, `Workflows/*.json`

What to check:
- Canvas: grep all control names for default patterns: `Button\d+`, `TextInput\d+`, `Label\d+`, `Gallery\d+`, `Icon\d+`, `Image\d+`
  - Calculate %: (default-named controls / total controls) — flag if > 20%
- Canvas: check for prefix convention: `btn`, `txt`, `lbl`, `gal`, `ico`, `img`, `drp`, `sldr` — positive signal
- Flows: in JSON, check each action's `metadata.operationMetadataId` vs. its display name property — flag unchanged/default names like `"Send_an_email_\(V2\)"`
- Flows: check `"status": "Stopped"` at flow level — each = dead code flag
- Flows: count actions with non-empty `description` or `note` fields — positive signal

Red flags: > 20% controls with default names, flow actions using default connector names, disabled (Stopped) flows in the solution, no action descriptions anywhere.

---

## Output Format

```markdown
# Power Platform Maintainability Report
**Solution:** [display name from solution.xml]
**Analyzed:** [date]

## Summary

| # | Guideline | Rating | Key Finding |
|---|-----------|--------|-------------|
| G1 | Keep Formulas Short | 🟢/🟡/🔴 | [one-liner] |
| G2 | Flatten Logic | ... | ... |
| G3 | Write Code Once | ... | ... |
| G4 | Keep Interfaces Small | ... | ... |
| G5 | Separate Concerns | ... | ... |
| G6 | Loose Coupling | ... | ... |
| G7 | Balanced Architecture | ... | ... |
| G8 | Keep Codebase Small | ... | ... |
| G9 | Automate Tests | ... | ... |
| G10 | Clean Code | ... | ... |

---

## G1 — Keep Formulas Short
**Rating:** 🟢/🟡/🔴

**Findings:**
- [specific finding: file path, control name, line count]

**Positive signals:**
- [what the solution does right]

**Recommendation:**
- [concrete next step referencing Power Platform feature]

[... repeat for G2–G10 ...]

---

## Overall Assessment
[2–3 sentences on the solution's maintainability posture, referencing the three SIG principles:
simple guidelines, start now, be proportional.]
```

## Rating Heuristics

| Rating | Meaning |
|--------|---------|
| 🟢 | Follows the guideline; at most isolated minor issues |
| 🟡 | Pattern of issues; some attention needed |
| 🔴 | Systematic violations; refactoring recommended |

Apply proportionality: a few long formulas = 🟡, majority of OnSelects are bloated = 🔴, one unnamed control in an otherwise clean app = 🟢.
