# Chapter 12: Helm Tests

## What Are Helm Tests?

Helm tests are Kubernetes resources annotated with `helm.sh/hook: test` that validate a release is working correctly. They are distinct from hooks — tests only run when `helm test` is explicitly invoked, not during `install`, `upgrade`, `delete`, or `rollback`.

**Purpose:** Tests answer the question: "Is this release healthy and functional?" after a deployment completes.

**Key characteristics:**
- Tests are defined inside chart templates alongside application resources.
- Tests are **opt-in** — they never run automatically during lifecycle operations.
- Tests do not block deployment. A release can be "deployed" even if its tests fail.
- Tests return an exit code that CI/CD pipelines can evaluate.

## How Helm Tests Work

When `helm test <release-name>` is invoked:

1. Helm reads the deployed release's manifest and identifies all resources with `helm.sh/hook: test`.
2. Helm creates (or re-creates) those test resources in the release namespace.
3. Helm waits for each test resource to reach a terminal state (`Succeeded` or `Failed` for Pods; `Complete` or `Failed` for Jobs).
4. Helm reports the status of each test.
5. By default, test resources are NOT deleted after the test completes (use delete policies to control cleanup).

### Test Resource Flow

```
helm test myapp
     │
     ▼
┌──────────────────────────────────────┐
│  Identify test-annotated resources   │
│  in the release manifest             │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Create test Pods / Jobs             │
│  Order by helm.sh/hook-weight        │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Wait for test resources             │
│  to reach terminal state             │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Collect results:                    │
│    PASS / FAIL                       │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Exit code: 0 (all pass)             │
│  Exit code: 1 (any fail)             │
└──────────────────────────────────────┘
```

## Test Resource Types

| Resource Type | Terminal States | Recommendation |
|---|---|---|
| **Pod** | `Succeeded`, `Failed` | **Most common.** Simple, direct execution of test commands. Must have `restartPolicy: Never`. |
| **Job** | `Complete`, `Failed` | **Recommended for complex tests.** Better retry handling via `backoffLimit`. Automatic cleanup via `ttlSecondsAfterFinished`. |
| **ConfigMap** | N/A | Do NOT use as test resource. Helm creates it but does not wait. |

**Why Pods are the most common test resource:**
- They are the simplest resource type with a terminal state.
- They map directly to "run a command and check the result."
- They are easy to debug with `kubectl logs`.

**Why Jobs are better for production:**
- `backoffLimit` prevents infinite retry loops.
- `ttlSecondsAfterFinished` auto-cleans the resource.
- `activeDeadlineSeconds` enforces a hard timeout.
- Native Kubernetes retry logic.

## `helm test` Command

### Syntax

```
helm test [RELEASE] [flags]
```

### All Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--filter` / `-f` | `strings` | `[]` | Run only tests matching this regex pattern. Can be specified multiple times for OR matching. |
| `--logs` | `bool` | `false` | Stream test pod logs to stdout during execution. Extremely useful for debugging. |
| `--timeout` | `duration` | `300s` (5m) | Maximum time to wait for all test resources to complete. Supports Go duration format: `30s`, `5m`, `1h`. |
| `--kubeconfig` | `string` | `""` | Path to kubeconfig file. |
| `--kube-context` | `string` | `""` | Name of kubeconfig context to use. |
| `--namespace` / `-n` | `string` | Current namespace | Namespace of the release. |
| `--debug` | `bool` | `false` | Enable verbose output. |
| `--kube-as-user` | `string` | `""` | Username to impersonate for the operation. |
| `--kube-as-group` | `strings` | `[]` | Group to impersonate (may be specified multiple times). |
| `--kube-token` | `string` | `""` | Bearer token for authentication. |
| `--kube-ca-file` | `string` | `""` | CA certificate file for cluster API. |

### Examples

**Run all tests for a release:**

```bash
helm test myapp
```

**Run specific tests by name (regex filter):**

