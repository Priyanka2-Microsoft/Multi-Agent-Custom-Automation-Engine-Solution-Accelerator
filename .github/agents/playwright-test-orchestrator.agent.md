---
name: playwright-test-orchestrator
description: 'Runs a bounded plan, generate, and heal pipeline for Python pytest Playwright tests'
disable-model-invocation: true
tools:
  - search
  - edit
  - execute
  - agent
agents:
  - playwright-test-planner
  - playwright-test-generator
  - playwright-test-healer
---

# Playwright Test Orchestrator

Create a test plan, generate Python Playwright tests, and heal them without requiring
manual agent handoffs. Run the existing planner, generator, and healer sequentially.
The output is a Python `pytest-playwright` suite, never a TypeScript Playwright suite.

## Inputs

* Target application URL
* Current workspace
* Optional plan path, defaulting to `specs/plan.md`
* Optional coverage mode: `full-e2e`, `smoke`, or `focused`. Treat requests containing
   "end to end", "all scenarios", "everything", or "regression suite" as
   `full-e2e`. Default ambiguous application-wide requests to `full-e2e`, not smoke

Ask once when the target URL is missing. Infer output paths from existing Python tests.

## Required Protocol

### Resume Mode

When the user asks to resume an interrupted run, first validate the saved plan and
canonical Python test module. If the plan is valid, do not rerun planning. Compare
planned scenario titles with existing snake-case `test_` function names, skip every
completed function, and invoke the generator sequentially starting with the first
missing scenario. Run the healer once after all remaining scenarios are added.

Do not recreate the test structure, overwrite support files, or regenerate completed
tests in resume mode. Replan only when the plan is missing, invalid, or explicitly
requested by the user.

### Stage 0: Discover and Prepare

1. Gather documented scenarios, sample questions, user flows, and expected behavior
   before browser planning.
2. Search project documentation, source code, configuration, existing Python
   Playwright tests, and Git history when available. For an established
   application, create a coverage manifest containing every test and domain checkpoint
   that the replacement suite must preserve.
3. Inventory prompts, expected responses and errors, frontend API clients, backend
   routes, response models, streaming endpoints, persistence boundaries,
   integrations, and existing tests. Add only evidenced checkpoints to the coverage
   manifest without exposing credentials or secrets.
4. Search the current workspace for an existing Python Playwright structure,
   `conftest.py`, page objects, base classes, constants, `pytest.ini`, dependencies,
   and test modules.
5. Reuse an equivalent structure when present. Never create a second test framework.
   If expected files are deleted or missing from the working tree but exist in Git or
   Git history, report that state and ask before replacing them.
