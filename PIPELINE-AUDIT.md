# Pipeline Audit & Failure Analysis

## Overview
This document provides an audit of the initial, broken GitHub Actions workflow (`.github/workflows/pipeline.yml`) in `multi-job-pipeline-drill`. 
In the initial state, the pipeline exhibits race conditions, isolated filesystem mismatches, unconstrained trigger scopes, lack of safety gates, and absent timeouts.

---

## Detailed Audit by Job

### 1. `lint`
* **Intended Purpose**: Verify code quality, formatting, and syntax rules across the repository before running deeper test suites and build procedures.
* **Current Issues / Failures**:
  * Missing dependencies: Runs without specifying dependencies, but more importantly, no other jobs wait for `lint` to complete. If linting fails, all other jobs continue running, consuming compute and polluting CI signals.
  * Missing execution timeout: No `timeout-minutes` is set; runaway processes or hanging scripts could run until GitHub's default 6-hour limit.
  * Configuration format incompatibility: The repository uses legacy `.eslintrc.json`, which exits with code 2 on modern ESLint v9+ unless legacy mode (`ESLINT_USE_FLAT_CONFIG: 'false'`) is specified.
* **Correct Fix**:
  * Keep `lint` as the entrypoint / root job of the validation sequence.
  * Add `timeout-minutes: 10`.
  * Set `ESLINT_USE_FLAT_CONFIG: 'false'` on the lint step to ensure compatibility with `.eslintrc.json`.

---

### 2. `unit-tests`
* **Intended Purpose**: Run isolated unit tests (`jest src/api.test.js`) against the application logic early in the cycle.
* **Current Issues / Failures**:
  * Runs concurrently with `lint` with no `needs:` declaration.
  * If linting detects fundamental syntax or formatting errors, `unit-tests` still runs unnecessarily.
  * Missing timeout: No `timeout-minutes` set.
* **Correct Fix**:
  * Add `needs: lint` with a comment explaining that unit tests should only run after code quality and syntax checks pass.
  * Add `timeout-minutes: 15`.

---

### 3. `build`
* **Intended Purpose**: Compile or bundle the service code into the distributable `dist/` directory (`mkdir -p dist && cp src/api.js dist/api.js`) and preserve the artifact for downstream jobs.
* **Current Issues / Failures**:
  * Missing `needs:` declaration: Runs concurrently with `lint` rather than waiting for early validation.
  * Missing artifact upload: The `dist/` folder is generated on the ephemeral runner machine and lost as soon as the job finishes. Downstream jobs running on separate virtual machines cannot access this build output.
  * Missing timeout: No `timeout-minutes` set.
* **Correct Fix**:
  * Add `needs: lint` so compilation only takes place on lint-verified code (running in parallel with `unit-tests`).
  * Add `actions/upload-artifact@v4` with `name: app-build` and `path: dist/` to persist the build artifact.
  * Add `timeout-minutes: 20`.

---

### 4. `integration-tests`
* **Intended Purpose**: Validate the built artifact (`dist/api.js`) in an integrated context.
* **Current Issues / Failures**:
  * Missing `needs: build`: The job starts immediately alongside all other jobs instead of waiting for `build` to finish.
  * Missing artifact download: Because GitHub Actions runs each job on an isolated virtual machine, `dist/api.js` does not exist on this runner.
  * Inevitable failure: `npm run test:integration` expects `dist/api.js` to exist and throws `Build output not found.`, failing the job every single run.
  * Missing timeout: No `timeout-minutes` set.
* **Correct Fix**:
  * Add `needs: build` to guarantee the build step completes before integration tests execute.
  * Add `actions/download-artifact@v4` with `name: app-build` and `path: dist` prior to executing `npm run test:integration`.
  * Add `timeout-minutes: 30`.

---

### 5. `deploy-staging`
* **Intended Purpose**: Deploy verified builds to the staging environment for pre-release verification.
* **Current Issues / Failures**:
  * Missing `needs:` declarations: Runs concurrently with test and build jobs, deploying completely unverified code.
  * Missing branch filter (`if:` condition): Executes on any branch push (including experimental feature branches) rather than restricting deployment to the `main` branch.
  * Missing timeout: No `timeout-minutes` set.
* **Correct Fix**:
  * Add `needs: [unit-tests, integration-tests]` to ensure both unit validation and integrated build tests pass before staging deployment.
  * Add conditional execution: `if: github.ref == 'refs/heads/main'`.
  * Add `timeout-minutes: 15`.

---

### 6. `deploy-production`
* **Intended Purpose**: Promote verified staging releases safely to the production environment.
* **Current Issues / Failures**:
  * Missing `needs: deploy-staging`: Can execute before staging deployment, or even before tests run, creating severe release risk.
  * Missing branch filter (`if:` condition): Fires on every branch push, risking production deployments from untested feature branches.
  * Missing timeout: No `timeout-minutes` set.
* **Correct Fix**:
  * Add `needs: deploy-staging` so production release is strictly gated behind successful staging deployment.
  * Add conditional execution: `if: github.ref == 'refs/heads/main'`.
  * Add `timeout-minutes: 15`.

---

### 7. `notify`
* **Intended Purpose**: Send a final notification summarizing pipeline completion or alert on failure.
* **Current Issues / Failures**:
  * Default conditional behavior: Without `if: always()`, this job is skipped if any upstream job fails, meaning failure alerts are never sent.
  * Missing dependency declaration: Without `needs:`, it runs at the start rather than acting as a final status reporter.
  * Missing timeout: No `timeout-minutes` set.
* **Correct Fix**:
  * Add `needs: [lint, unit-tests, build, integration-tests, deploy-staging, deploy-production]`.
  * Add `if: always()` so it runs unconditionally after all jobs complete.
  * Add `timeout-minutes: 10`.