```bash
# Run tests matching "smoke"
helm test myapp --filter smoke

# Run tests matching "smoke" OR "integration"
helm test myapp --filter smoke --filter integration

# Run only tests whose name starts with "db-"
helm test myapp --filter "^db-"
```

**Run tests with live log streaming:**

```bash
helm test myapp --logs
```

This streams pod logs in real time, which is invaluable for seeing what the test is doing and where it fails.

**Run tests with a custom timeout:**

```bash
# Allow up to 10 minutes for long-running tests
helm test myapp --timeout 10m

# Quick timeout for fast smoke tests
helm test myapp --timeout 30s
```

**Run tests in a specific namespace:**

```bash
helm test myapp --namespace staging
```

**Full production command:**

```bash
helm test myapp \
  --namespace production \
  --timeout 5m \
  --logs \
  --filter "smoke|health"
```

## Test Output

### Successful Test Run

```
NAME: myapp
LAST DEPLOYED: Sat Jul 11 15:30:00 2026
NAMESPACE: production
STATUS: deployed
REVISION: 4
TEST SUITE:     myapp-smoke-test
Last Started:   Sat Jul 11 15:35:00 2026
Last Completed: Sat Jul 11 15:35:12 2026
Phase:          Succeeded
TEST SUITE:     myapp-db-connectivity-test
Last Started:   Sat Jul 11 15:35:00 2026
Last Completed: Sat Jul 11 15:35:08 2026
Phase:          Succeeded
```

If all test suites pass, `helm test` returns exit code **0**.

### Failed Test Run

```
NAME: myapp
LAST DEPLOYED: Sat Jul 11 15:30:00 2026
NAMESPACE: production
STATUS: deployed
REVISION: 4
TEST SUITE:     myapp-smoke-test
Last Started:   Sat Jul 11 15:35:00 2026
Last Completed: Sat Jul 11 15:35:15 2026
Phase:          Failed
NOTES:
Smoke test failed. Check logs with:
  kubectl logs myapp-smoke-test -n production
```

If any test suite fails, `helm test` returns exit code **1**.

**Exam Tip:** The release `STATUS` remains `deployed` even when tests fail. Test failure does not change the release state. Only the `helm test` exit code indicates test success or failure.

## Test Pod Lifecycle

```
             ┌─────────────────┐
             │  Test Resource   │
             │    Created       │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  Pod: Running    │
             │  (commands       │
             │   execute)       │
             └────────┬────────┘
                      │
            ┌─────────┴─────────┐
            │                   │
            ▼                   ▼
   ┌─────────────────┐  ┌─────────────────┐
   │  Phase:          │  │  Phase:          │
   │  Succeeded       │  │  Failed          │
   │  (exit code 0)   │  │  (exit code != 0)│
   └────────┬────────┘  └────────┬────────┘
            │                    │
            ▼                    ▼
   ┌─────────────────┐  ┌─────────────────┐
   │  helm test:      │  │  helm test:      │
   │  PASS            │  │  FAIL            │
   └────────┬────────┘  └────────┬────────┘
            │                    │
            ▼                    ▼
   ┌─────────────────┐  ┌─────────────────┐
   │  Delete policy:  │  │  Delete policy:  │
   │  hook-succeeded   │  │  hook-failed     │
   │  (cleaned up)    │  │  (cleaned up)    │
   └─────────────────┘  └─────────────────┘
```

## Test Cleanup with Delete Policies

Test resources use the same delete policy annotations as lifecycle hooks. The behavior is identical:

### Delete Policy Options for Tests

| Policy | When Deletion Occurs |
|---|---|
| `before-hook-creation` | Before creating a new test resource of the same name |
| `hook-succeeded` | After the test succeeds (Pod phase: `Succeeded`) |
| `hook-failed` | After the test fails (Pod phase: `Failed`, or Job failed) |
| Combination (comma-separated) | At each matching trigger |