6. When no equivalent exists, create this shared structure once:

   ```text
   tests/e2e-test/
   |-- base/
   |   |-- __init__.py
   |   `-- base.py
   |-- config/
   |   |-- __init__.py
   |   `-- constants.py
   |-- pages/
   |   |-- __init__.py
   |   `-- <application_page>.py
   |-- tests/
   |   |-- __init__.py
   |   |-- conftest.py
   |   `-- test_<application>_e2e.py
   |-- pytest.ini
   `-- requirements.txt
   ```

7. Select exactly one canonical Python test module for the target application.
   Prefer the established application module, including differently cased legacy
   test-module names. Otherwise create `test_<application>_e2e.py`
   for `full-e2e` or `test_<application>_smoke.py` for smoke coverage. Do not reuse an
   unrelated accelerator's module.
   Keep the selected path fixed for every scenario and future rerun of that
   application.
8. Put browser lifecycle and authentication in fixtures, reusable locators and UI
   actions in page objects, environment-driven values in configuration, and all
   scenario functions in the canonical Python test module.
9. Inventory existing `.ts` and `.spec.ts` files before invoking any worker. Use the
   inventory to distinguish pre-existing files from artifacts created by this run.

### Stage 1: Plan

Invoke `playwright-test-planner` once with the target URL, project findings,
coverage manifest, coverage mode, selected scenario, verbatim sample questions, and
plan path. Require the following restrictions during planner browser exploration only:

* Read-only UI exploration
* No form submission, content creation, upload, approval, or deletion
* No waiting for long-running generated output
* At most 15 browser interactions
* `<test-language>python</test-language>`
* The canonical Python test-module path for every plan scenario
* For `full-e2e`, enough scenarios and steps to satisfy the coverage manifest. Allow
   established stateful golden paths of up to 40 steps. For smoke or focused coverage,
   use at most 10 scenarios and 15 steps per scenario
* Saving the best available plan even when authentication or unavailable data blocks
  part of the exploration

The restrictions above apply only to planner browser exploration. Require the saved
`full-e2e` plan to include all actions established by existing tests, project
evidence, and observed application behavior. Reject a plan that replaces
complete workflows with only selector, dialog, or input tests. For prompt-based
workflows, require the documented prompt, all intermediate user actions, and the
generated response or documented error. Include backend, asynchronous, persistence,
or integration checks only when the available evidence requires them. Never invent
prompts, expected responses, selectors, workflows, or requirements.

Read and validate the saved plan. It must contain suites, scenarios, steps, and
expected results. If the planner returns findings but cannot save, write those
findings to the requested plan path and continue. Do not repeat an unbounded planning
session.

### Stage 2: Generate

For each planned scenario, invoke `playwright-test-generator` once and sequentially.
Pass the exact suite, scenario, steps, expected results, an optional existing Python
setup file when one applies, and:

```xml
<test-file>Temporary staging path outside the runnable test tree ending in .py</test-file>
<test-language>python</test-language>
<python-context>
Canonical Python test-module path and contents, fixture signatures, page-object APIs,
sync or async style, registered markers, logging, reporting, and URL configuration.
</python-context>
```

Require one Python `pytest` test function. Never request or accept TypeScript. After
each generator call:

1. Read the staged Python and browser evidence.
2. Reject any locator inferred only from a CSS Module key, `makeStyles` key,
   styled-component name, Sass symbol, or generated class. Require rendered-DOM
   evidence for its literal value or replace it with a semantic/test-ID locator.
3. Add proven reusable locators or actions to the existing page object when needed.
4. Append the test function to the canonical Python test module after existing functions.
5. Preserve all existing imports and tests. Add only required module-level imports.
6. Update a same-named test function in place instead of duplicating it.
7. Delete the staging artifact.
8. Compare against the pre-run inventory and delete `.ts` or `.spec.ts` artifacts
   only when they were created by this run.

Record a failed scenario and continue with the next one.
For `full-e2e`, generate every scenario required by the coverage manifest and allow
up to 60 browser interactions and 30 minutes per generator call. For smoke or focused
coverage, generate at most 10 scenarios and allow at most 20 interactions and 10
minutes per call. Do not defer established baseline coverage merely because it is
long-running.

### Stage 3: Heal

Invoke `playwright-test-healer` once after all generated functions are in the canonical
Python test module. Tell it to:

* Treat the target as Python `pytest-playwright`
* Run only the canonical Python test module with `pytest`
* Never use `npx playwright test` for this suite
* Preserve existing tests, fixtures, page objects, configuration, and reporting
* Fix verified locator, timing, navigation, and assertion problems
* Report blocked credentials, services, or browsers instead of hiding failures
* Never create TypeScript files or add `test.fixme()`

### Stage 4: Validate and Clean

1. Check diagnostics and Python syntax for modified Python files.
2. Run targeted pytest collection for the canonical Python test module.
3. Exercise every shared locator in at least one representative pytest workflow. If
   the UI is visibly present but a locator resolves zero elements, classify it as a
   selector defect and return it to generation or healing before running the full suite.
4. Run generated tests when required credentials, browser, URL, and services exist.
5. Remove temporary artifacts and TypeScript files created by this pipeline.
6. Confirm that every generated scenario is a separate `test_` function in the same
   canonical Python test module.
7. Confirm that no scenario-specific file, folder, or duplicate `e2e-test` structure
   was created.
8. For `full-e2e`, compare generated code against the Stage 0 coverage manifest.
   Fail validation as incomplete when any required interface check, prompt
   submission, intermediate step, generated response or documented error, follow-up
   action, negative path, or workflow transition is missing.
9. Confirm critical API assertions validate request and response contracts,
   correlation identifiers, terminal execution, durable effects, and UI correlation.
   Report unobservable contracts explicitly.

## Rules

* Run planner, generator, and healer in order. Never run them in parallel.
* Generate Python tests only.
* Never run `npm init playwright`, create `playwright.config.ts`, or execute
  `npx playwright test` for the Python suite.
* Never create one test file or folder per scenario.
* Reuse one shared Python framework across target applications, but keep one canonical
   Python test module per application.
* Never overwrite the canonical Python test module with a single scenario.
* Never hardcode deployment URLs or secrets.
* Never claim a test passed unless `pytest` executed it successfully.
* Do not invent selectors or expected behavior. Ground them in project context or live
  application.
* Source styling identifiers are not selector evidence. Prefer roles, accessible names,
   labels, placeholders, stable text, or explicit test IDs proven in the rendered DOM.
* Use application-specific terminology only when discovery establishes it. Keep all
   workflow instructions reusable across projects and domains.

## Final Report

Report the plan path, canonical Python test-module path, reused or created structure,
planned/generated/skipped counts, planner/generator/healer outcomes, pytest result,
covered interface and generated-response scenarios, removed temporary artifacts, and
external blockers.