### Recommended Delete Policy for Tests

```yaml
metadata:
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

**Rationale:**
- `before-hook-creation`: Ensures each `helm test` run starts fresh (cleans up previous test pods).
- `hook-succeeded`: Cleans up pods after successful tests (reduces cluster clutter).
- `hook-failed` is deliberately **not** included: failed test pods should persist for debugging.

### Example with Cleanup

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-health-check
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: health-check
      image: curlimages/curl:8.4.0
      command:
        - sh
        - -c
        - |
          curl -f -s http://{{ .Release.Name }}-service:{{ .Values.service.port }}/health
```

**After a successful test:** the pod is deleted (due to `hook-succeeded`).

**After a failed test:** the pod remains in `Failed` state (no `hook-failed` policy) so you can inspect logs.

## Writing Good Helm Tests

### What to Test

| Test Category | What It Validates | Priority |
|---|---|---|
| **Connectivity** | Can the application accept TCP connections on its service port? | High |
| **Health endpoint** | Does the application's `/health` endpoint return 200? | High |
| **Readiness endpoint** | Does `/ready` or `/healthz` report readiness? | High |
| **API responses** | Do key API endpoints return expected data? | Medium |
| **Database connectivity** | Can the application reach and query its database? | High |
| **Configuration** | Are environment variables, mounted configs, and secrets correct? | Medium |
| **Dependencies** | Can the application reach external services (Redis, Kafka, S3)? | Medium |
| **Data integrity** | Does seed data exist? Are expected fixtures present? | Low |
| **Performance baseline** | Does a simple request return within a threshold? | Low |

### Test Design Principles

1. **Idempotent:** Running the test multiple times produces the same result. No side effects.
2. **Self-contained:** The test does not depend on external state beyond what the chart provides.
3. **Time-bounded:** Every test has a timeout. Use `timeout` command or `activeDeadlineSeconds`.
4. **Read-only:** Tests should not modify application data. They validate, not mutate.
5. **Independent:** Each test should pass or fail independently of other tests.
6. **Diagnosable:** On failure, the test output should clearly indicate what went wrong.

## Test Examples

### Example 1: HTTP Health Check Test

Validates that the deployed application responds to health checks.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-health-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: health-check
      image: curlimages/curl:8.4.0
      command:
        - sh
        - -c
        - |
          set -e
          URL="http://{{ .Release.Name }}-service:{{ .Values.service.port }}"

          echo "=== Health Endpoint Test ==="
          echo "Target: $URL/health"

          STATUS=$(curl -s -o /tmp/response.json -w "%{http_code}" --connect-timeout 5 --max-time 10 $URL/health)
          echo "Status code: $STATUS"

          if [ "$STATUS" != "200" ]; then
            echo "FAIL: Expected 200, got $STATUS"
            cat /tmp/response.json
            exit 1
          fi

          echo "Response body:"
          cat /tmp/response.json

          # Validate JSON structure (optional)
          if command -v jq > /dev/null 2>&1; then
            STATUS_FIELD=$(jq -r '.status // empty' /tmp/response.json)
            if [ "$STATUS_FIELD" != "ok" ]; then
              echo "FAIL: status field is not 'ok'"
              exit 1
            fi
          fi

          echo "PASS: Health endpoint returned 200 OK"
```

### Example 2: Database Connectivity Test

Validates that the application can connect to and query its database.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 120
  activeDeadlineSeconds: 60
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-check
          image: postgres:16-alpine
          env:
            - name: PGHOST
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.existingSecret }}
                  key: host
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.existingSecret }}
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.existingSecret }}
                  key: password
            - name: PGDATABASE
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.existingSecret }}
                  key: database
          command:
            - sh
            - -c
            - |
              set -e

              echo "=== Database Connectivity Test ==="
              echo "Host: $PGHOST"
              echo "Database: $PGDATABASE"
              echo "User: $PGUSER"

              # Test 1: Connectivity
              echo "Test 1: Basic connectivity..."
              if ! pg_isready -t 10; then
                echo "FAIL: Cannot reach database"
                exit 1
              fi
              echo "PASS: Database is reachable"

              # Test 2: Can execute a query
              echo "Test 2: Query execution..."
              RESULT=$(psql -t -c "SELECT 1 AS test_value" 2>&1)
              if echo "$RESULT" | grep -q "1"; then
                echo "PASS: Query executed successfully"
              else
                echo "FAIL: Query failed: $RESULT"
                exit 1
              fi

              # Test 3: Expected tables exist
              echo "Test 3: Schema validation..."
              psql -t -c "\dt {{ .Values.database.schema }}.*" 2>&1 || {
                echo "WARN: Schema check failed (may be expected for fresh installs)"
              }

              echo "=== All database tests passed ==="
```

### Example 3: Configuration Verification Test

Validates that the application's configuration (from ConfigMaps and Secrets) is correct.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-config-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: config-verifier
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          set -e
          echo "=== Configuration Verification Test ==="

          # Verify environment variables
          echo "Test 1: Required environment variables..."
          REQUIRED_VARS="APP_ENV APP_LOG_LEVEL DB_HOST REDIS_HOST"
          for VAR in $REQUIRED_VARS; do
            VALUE=$(printenv "$VAR" 2>/dev/null || echo "")
            if [ -z "$VALUE" ]; then
              echo "FAIL: $VAR is not set"
              exit 1
            fi
            echo "  $VAR = $VALUE"
          done
          echo "PASS: All required variables are set"

          # Verify mounted config file
          echo "Test 2: Config file presence..."
          if [ ! -f /etc/app/config.yaml ]; then
            echo "FAIL: /etc/app/config.yaml not found"
            exit 1
          fi
          echo "PASS: Config file exists at /etc/app/config.yaml"

          # Verify JSON config validity (if applicable)
          if [ -f /etc/app/config.json ]; then
            echo "Test 3: JSON config validity..."
            if command -v jq > /dev/null 2>&1; then
              if jq empty /etc/app/config.json 2>/dev/null; then
                echo "PASS: JSON config is valid"
              else
                echo "FAIL: JSON config is malformed"
                exit 1
              fi
            else
              echo "SKIP: jq not available in test image"
            fi
          fi

          echo "=== Configuration verification passed ==="
      env:
        - name: APP_ENV
          value: {{ .Values.app.env }}
        - name: APP_LOG_LEVEL
          value: {{ .Values.app.logLevel }}
        - name: DB_HOST
          value: {{ .Values.database.host }}
        - name: REDIS_HOST
          value: {{ .Values.redis.host }}
      volumeMounts:
        - name: config
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: {{ .Release.Name }}-config
```

### Example 4: Multi-Endpoint API Validation Test

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-api-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 300
  activeDeadlineSeconds: 120
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: api-tester
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              set -e
              BASE="http://{{ .Release.Name }}-service:{{ .Values.service.port }}"
              FAILURES=0

              run_test() {
                local name=$1
                local method=$2
                local path=$3
                local expected_code=$4
                local expected_body_match=$5

                echo ""
                echo "TEST: $name"
                echo "  $method $BASE$path"
                echo "  Expected status: $expected_code"

                RESPONSE=$(curl -s -w "\n%{http_code}" \
                  -X "$method" \
                  --connect-timeout 5 \
                  --max-time 10 \
                  "$BASE$path" 2>&1)

                HTTP_CODE=$(echo "$RESPONSE" | tail -n1)
                BODY=$(echo "$RESPONSE" | sed '$d')

                echo "  Got status: $HTTP_CODE"

                if [ "$HTTP_CODE" != "$expected_code" ]; then
                  echo "  FAIL: Expected $expected_code, got $HTTP_CODE"
                  echo "  Body: $BODY"
                  FAILURES=$((FAILURES + 1))
                  return 1
                fi

                if [ -n "$expected_body_match" ]; then
                  if echo "$BODY" | grep -q "$expected_body_match"; then
                    echo "  PASS: Body contains '$expected_body_match'"
                  else
                    echo "  FAIL: Body does not contain '$expected_body_match'"
                    FAILURES=$((FAILURES + 1))
                    return 1
                  fi
                fi

                echo "  PASS"
              }

              run_test "Health check"     "GET"  "/health"       200 '"status":"ok"'
              run_test "Readiness"        "GET"  "/ready"        200
              run_test "API version"      "GET"  "/api/v1/version" 200
              run_test "Metrics"          "GET"  "/metrics"      200
              run_test "Not found 404"    "GET"  "/nonexistent"  404

              echo ""
              if [ "$FAILURES" -gt 0 ]; then
                echo "FAIL: $FAILURES test(s) failed"
                exit 1
              fi
              echo "PASS: All API tests passed"
```

## Debugging Tests

### Using `--logs` Flag

The `--logs` flag is the fastest way to see what's happening in tests:

```bash
helm test myapp --logs --timeout 5m
```

This streams pod output directly to your terminal, allowing real-time debugging.

### Inspecting Test Pods

```bash
# List test pods for a release
kubectl get pods -n <namespace> -l helm.sh/chart=<chart-name> \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,TEST:.metadata.annotations.helm\.sh/hook

# Describe a failed test pod
kubectl describe pod <test-pod-name> -n <namespace>

# Get detailed pod status (including container exit codes)
kubectl get pod <test-pod-name> -n <namespace> -o json | \
  jq '.status.containerStatuses[] | {name: .name, state: .state, restartCount: .restartCount}'
```

### Analyzing Test Output

```bash
# View logs of a completed test pod
kubectl logs <test-pod-name> -n <namespace>

# View logs for a specific container (if multiple)
kubectl logs <test-pod-name> -n <namespace> -c <container-name>

# View previous container logs (if pod restarted)
kubectl logs <test-pod-name> -n <namespace> --previous
```

### Common Test Failure Diagnosis

| Symptom | Likely Cause | Debugging Command |
|---|---|---|
| Pod stuck in `Pending` | Insufficient cluster resources, missing PVC, or node affinity that can't be satisfied | `kubectl describe pod <name>` + check `Events` section |
| Pod stuck in `ContainerCreating` | Image pull failure or init container problem | `kubectl describe pod <name>` + check `Events` for `ErrImagePull` |
| `CrashLoopBackOff` | Test command exits non-zero immediately | `kubectl logs <pod-name>` to see error output |
| `Error` phase | Container exited with error | `kubectl logs <pod-name>` |
| `Unknown` phase | Pod was running but kubelet lost contact | Check node health, network connectivity |
| Test times out (> `--timeout`) | Test hanging on an operation (e.g., waiting for DB that isn't reachable) | Add `timeout` wrapper in test command; verify network connectivity |

### Debugging a Failing Test Step by Step

```bash
# 1. Find the failed test pod
kubectl get pods -n production | grep test

# 2. Check its status and events
kubectl describe pod myapp-health-test -n production

# 3. Read its logs for error messages
kubectl logs myapp-health-test -n production

# 4. If the pod already terminated, check its exit code
kubectl get pod myapp-health-test -n production \
  -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'

# 5. Re-run the test interactively (override command)
kubectl run debug-test --rm -it --restart=Never \
  --image=curlimages/curl:8.4.0 -- sh
# Then manually execute curl commands to the application service
```

## Integrating `helm test` in CI/CD

### Exit Codes

| Exit Code | Meaning |
|---|---|
| `0` | All test suites passed |
| `1` | At least one test suite failed |

This makes `helm test` directly usable in CI/CD conditionals.

### GitHub Actions Integration

```yaml
name: Deploy and Test

on:
  push:
    branches: [main]

jobs:
  deploy-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}

      - name: Install Helm chart
        run: |
          helm upgrade --install myapp ./chart \
            --namespace staging \
            --create-namespace \
            --wait \
            --timeout 10m \
            --values values/staging.yaml

      - name: Run Helm tests
        run: |
          helm test myapp \
            --namespace staging \
            --timeout 5m \
            --logs
        # Exit code 0 = pass, 1 = fail
        # GitHub Actions will mark the step as failed on exit code 1

      - name: On test failure — rollback
        if: failure()
        run: |
          echo "Tests failed! Rolling back..."
          helm rollback myapp --namespace staging

      - name: On test success — promote to production
        if: success()
        run: |
          echo "Tests passed! Ready for production promotion."
```

### GitLab CI Integration

```yaml
deploy-and-test:
  stage: deploy
  image: alpine/helm:3.13.0
  before_script:
    - apk add --no-cache curl
    - mkdir -p ~/.kube && echo "$KUBECONFIG" > ~/.kube/config
  script:
    - helm upgrade --install myapp ./chart
      --namespace staging
      --create-namespace
      --wait
      --timeout 10m
      --values values/staging.yaml
    - helm test myapp --namespace staging --timeout 5m --logs
  after_script:
    - |
      if [ "$CI_JOB_STATUS" != "success" ]; then
        echo "Deploy or test failed! Triggering rollback..."
        helm rollback myapp --namespace staging
      fi
  only:
    - main
```

### Scripting with Exit Code Checks

```bash
#!/usr/bin/env bash
set -euo pipefail

# Deploy
helm upgrade --install myapp ./chart \
  --namespace prod \
  --wait --timeout 10m

# Run tests
if helm test myapp --namespace prod --timeout 5m --logs; then
  echo "All tests passed. Deployment successful."
else
  TEST_EXIT=$?
  echo "Tests failed with exit code $TEST_EXIT"
  echo "Rolling back to previous revision..."
  helm rollback myapp --namespace prod
  exit $TEST_EXIT
fi
```

**Production Note:** Always run `helm test` **after** `--wait` in CI/CD. If you don't wait for the deployment to be ready, tests will likely fail because pods haven't started yet.

## Production Validation Patterns

### 1. Smoke Tests

Lightweight tests run immediately after deployment to verify basic functionality.

```bash
# Run only smoke tests
helm test myapp --filter smoke --timeout 2m --logs
```

### 2. Acceptance Tests

More comprehensive tests run against staging before production promotion.

```bash
# Run full test suite against staging
helm test myapp --namespace staging --timeout 10m --logs
```

### 3. Canary Validation

Test a canary deployment before shifting traffic.

```bash
# Deploy canary
helm upgrade --install myapp-canary ./chart \
  --namespace prod \
  --set replicaCount=1 \
  --set canary.enabled=true

# Test canary
if helm test myapp-canary --namespace prod --timeout 3m --logs; then
  echo "Canary validated. Promoting to full deployment..."
  helm upgrade --install myapp ./chart --namespace prod --set replicaCount=10
else
  echo "Canary failed. Deleting canary..."
  helm uninstall myapp-canary --namespace prod
fi
```

### 4. Pre-Production Gate

Use `helm test` as a gating mechanism in CI/CD before allowing production deployment:

```yaml
# In a deployment pipeline
stages:
  - deploy-staging
  - test-staging   # <-- gate
  - deploy-prod

test-staging:
  stage: test-staging
  script:
    - helm test myapp --namespace staging --timeout 5m --logs
  # If this fails, the pipeline stops; production is not deployed
```

## Test Failure Handling

### What Happens When a Test Fails

1. The test pod enters `Failed` phase.
2. `helm test` prints the failure and returns exit code `1`.
3. **The release status is NOT changed.** The release remains `deployed`.
4. **No automatic rollback occurs.** You must roll back manually or via CI/CD logic.

### Re-running Tests

Tests can be re-run at any time — they are idempotent:

```bash
# Fix the issue, then re-run tests
helm test myapp --namespace prod --timeout 5m --logs

# Re-run only failed tests
helm test myapp --namespace prod --filter "^failed-test-name" --logs
```

### Cleaning Up After Failed Tests

Failed tests persist unless you have a `hook-failed` delete policy. To manually clean up:

```bash
# Delete all test pods for a release
kubectl delete pod -n <namespace> -l helm.sh/chart=<chart-name> \
  --field-selector metadata.annotations.helm\.sh/hook=test

# Or delete by name
kubectl delete pod <test-pod-name> -n <namespace>
```

## Best Practices

### 1. Tests Must Be Idempotent

A test run multiple times should produce the same result. Avoid:
- Writing to application databases during tests.
- Creating resources that persist beyond the test.
- Depending on mutable external state.

**Good:**
```bash
curl -s http://app-service:8080/health
```

**Bad:**
```bash
curl -X POST http://app-service:8080/api/users -d '{"name":"test-user"}'
curl -s http://app-service:8080/api/users  # depends on previous POST
```

### 2. Always Set Timeouts

Every test must be bounded in time. Unbounded tests can hang CI/CD pipelines.

```yaml
# Pod-level timeout (in test script)
command:
  - sh
  - -c
  - timeout 120 /app/run-tests.sh || { echo "Timed out"; exit 1; }

# Job-level timeout
spec:
  activeDeadlineSeconds: 120   # Hard timeout at Kubernetes level
```

### 3. Clean Up After Tests

Use delete policies to prevent test pod accumulation:

```yaml
metadata:
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

For Jobs, also use `ttlSecondsAfterFinished`:

```yaml
spec:
  ttlSecondsAfterFinished: 300   # Auto-delete 5 minutes after completion
```

### 4. Use Meaningful Test Names

Test names should be descriptive and consistent:

```yaml
# Good
metadata:
  name: {{ .Release.Name }}-health-check
  name: {{ .Release.Name }}-db-connectivity
  name: {{ .Release.Name }}-api-smoke

# Bad
metadata:
  name: {{ .Release.Name }}-test1
  name: {{ .Release.Name }}-test2
  name: {{ .Release.Name }}-foo
```

### 5. Test One Thing Per Resource

Each test Pod or Job should validate a single concern. This makes debugging straightforward:
- One Pod = one type of validation.
- If it fails, you know exactly what broke.
- Use `helm.sh/hook-weight` to order multi-step test sequences.

### 6. Provide Clear Failure Messages

Test output must clearly indicate what failed:

```bash
# Good
echo "FAIL: Expected status 200, got $STATUS. Response: $RESPONSE"
exit 1

# Bad
exit 1   # No indication of what went wrong
```

### 7. Use `--wait` Before `helm test`

In CI/CD, always sequence:

```bash
helm upgrade --install myapp ./chart --wait --timeout 10m
helm test myapp --timeout 5m --logs
```

Without `--wait`, tests may run before pods are ready, causing false failures.

### 8. Version-Lock Test Images

Pin specific image tags for test containers to ensure deterministic behavior:

```yaml
# Good
image: curlimages/curl:8.4.0

# Bad
image: curlimages/curl:latest   # May change behavior between runs
```

### 9. Keep Tests Lightweight

Tests should validate, not perform heavy computation. Use small images:
- `busybox:1.36` (~4 MB) for simple script checks
- `curlimages/curl:8.4.0` (~11 MB) for HTTP tests
- `postgres:16-alpine` (~250 MB) for database checks

Avoid full OS images like `ubuntu:22.04` (~78 MB) unless absolutely necessary.

### 10. Document Test Invocation

Include test instructions in the chart's `NOTES.txt`:

```txt
Deployment complete! To verify:

  helm test {{ .Release.Name }} --namespace {{ .Release.Namespace }} --logs
```

This ensures all operators know how to validate the deployment.
