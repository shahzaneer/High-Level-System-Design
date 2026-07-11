# Chapter 22: Practice Scenarios

This chapter provides 105 hands-on exercises covering all aspects of Helm, from basic chart installation to production-grade deployment patterns. Each exercise includes a scenario description, environment context, task definition, and expected solution with explanation.

---

## CATEGORY A: Chart Installation (16 Exercises)

### Exercise A-1: Default Installation

**Scenario:** You are deploying the `nginx` chart from the Bitnami repository for the first time. You want the simplest possible installation with all default values.

**Environment:**
- Bitnami repository already added as `bitnami`
- Kubernetes cluster running and `kubectl` configured
- Helm v3 installed

**Task:** Install the `nginx` Helm chart from the `bitnami` repository with the release name `my-nginx` and all default values.

**Solution:**
```bash
helm install my-nginx bitnami/nginx
```
This installs the `bitnami/nginx` chart under the release name `my-nginx` using every default value defined in the chart's `values.yaml`. The release is created in the current kubectl namespace.

---

### Exercise A-2: Install with `--set`

**Scenario:** You want to deploy WordPress from the Bitnami repository, but you need to override the WordPress admin username on the command line.

**Environment:**
- Bitnami repository added as `bitnami`
- You need to set the WordPress username to `superadmin`

**Task:** Install `bitnami/wordpress` with release name `my-blog`, overriding `wordpressUsername` to `superadmin` using `--set`.

**Solution:**
```bash
helm install my-blog bitnami/wordpress --set wordpressUsername=superadmin
```
The `--set` flag overrides `wordpressUsername` in the chart's values. Helm merges this single override with all other default values before rendering templates.

---

### Exercise A-3: Install with a Values File

**Scenario:** Your team maintains a custom values file for the staging environment. You need to deploy the `mysql` chart using that file.

**Environment:**
- File `/home/user/helm-values/staging-mysql.yaml` exists
- Bitnami repository available

**Task:** Install `bitnami/mysql` as release `staging-db` using the values file at `/home/user/helm-values/staging-mysql.yaml`.

**Solution:**
```bash
helm install staging-db bitnami/mysql --values /home/user/helm-values/staging-mysql.yaml
```
The `--values` (or `-f`) flag loads key-value overrides from a YAML file. The file is merged with the chart's default `values.yaml`, with the user's file taking precedence.

---

### Exercise A-4: Install with `--set-string` for Number-as-String

**Scenario:** A chart expects a configuration value as a string, but the value looks like a number (e.g., a port or a version string). Using `--set` would cause Helm to interpret it as a number, breaking the template.

**Environment:**
- A custom chart `myorg/config-app` is in a local repository
- The chart requires `app.configVersion` as a string value `"2.0"`

**Task:** Install `myorg/config-app` as release `config-svc`, overriding `app.configVersion` to the string `"2.0"`, ensuring it is not coerced into a float.

**Solution:**
```bash
helm install config-svc myorg/config-app --set-string app.configVersion=2.0
```
`--set-string` forces the value to be treated as a string literal even when it looks numeric. Without it, `--set app.configVersion=2.0` would produce the float `2` in the values map, which could break YAML type expectations.

---

### Exercise A-5: Install with `--set-json`

**Scenario:** A chart requires a complex nested value (a list of objects) that cannot be easily expressed with dot-notation `--set`. You decide to use `--set-json` to supply a JSON literal.

**Environment:**
- Chart `myorg/microservices` expects `services[].name` and `services[].port`
- You need to define three services

**Task:** Install `myorg/microservices` as release `ms-platform`, providing the services list as JSON.

**Solution:**
```bash
helm install ms-platform myorg/microservices \
  --set-json 'services=[{"name":"auth","port":8080},{"name":"api","port":8081},{"name":"worker","port":8082}]'
```
`--set-json` accepts a JSON string and deserialises it into the values map. This is the cleanest way to pass lists, maps, and nested structures from the command line.

---

### Exercise A-6: Install with Multiple Values Files

**Scenario:** Your organisation uses a hierarchy of values files: a base file with common settings and an environment-specific file with overrides. You need to apply both during installation.

**Environment:**
- `base-values.yaml` contains organisation-wide defaults
- `prod-values.yaml` contains production-specific overrides (higher priority)

**Task:** Install `bitnami/redis` as release `cache-prod`, loading `base-values.yaml` first, then `prod-values.yaml`.

**Solution:**
```bash
helm install cache-prod bitnami/redis \
  -f base-values.yaml \
  -f prod-values.yaml
```
Helm loads values files in the order specified. Later files override keys from earlier files. The final merged result is used for rendering.

---

### Exercise A-7: Install a Specific Chart Version

**Scenario:** The latest version of the `prometheus` chart introduces breaking changes. You need to install version `15.10.2` which your team has validated.

**Environment:**
- Prometheus community repository added as `prometheus-community`

**Task:** Install `prometheus-community/prometheus` version `15.10.2` as release `monitoring`.

**Solution:**
```bash
helm install monitoring prometheus-community/prometheus --version 15.10.2
```
The `--version` flag pins the chart to a specific version. Helm fetches the exact chart version from the repository index, avoiding unintended upgrades.

---

### Exercise A-8: Install with `--generate-name`

**Scenario:** You are running an automated test that creates temporary releases. You do not care about the release name and want Helm to generate a unique one to avoid collisions.

**Environment:**
- A CI pipeline deploying ephemeral test instances
- The chart `testing/e2e-app` is available

**Task:** Install `testing/e2e-app` with an auto-generated release name.

**Solution:**
```bash
helm install --generate-name testing/e2e-app
```
`--generate-name` appends a random suffix to the chart name to produce a unique release name, e.g., `e2e-app-1689273647`. This prevents name collisions in automated environments.

---

### Exercise A-9: Install with Custom Release Name

**Scenario:** You want to deploy Redis under a descriptive name that matches your service naming convention.

**Environment:**
- Bitnami repository available

**Task:** Install `bitnami/redis` with the release name `session-store-prod`.

**Solution:**
```bash
helm install session-store-prod bitnami/redis
```
The first positional argument after `install` is the release name. Helm tracks the release by this name for all future operations (upgrade, rollback, uninstall).

---

### Exercise A-10: Install into a Namespace That Does Not Exist

**Scenario:** You are deploying a new microservice and want it in a dedicated namespace `payment-svc` that currently does not exist.

**Environment:**
- Namespace `payment-svc` does not exist in the cluster
- Chart `myorg/payment-api` is available

**Task:** Install `myorg/payment-api` as release `payment` into namespace `payment-svc`, creating the namespace automatically.

**Solution:**
```bash
helm install payment myorg/payment-api --namespace payment-svc --create-namespace
```
The `--create-namespace` flag tells Helm to create the target namespace if it does not already exist. Without this flag, installing into a non-existent namespace produces an error.

---

### Exercise A-11: Install with `--atomic`

**Scenario:** You are deploying a critical service and want Helm to automatically clean up all created resources if the installation fails at any point.

**Environment:**
- Chart `myorg/payment-api` is available
- Some dependencies may be unavailable, causing the deployment to fail

**Task:** Install `myorg/payment-api` as release `payment` with the guarantee that no partial resources are left behind on failure.

**Solution:**
```bash
helm install payment myorg/payment-api --atomic
```
With `--atomic`, if the installation fails (any hook fails or the release does not reach the expected state), Helm automatically deletes all the resources it created. This prevents orphaned Kubernetes objects.

---

### Exercise A-12: Install with `--wait` and `--timeout`

**Scenario:** You are deploying a StatefulSet that takes a long time to become ready. The default 5-minute timeout is insufficient.

**Environment:**
- Chart `myorg/big-data-db` deploys a StatefulSet that needs up to 10 minutes to initialise
- The deployment must block until ready

**Task:** Install `myorg/big-data-db` as release `analytics-db`, waiting for all resources to be ready with a 15-minute timeout.

**Solution:**
```bash
helm install analytics-db myorg/big-data-db --wait --timeout 15m
```
`--wait` blocks until all Pods, PVCs, Services, and deployments reach a ready state (subject to the number of replicas). `--timeout 15m` extends the wait limit from the default 5 minutes to 15 minutes.

---

### Exercise A-13: Install from a Local Chart Directory

**Scenario:** You are developing a chart locally and want to test it without packaging or pushing to a repository.

**Environment:**
- Chart source at `/home/user/projects/myapp-chart/` with `Chart.yaml`, `values.yaml`, and `templates/`

**Task:** Install the local chart as release `myapp-dev` directly from the directory.

**Solution:**
```bash
helm install myapp-dev /home/user/projects/myapp-chart/
```
Helm accepts a path to a chart directory. It reads the `Chart.yaml` and `values.yaml` from the directory and renders all templates in the `templates/` folder. This is the fastest workflow for chart development.

---

### Exercise A-14: Install from a Packaged `.tgz` Archive

**Scenario:** The CI pipeline has packaged a chart into a `.tgz` archive and stored it in an artifact repository. You need to install from that archive.

**Environment:**
- Packaged chart at `/opt/charts/myapp-1.2.3.tgz`

**Task:** Install the packaged chart as release `myapp-staging` directly from the `.tgz` file.

**Solution:**
```bash
helm install myapp-staging /opt/charts/myapp-1.2.3.tgz
```
Helm supports installing directly from a `.tgz` chart archive. The archive must contain `Chart.yaml`, `values.yaml`, and all template files at the archive root. Helm decompresses and installs the chart.

---

### Exercise A-15: Install from an OCI Registry

**Scenario:** Your team stores Helm charts in an OCI-compliant container registry (e.g., Docker Hub, AWS ECR, or Harbor). You need to deploy a chart from there.

**Environment:**
- OCI chart is pushed to `oci://registry.example.com/charts/myapp` with tag `1.5.0`
- You have authenticated via `helm registry login`

**Task:** Install the OCI-hosted chart as release `myapp-prod`.

**Solution:**
```bash
helm install myapp-prod oci://registry.example.com/charts/myapp --version 1.5.0
```
Helm v3.8+ natively supports OCI registries. The `oci://` scheme tells Helm to interact with the OCI protocol. You must specify `--version` when installing from OCI because OCI registries do not have a "latest" tag concept for Helm charts.

---

### Exercise A-16: Install with `--set` Overriding Nested Values

**Scenario:** A chart has deeply nested configuration and you need to override a specific leaf value without a values file.

**Environment:**
- Chart `myorg/webapp` has structure `ingress.annotations."nginx.ingress.kubernetes.io/ssl-redirect"`
- You want to set it to `"false"`

**Task:** Install `myorg/webapp` as release `web-portal`, overriding the nested annotation.

**Solution:**
```bash
helm install web-portal myorg/webapp \
  --set ingress.annotations."nginx\.ingress\.kubernetes\.io/ssl-redirect"=false
```
When a key contains dots, you escape them with backslashes so Helm does not interpret them as nested key separators. The `false` value is parsed as a boolean.

---

## CATEGORY B: Release Upgrade (12 Exercises)

### Exercise B-1: Upgrade Changing a Single Value

**Scenario:** Your `my-nginx` release is running with default settings. You want to increase the replica count from 1 to 3.

**Environment:**
- Release `my-nginx` is installed from `bitnami/nginx`

**Task:** Upgrade the `my-nginx` release to set `replicaCount` to 3.

**Solution:**
```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=3
```
`helm upgrade` replaces the previous revision's resources with the newly rendered ones. Only the Kubernetes objects that differ are updated; unchanged resources are left untouched.

---

### Exercise B-2: Upgrade with a New Values File

**Scenario:** The operations team has provided an updated values file with tuned resource limits and replica settings for the production environment.

**Environment:**
- Release `api-gateway` installed from `myorg/gateway`
- New values file: `prod-v2.yaml`

**Task:** Upgrade `api-gateway` using `prod-v2.yaml` as the values source.

**Solution:**
```bash
helm upgrade api-gateway myorg/gateway -f prod-v2.yaml
```
Values from `prod-v2.yaml` are merged with the chart's defaults. Values not specified in `prod-v2.yaml` fall back to the chart defaults, unless `--reuse-values` is also specified.

---

### Exercise B-3: Upgrade Preserving Previous Values (`--reuse-values`)

**Scenario:** During a previous upgrade you set several custom values. Now you only want to change the image tag, keeping all other custom values as they were.

**Environment:**
- Release `webapp` has custom values set during the initial install
- You only want to change `image.tag` to `2.4.1`

**Task:** Upgrade `webapp` to set `image.tag=2.4.1` while preserving all previously set custom values.

**Solution:**
```bash
helm upgrade webapp myorg/webapp --set image.tag=2.4.1 --reuse-values
```
`--reuse-values` merges the previously supplied custom values (from the last install/upgrade) with the new `--set` value. Without this flag, Helm would reset to chart defaults for any value not explicitly provided.

---

### Exercise B-4: Upgrade Resetting Values (`--reset-values`)

**Scenario:** You previously installed a chart with many custom values for testing. Now you want to upgrade and discard all those custom values, reverting completely to the chart defaults.

**Environment:**
- Release `test-db` has accumulated many custom `--set` overrides across multiple upgrades
- You want a clean reset to chart defaults

**Task:** Upgrade `test-db` to the same chart version, resetting all values to defaults.

**Solution:**
```bash
helm upgrade test-db bitnami/mysql --reset-values
```
`--reset-values` discards all previously supplied custom values and uses only the chart's built-in `values.yaml`. Any new `--set` or `-f` provided alongside `--reset-values` are applied on top of the defaults.

---

### Exercise B-5: Upgrade with `--install` (Idempotent Deploy)

**Scenario:** Your CI pipeline runs the same command whether the release exists or not. You want a single command that installs if missing or upgrades if present.

**Environment:**
- Release `ci-service` may or may not exist from a previous pipeline run

**Task:** Write a command that ensures `ci-service` is deployed at the desired state, regardless of whether it currently exists.

**Solution:**
```bash
helm upgrade --install ci-service myorg/microservice -f ci-values.yaml
```
`--install` makes the `upgrade` command idempotent: if the release does not exist, Helm performs an `install` instead. If the release exists, it performs an `upgrade`. This is the standard pattern for CI/CD pipelines.

---

### Exercise B-6: Upgrade with `--atomic` (Auto-Rollback)

**Scenario:** You are upgrading a production release. If the upgrade fails, you want Helm to automatically roll back to the last successful revision.

**Environment:**
- Release `prod-api` is on revision 5 and is healthy
- You are upgrading to a new chart version that may fail

**Task:** Upgrade `prod-api` with automatic rollback on failure.

**Solution:**
```bash
helm upgrade prod-api myorg/api --set image.tag=3.0.0 --atomic
```
`--atomic` in an upgrade context means: if the upgrade fails (any hook fails, Pods crash, timeout exceeded), Helm automatically rolls back to the previous successful revision. This is the safest mode for production upgrades.

---

### Exercise B-7: Upgrade with `--force` (Recreate Resources)

**Scenario:** A ConfigMap used by your deployment has changed, but Kubernetes does not automatically restart Pods when a ConfigMap is updated. You need to force the Pods to be recreated.

**Environment:**
- Release `config-consumer` uses a ConfigMap that you are updating via Helm

**Task:** Upgrade `config-consumer` and force the recreation of all managed resources.

**Solution:**
```bash
helm upgrade config-consumer myorg/app --set config.logLevel=debug --force
```
`--force` tells Helm to use `kubectl apply --force` semantics, which deletes and recreates resources that cannot be updated in place (like StatefulSets with immutable fields). It also forces rolling updates on Deployments even when only non-pod-spec fields change.

---

### Exercise B-8: Upgrade Changing Image Tag

**Scenario:** A new container image has been built and pushed. You need to update the running deployment to use the new tag.

**Environment:**
- Release `inventory-svc` is currently on image tag `1.2.0`
- New image tag `1.3.0` is available in the registry

**Task:** Upgrade `inventory-svc` to use the new image tag `1.3.0`.

**Solution:**
```bash
helm upgrade inventory-svc myorg/inventory --set image.tag=1.3.0
```
If the chart uses `image.tag` as the primary image version parameter, this command updates the Deployment's container image. Kubernetes performs a rolling update, spinning up new Pods and terminating old ones.

---

### Exercise B-9: Upgrade Adding an Environment Variable

**Scenario:** The application needs a new environment variable `FEATURE_FLAG_NEW_UI` set to `true`.

**Environment:**
- Release `frontend` is deployed from `myorg/web-ui`
- The chart supports `env` as a list of name/value pairs

**Task:** Upgrade `frontend` to add the new environment variable.

**Solution:**
```bash
helm upgrade frontend myorg/web-ui \
  --set-string env[0].name=FEATURE_FLAG_NEW_UI \
  --set-string env[0].value=true \
  --reuse-values
```
`--reuse-values` preserves all existing custom values. Then `--set-string` appends the new env var. Using `--set-string` ensures the boolean-like value `true` is passed as the string `"true"` rather than being coerced to a YAML boolean.

---

### Exercise B-10: Upgrade with `--dry-run` to Preview

**Scenario:** Before applying an upgrade to production, you want to inspect the exact Kubernetes manifests that would be created, without making any changes.

**Environment:**
- Release `prod-cache` is currently running
- You plan to upgrade with new memory limits

**Task:** Preview what `prod-cache` would look like after the planned upgrade without actually applying it.

**Solution:**
```bash
helm upgrade prod-cache bitnami/redis --set master.persistence.size=20Gi --dry-run
```
`--dry-run` renders all templates with the proposed values and prints the resulting Kubernetes manifests to stdout, without creating or updating anything in the cluster. This is the primary validation step before any production upgrade.

---

### Exercise B-11: Upgrade with `--wait`

**Scenario:** You are upgrading a StatefulSet and need the command to block until all new Pods are ready before continuing the pipeline.

**Environment:**
- Release `analytics-db` is a StatefulSet with 5 replicas
- Restart takes approximately 3 minutes per pod

**Task:** Upgrade `analytics-db` and wait for all resources to reach a ready state, with an extended timeout.

**Solution:**
```bash
helm upgrade analytics-db bitnami/postgresql --set primary.persistence.size=500Gi --wait --timeout 20m
```
`--wait` blocks until all Deployments, StatefulSets, and DaemonSets reach their desired replica count and all Pods are Ready. The `--timeout` extends the deadline to accommodate the StatefulSet's slow initialisation.

---

### Exercise B-12: Upgrade Changing Resource Limits

**Scenario:** The application is hitting memory limits during peak hours. You need to double the memory limit and request.

**Environment:**
- Release `api-workers` deployed
- Chart supports `resources.limits.memory` and `resources.requests.memory`

**Task:** Upgrade `api-workers` to set memory limits to `1Gi` and requests to `512Mi`.

**Solution:**
```bash
helm upgrade api-workers myorg/worker \
  --set resources.limits.memory=1Gi \
  --set resources.requests.memory=512Mi \
  --reuse-values
```
The `--set` flags target nested YAML paths. `--reuse-values` ensures other custom settings (like replica count, env vars) are preserved. After the upgrade, Pods are recreated with the new resource constraints.

---

## CATEGORY C: Rollback (8 Exercises)

### Exercise C-1: Rollback to Previous Revision

**Scenario:** An upgrade to `frontend` introduced a bug. You need to revert to the revision before the upgrade.

**Environment:**
- Release `frontend` is at revision 8
- Revision 7 was stable

**Task:** Roll back `frontend` to the immediately previous revision.

**Solution:**
```bash
helm rollback frontend
```
Without a revision number, `helm rollback` defaults to the previous revision (`CURRENT - 1`). Helm creates a new revision that is a copy of the target revision's state.

---

### Exercise C-2: Rollback to a Specific Revision

**Scenario:** You performed multiple bad upgrades. The last known-good state was revision 5, and the current is revision 8.

**Environment:**
- Release `backend-api` is at revision 8
- Revision 5 was the last stable deployment

**Task:** Roll back `backend-api` specifically to revision 5.

**Solution:**
```bash
helm rollback backend-api 5
```
The second positional argument is the target revision number. Helm retrieves the manifest and values from revision 5 and creates a new revision that matches it. The release history will show a new revision (e.g., 9) that is equivalent to 5.

---

### Exercise C-3: View History Before Rollback

**Scenario:** You suspect a recent change caused issues, but you are unsure which revision to roll back to. You want to review the history first.

**Environment:**
- Release `auth-service` has undergone numerous upgrades over the past week

**Task:** List all revisions of `auth-service` to identify the correct rollback target.

**Solution:**
```bash
helm history auth-service
```
Output:
```
REVISION  UPDATED                   STATUS          CHART               APP VERSION  DESCRIPTION
1         Mon Jul  7 09:15:00 2025  superseded      auth-service-1.0.0  1.0.0        Install complete
2         Mon Jul  7 14:30:00 2025  superseded      auth-service-1.1.0  1.1.0        Upgrade complete
3         Tue Jul  8 10:00:00 2025  deployed        auth-service-1.2.0  1.2.0        Upgrade complete
```
Look for the last revision with status `deployed` or `superseded` at a date when the service was known to be stable.

---

### Exercise C-4: Rollback with `--dry-run`

**Scenario:** Before executing a rollback, you want to see exactly which Kubernetes manifests would be restored.

**Environment:**
- Release `data-pipeline` at revision 12, stable revision was 10

**Task:** Preview the rollback to revision 10 without actually performing it.

**Solution:**
```bash
helm rollback data-pipeline 10 --dry-run
```
`--dry-run` renders the manifests from revision 10 and prints them to stdout without applying any changes. This lets you audit the exact state that would be restored before committing to the rollback.

---

### Exercise C-5: Rollback and Verify

**Scenario:** After rolling back, you need to confirm the rollback completed successfully and the release is healthy.

**Environment:**
- Rollback of `monitoring-stack` to revision 4 was just executed

**Task:** Verify that the rollback was applied correctly and the release is in a healthy state.

**Solution:**
```bash
helm rollback monitoring-stack 4 --wait && helm status monitoring-stack
```
Adding `--wait` to rollback blocks until all resources are ready. Then `helm status` shows the current state, including the latest revision number, deployment status, and any notes. A healthy release will show `STATUS: deployed`.

---

### Exercise C-6: Rollback with `--wait`

**Scenario:** You are rolling back a database deployment. You need the rollback to fully complete (all Pods ready) before proceeding to run data integrity checks.

**Environment:**
- Release `db-primary` at revision 7, rolling back to revision 6
- Database pods take time to become ready

**Task:** Roll back `db-primary` to revision 6 and wait for full readiness.

**Solution:**
```bash
helm rollback db-primary 6 --wait --timeout 10m
```
`--wait` ensures the rollout is complete before the command returns. `--timeout 10m` gives the database Pods enough time to restart and become ready. Without `--wait`, the command returns immediately while Kubernetes processes the changes in the background.

---

### Exercise C-7: Check Currently Deployed Revision

**Scenario:** You need to know which revision of a release is currently active (the `deployed` one).

**Environment:**
- Multiple releases in the cluster
- You need to find the current revision of `payment-svc`

**Task:** Determine which revision of `payment-svc` is the currently deployed one.

**Solution:**
```bash
helm history payment-svc --max 1
```
Or, more reliably:
```bash
helm history payment-svc | grep deployed
```
The `history` command lists all revisions. The row with `STATUS: deployed` indicates the active revision. The `--max` flag can limit output to the most recent N entries.

---

### Exercise C-8: Configure Maximum History Limit

**Scenario:** Your release has accumulated over 200 revisions due to frequent CI-driven upgrades. You want to limit the stored history to 10 to save etcd storage.

**Environment:**
- Release `ci-app` has 250 revisions in its history
- You want to cap future storage at 10 revisions

**Task:** Upgrade `ci-app` to limit the maximum number of stored revisions to 10.

**Solution:**
```bash
helm upgrade ci-app myorg/ci-app --set maxHistory=10 --history-max 10 --reuse-values
```
`--history-max` tells Helm to keep at most N revisions in the release's secret store. Older revisions are automatically pruned on subsequent upgrades. This is critical for CI-driven deployments that can generate hundreds of revisions and strain etcd.

---

## CATEGORY D: Repository Management (10 Exercises)

### Exercise D-1: Add a Repository

**Scenario:** You need to access charts from the Bitnami repository.

**Environment:**
- Fresh Helm installation with no repositories configured
- Bitnami repository URL: `https://charts.bitnami.com/bitnami`

**Task:** Add the Bitnami repository with the name `bitnami`.

**Solution:**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
This downloads the repository's `index.yaml` and stores the repository entry in the local Helm configuration. The name `bitnami` is an alias used to reference the repository in install/upgrade/search commands.

---

### Exercise D-2: Update All Repositories

**Scenario:** You added several repositories weeks ago. New chart versions have been published since then, and your local index cache is stale.

**Environment:**
- Multiple repositories added: `bitnami`, `prometheus-community`, `ingress-nginx`

**Task:** Refresh the local index cache for all configured repositories.

**Solution:**
```bash
helm repo update
```
This fetches the latest `index.yaml` from every configured repository and updates the local cache. Run this before `helm search repo` or `helm install` to ensure you see the latest available chart versions.

---

### Exercise D-3: Search a Repository by Keyword

**Scenario:** You need a MySQL chart from the Bitnami repository but don't know the exact chart name.

**Environment:**
- Bitnami repository is added

**Task:** Search all configured repositories for charts related to `mysql`.

**Solution:**
```bash
helm search repo mysql
```
Output example:
```
NAME            CHART VERSION   APP VERSION     DESCRIPTION
bitnami/mysql   9.10.0          8.0.36          MySQL is a fast, reliable, scalable...
```
The search scans the local index cache of all repositories for the keyword in chart names and descriptions.

---

### Exercise D-4: Search a Repository with Regex

**Scenario:** You want to find charts whose names match a pattern: starting with `nginx` or containing `proxy`.

**Environment:**
- Multiple repositories added

**Task:** Search for charts matching the regex pattern `nginx|proxy`.

**Solution:**
```bash
helm search repo --regexp "nginx|proxy"
```
The `--regexp` flag interprets the search query as a regular expression. This is useful when you need to match multiple patterns in a single search. Without `--regexp`, the query is a simple substring match.

---

### Exercise D-5: Search ArtifactHub

**Scenario:** You want to find a Helm chart but are unsure which repository hosts it. You decide to search the public ArtifactHub catalog.

**Environment:**
- No specific repository known for the `cert-manager` chart

**Task:** Search ArtifactHub for charts matching `cert-manager`.

**Solution:**
```bash
helm search hub cert-manager
```
`helm search hub` queries ArtifactHub (https://artifacthub.io), the central public registry for Helm charts. Results include charts from many repositories, with URLs you can add using `helm repo add`.

---

### Exercise D-6: Remove a Repository

**Scenario:** Your team is no longer using a third-party repository and you want to clean up your Helm configuration.

**Environment:**
- Repository `old-vendor` is no longer needed

**Task:** Remove the `old-vendor` repository from your Helm configuration.

**Solution:**
```bash
helm repo remove old-vendor
```
This deletes the repository entry from the local Helm configuration. The cached index is also removed. Future `helm search repo` and `helm install` commands will no longer reference this repository.

---

### Exercise D-7: List All Configured Repositories

**Scenario:** You need to audit which repositories are configured on a new machine before starting development.

**Environment:**
- Unknown repository configuration on a colleague's machine

**Task:** List all Helm repositories currently configured.

**Solution:**
```bash
helm repo list
```
Output example:
```
NAME                    URL
bitnami                 https://charts.bitnami.com/bitnami
prometheus-community    https://prometheus-community.github.io/helm-charts
ingress-nginx           https://kubernetes.github.io/ingress-nginx
```
This displays the alias, URL, and status of every configured repository.

---

### Exercise D-8: Add a Repository with Username and Password

**Scenario:** Your organisation uses a private chart repository that requires HTTP Basic Authentication.

**Environment:**
- Private repo at `https://charts.internal.example.com`
- Username: `helm-bot`
- Password: stored in environment variable `$HELM_REPO_PASS`

**Task:** Add the private repository with authentication credentials.

**Solution:**
```bash
helm repo add internal https://charts.internal.example.com \
  --username helm-bot \
  --password "$HELM_REPO_PASS"
```
`--username` and `--password` provide Basic Auth credentials. Helm stores the password in the local configuration. For CI pipelines, prefer reading the password from a secret manager and passing it via environment variable.

---

### Exercise D-9: Add a Repository with TLS Certificate

**Scenario:** A private repository uses a self-signed TLS certificate. You have the CA certificate file and need to trust it.

**Environment:**
- Private repo at `https://charts.secure.example.com`
- CA certificate at `/etc/ssl/certs/internal-ca.crt`

**Task:** Add the repository with the custom CA certificate.

**Solution:**
```bash
helm repo add secure https://charts.secure.example.com \
  --ca-file /etc/ssl/certs/internal-ca.crt
```
`--ca-file` specifies a custom Certificate Authority certificate to verify the repository's TLS certificate. Alternatively, use `--insecure-skip-tls-verify` to bypass TLS validation (not recommended for production).

---

### Exercise D-10: Create an Index for a Custom Repository

**Scenario:** You maintain an internal chart repository served via a simple HTTP server. After adding a new packaged chart, you need to regenerate the index.

**Environment:**
- Chart packages in `/var/www/charts/`
- New chart `myapp-1.0.0.tgz` was just added

**Task:** Generate (or regenerate) the repository index file at `/var/www/charts/index.yaml`.

**Solution:**
```bash
helm repo index /var/www/charts/ --url https://charts.internal.example.com
```
`helm repo index` scans the directory for all `.tgz` chart archives, reads their `Chart.yaml` files, and generates an `index.yaml` with metadata for every chart. The `--url` flag sets the base URL for chart download links in the index.

---

## CATEGORY E: Chart Inspection (10 Exercises)

### Exercise E-1: Show Chart Metadata

**Scenario:** Before installing a chart, you want to read its metadata: name, version, app version, and description.

**Environment:**
- Chart `bitnami/nginx` is available in the repository cache

**Task:** Display the metadata (Chart.yaml contents) of the `bitnami/nginx` chart.

**Solution:**
```bash
helm show chart bitnami/nginx
```
Output:
```yaml
apiVersion: v2
name: nginx
description: NGINX Open Source is a web server...
type: application
version: 15.6.0
appVersion: 1.25.4
```
This shows the `Chart.yaml` content, including the chart's API version, type, and maintainers. Useful for understanding what the chart provides.

---

### Exercise E-2: Show Chart Values

**Scenario:** You want to see all configurable values of a chart before deciding what to override.

**Environment:**
- Chart `bitnami/postgresql` is available

**Task:** Display all configurable values and their defaults for `bitnami/postgresql`.

**Solution:**
```bash
helm show values bitnami/postgresql
```
Output:
```yaml
architecture: standalone
auth:
  username: postgres
  database: postgres
primary:
  persistence:
    size: 8Gi
```
This displays the chart's `values.yaml` file. Every key shown can be overridden with `--set` or `-f`. This is the first command to run when working with an unfamiliar chart.

---

### Exercise E-3: Show Chart README

**Scenario:** The chart's README contains important documentation about prerequisites, parameters, and upgrade considerations.

**Environment:**
- Chart `bitnami/mongodb` is available

**Task:** Display the README documentation of the `bitnami/mongodb` chart.

**Solution:**
```bash
helm show readme bitnami/mongodb
```
The README typically describes the chart's purpose, prerequisites, parameters table, and common deployment patterns. Always read this before installing a chart in production.

---

### Exercise E-4: Show All Chart Information

**Scenario:** You want a comprehensive view of everything the chart provides: metadata, values, and README, all in one command.

**Environment:**
- Chart `bitnami/redis` is available

**Task:** Display all available information for `bitnami/redis`.

**Solution:**
```bash
helm show all bitnami/redis
```
`helm show all` outputs the chart metadata, default values, and README sequentially. This is convenient for getting the full picture of a chart in a single command.

---

### Exercise E-5: Get Deployed Values for a Release

**Scenario:** You need to know what custom values were applied to an existing release (without the chart defaults mixed in).

**Environment:**
- Release `my-nginx` was installed with several custom `--set` values months ago
- Nobody documented the exact values used

**Task:** Retrieve only the user-supplied values that were used to deploy `my-nginx`.

**Solution:**
```bash
helm get values my-nginx
```
This returns only the values explicitly provided by the user during `install` or `upgrade`. It does not include chart defaults. This is the command to run when you need to know what was customised.

---

### Exercise E-6: Get All Deployed Values Including Defaults

**Scenario:** You need a complete picture of every value that was used to render the templates for a release, including chart defaults merged with user overrides.

**Environment:**
- Release `my-nginx` is deployed

**Task:** Retrieve the full effective values (defaults + user overrides) for `my-nginx`.

**Solution:**
```bash
helm get values my-nginx --all
```
The `--all` flag includes all chart default values in addition to user-supplied overrides. The output is the complete values map that was used to render the current revision.

---

### Exercise E-7: Get Rendered Manifest for a Release

**Scenario:** You want to see all Kubernetes resources (Deployment, Service, ConfigMap, etc.) that were created by a Helm release.

**Environment:**
- Release `monitoring` is deployed

**Task:** Output the full Kubernetes manifest of all resources managed by the `monitoring` release.

**Solution:**
```bash
helm get manifest monitoring
```
This outputs the complete set of rendered templates (YAML documents separated by `---`) that were applied to the cluster. This is what `kubectl apply` received when the release was installed or upgraded.

---

### Exercise E-8: Get Manifest for a Specific Revision

**Scenario:** You need to see what Kubernetes resources looked like at a specific point in time, not the current revision.

**Environment:**
- Release `api-gateway` has 15 revisions
- Revision 10 had a specific ingress configuration you need to inspect

**Task:** Retrieve the rendered manifest for revision 10 of `api-gateway`.

**Solution:**
```bash
helm get manifest api-gateway --revision 10
```
The `--revision` flag retrieves the manifest from a specific historical revision. This is invaluable for auditing what changed between revisions and diagnosing when a regression was introduced.

---

### Exercise E-9: Get Release Notes

**Scenario:** The chart author included helpful post-installation notes. You need to retrieve them for a deployed release.

**Environment:**
- Release `wordpress` was installed but the installation is complex
- The chart provides helpful configuration steps in its notes

**Task:** Retrieve the post-install/upgrade notes for the `wordpress` release.

**Solution:**
```bash
helm get notes wordpress
```
Release notes are defined in the chart's `templates/NOTES.txt` template. They typically contain instructions for accessing the service, retrieving generated passwords, and next steps.

---

### Exercise E-10: Get Release Hooks

**Scenario:** A release uses Helm hooks for database migrations and pre-install checks. You need to see what hooks were executed and their status.

**Environment:**
- Release `data-service` uses `pre-upgrade` and `post-install` hooks

**Task:** Retrieve the hook resources and their execution status for the `data-service` release.

**Solution:**
```bash
helm get hooks data-service
```
This displays all hook resources (Jobs, Pods) associated with the release, including their hook type (pre-install, post-upgrade, etc.) and deletion policy. It helps diagnose why a hook may have failed.

---

## CATEGORY F: Chart Development (10 Exercises)

### Exercise F-1: Create a New Chart

**Scenario:** You are starting a new microservice and need to create a Helm chart from scratch.

**Environment:**
- Empty project directory `/home/user/projects/`

**Task:** Create a new Helm chart named `user-service` in the current directory.

**Solution:**
```bash
helm create user-service
```
This scaffolds a new chart directory `user-service/` containing:
```
user-service/
  Chart.yaml
  values.yaml
  charts/
  templates/
    deployment.yaml
    service.yaml
    hpa.yaml
    ingress.yaml
    serviceaccount.yaml
    NOTES.txt
    _helpers.tpl
    tests/
```
This is the starting template for any new chart. You then modify the generated files.

---

### Exercise F-2: Lint a Chart

**Scenario:** You have made changes to your chart and want to check it for errors, missing required fields, and best-practice violations.

**Environment:**
- Chart directory at `/home/user/projects/user-service/`

**Task:** Lint the `user-service` chart and fix any issues found.

**Solution:**
```bash
helm lint /home/user/projects/user-service/
```
`helm lint` checks for:
- Chart.yaml schema conformance (missing required fields)
- Template rendering errors (syntax errors, undefined values)
- Missing recommended metadata (icon, source, maintainers)
- Deprecated API versions

Output shows any errors (must fix) or warnings (should fix).

---

### Exercise F-3: Package a Chart

**Scenario:** You have finalised your chart and need to create a distributable `.tgz` archive to publish to a repository.

**Environment:**
- Chart at `/home/user/projects/user-service/`
- Chart version is `0.1.0` as specified in `Chart.yaml`

**Task:** Package the `user-service` chart into a `.tgz` archive.

**Solution:**
```bash
helm package /home/user/projects/user-service/
```
This produces `user-service-0.1.0.tgz` in the current directory. The archive includes `Chart.yaml`, `values.yaml`, all templates, and dependencies, but excludes development files like `.git/` and editor backup files.

---

### Exercise F-4: Pull a Chart Without Installing

**Scenario:** You want to download a chart from a repository to inspect its templates locally, but you do not want to install it.

**Environment:**
- Repository `bitnami` is configured

**Task:** Download the `bitnami/nginx` chart to the local filesystem without installing it.

**Solution:**
```bash
helm pull bitnami/nginx
```
This downloads the `.tgz` package to the current directory. Use `--untar` to extract it immediately, or `--untardir` to specify an extraction directory. Without these flags, just the archive is saved.

---

### Exercise F-5: Pull a Chart and Untar

**Scenario:** You want to study the chart's templates and values. Downloading and extracting should happen in one step.

**Environment:**
- Repository `bitnami` is configured

**Task:** Download `bitnami/nginx` and extract it into a directory for local inspection.

**Solution:**
```bash
helm pull bitnami/nginx --untar
```
`--untar` unpacks the chart into a `nginx/` directory in the current path. Use `--untardir /path/to/dir` to extract elsewhere. This is the standard workflow for studying third-party charts before customising them.

---

### Exercise F-6: Verify a Signed Chart

**Scenario:** Your organisation signs charts with GPG for supply-chain security. You need to verify a chart's signature before installing.

**Environment:**
- GPG public key `security@company.com` is in your keyring
- Chart `user-service-0.1.0.tgz` and its provenance file `user-service-0.1.0.tgz.prov` are present

**Task:** Verify the provenance (cryptographic signature) of the packaged chart.

**Solution:**
```bash
helm verify user-service-0.1.0.tgz
```
`helm verify` checks the chart's `.prov` provenance file against the keyring to confirm the chart was signed by the expected key and has not been tampered with. This requires both the chart archive and its `.prov` file.

---

### Exercise F-7: Add a Dependency to a Chart

**Scenario:** Your `user-service` chart depends on a Redis instance. You want to declare Redis as a dependency.

**Environment:**
- `user-service` chart exists locally
- Redis chart is available from the Bitnami repository

**Task:** Add `bitnami/redis` as a dependency to `user-service`, version constraint `>=17.0.0 <18.0.0`.

**Solution:**
Add to `user-service/Chart.yaml`:
```yaml
dependencies:
  - name: redis
    version: ">=17.0.0 <18.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```
Then run:
```bash
helm dependency update user-service/
```
The `dependencies` section in `Chart.yaml` declares sub-charts. The `condition` field optionally toggles the dependency based on a values key (`redis.enabled`). `helm dependency update` downloads the dependency into the `charts/` directory and generates a `Chart.lock` file.

---

### Exercise F-8: Update Dependencies

**Scenario:** A new version of the Redis chart has been released with a security fix. You need to update your chart's dependencies.

**Environment:**
- `user-service` chart has outdated `Chart.lock`
- The version constraint in `Chart.yaml` allows the newer version

**Task:** Update all dependencies of `user-service` to the latest compatible versions.

**Solution:**
```bash
helm dependency update user-service/
```
This re-downloads all declared dependencies, resolving them to the latest versions that satisfy the constraints in `Chart.yaml`. The `charts/` directory is refreshed and `Chart.lock` is regenerated with new digests.

---

### Exercise F-9: List Dependencies

**Scenario:** You need to audit which dependencies a chart brings in.

**Environment:**
- Chart `user-service` has multiple dependencies

**Task:** List all dependencies of the `user-service` chart with their versions and status.

**Solution:**
```bash
helm dependency list user-service/
```
Output:
```
NAME    VERSION         REPOSITORY                              STATUS
redis   17.11.0         https://charts.bitnami.com/bitnami      ok
```
`STATUS` shows `ok` if the dependency is downloaded and matches the lock, `missing` if not downloaded, or `wrong version` if the `charts/` directory has a version different from `Chart.lock`.

---

### Exercise F-10: Build (Regenerate) Dependency Lock

**Scenario:** You manually modified the `charts/` directory contents and need to regenerate the lock file to match.

**Environment:**
- `user-service/Chart.lock` is out of sync with `charts/`
- Dependencies exist in `charts/`

**Task:** Regenerate the `Chart.lock` file to reflect the current state of downloaded dependencies.

**Solution:**
```bash
helm dependency build user-service/
```
`helm dependency build` regenerates the `Chart.lock` from the `charts/` directory contents (as opposed to `update`, which re-downloads from repositories). Use `build` when you already have the dependency archives and just need the lock file.

---

## CATEGORY G: Templating (8 Exercises)

### Exercise G-1: Render Template Locally

**Scenario:** During chart development, you want to see the rendered Kubernetes manifests without talking to a cluster.

**Environment:**
- Chart at `/home/user/projects/user-service/`
- No Kubernetes cluster required

**Task:** Render the `user-service` chart templates locally with default values.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/
```
`helm template` renders all templates using the chart's default `values.yaml` and prints the resulting Kubernetes manifests to stdout. It does not connect to any cluster. This is the primary local development loop command.

---

### Exercise G-2: Render Template with Values File

**Scenario:** You want to see how the manifests change when a specific values file is applied, without installing.

**Environment:**
- Chart at `/home/user/projects/user-service/`
- Values file: `staging-values.yaml`

**Task:** Render the chart templates using `staging-values.yaml` to preview the staging manifests.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/ -f staging-values.yaml
```
The `-f` flag works the same as with `install`/`upgrade`: it merges the file's values with chart defaults for rendering. The output is the complete manifest set for the staging environment.

---

### Exercise G-3: Render Template with `--set`

**Scenario:** You want to quickly test how a single value override affects the output without creating a values file.

**Environment:**
- Chart at `/home/user/projects/user-service/`

**Task:** Render the chart templates overriding `replicaCount` to `5` and `service.type` to `NodePort`.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/ \
  --set replicaCount=5 \
  --set service.type=NodePort
```
`helm template` accepts `--set` just like `install`. This is a fast way to experiment with value overrides during development without creating temporary files.

---

### Exercise G-4: Render Only a Specific Template (`--show-only`)

**Scenario:** Your chart has many templates. You only want to inspect the Ingress manifest to verify a recent change.

**Environment:**
- Chart at `/home/user/projects/user-service/`
- The ingress template is at `templates/ingress.yaml`

**Task:** Render only the Ingress template from the chart.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/ --show-only templates/ingress.yaml
```
`--show-only` filters the output to only the specified template file(s). You can provide the flag multiple times to show several templates. This reduces noise when debugging a specific resource.

---

### Exercise G-5: Render Template as an Upgrade Simulation

**Scenario:** You want to see what would change if you upgraded an existing release, without actually performing the upgrade. You need the output to be a diff against the current revision.

**Environment:**
- Release `my-nginx` is deployed
- You plan to change `replicaCount` from 1 to 3

**Task:** Simulate the upgrade and see the difference between the current state and the proposed state.

**Solution:**
```bash
helm diff upgrade my-nginx bitnami/nginx --set replicaCount=3
```
*Note: `helm diff` requires the `helm-diff` plugin.* Install it with:
```bash
helm plugin install https://github.com/databus23/helm-diff
```
Alternatively, use built-in dry-run:
```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=3 --dry-run > proposed.yaml
helm get manifest my-nginx > current.yaml
diff current.yaml proposed.yaml
```

---

### Exercise G-6: Render Template with API Versions

**Scenario:** Your chart targets Kubernetes 1.27+, but you want to verify the templates work on an older cluster with different API versions.

**Environment:**
- Chart uses `networking.k8s.io/v1` for Ingress
- Target cluster has older capabilities

**Task:** Render the chart and validate it against a specific set of available API versions.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/ \
  --api-versions networking.k8s.io/v1 \
  --api-versions apps/v1 \
  --api-versions v1
```
`--api-versions` tells Helm which API versions are available in the target cluster. Helm uses this to resolve `.Capabilities.APIVersions` in templates. If your template uses `{{ .Capabilities.APIVersions.Has }}`, this flag can simulate different cluster capabilities.

---

### Exercise G-7: Render Template with `--kube-version`

**Scenario:** Your CI pipeline tests charts against multiple Kubernetes versions. You need to simulate rendering on Kubernetes 1.28.

**Environment:**
- Chart at `/home/user/projects/user-service/`
- CI needs to validate against k8s 1.28

**Task:** Render the chart as if the target cluster were Kubernetes 1.28.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/ --kube-version 1.28
```
`--kube-version` sets the `.Capabilities.KubeVersion` value in templates. This affects version-gated logic like `{{ semverCompare ">=1.25" .Capabilities.KubeVersion.Version }}`. Essential for CI pipelines that validate charts across Kubernetes versions.

---

### Exercise G-8: Debug Template Rendering

**Scenario:** Your template is failing to render, and the error message is cryptic. You need detailed debugging output.

**Environment:**
- Chart at `/home/user/projects/user-service/`
- Template error: `nil pointer evaluating interface {}.image`

**Task:** Render the chart with debug output to identify the exact template and line causing the error.

**Solution:**
```bash
helm template user-service /home/user/projects/user-service/ --debug
```
`--debug` prints verbose output including each template file as it is processed, the values used for rendering, and detailed error messages with line numbers. This is the first flag to add when template rendering fails with an unclear error.

---

## CATEGORY H: Debugging (8 Exercises)

### Exercise H-1: Dry-Run Install

**Scenario:** Before installing a complex chart, you want to see every Kubernetes resource it would create, without making any changes.

**Environment:**
- Chart `bitnami/wordpress` is available
- You want to preview without committing

**Task:** Perform a dry-run installation of `bitnami/wordpress` to audit all resources.

**Solution:**
```bash
helm install wp-test bitnami/wordpress --dry-run
```
`--dry-run` renders all templates, performs the server-side validation that would occur during installation, and prints the full manifests. No resources are created. This catches structural YAML issues and schema violations before they affect the cluster.

---

### Exercise H-2: Dry-Run with Debug

**Scenario:** A dry-run shows unexpected output and you cannot trace which template or value is causing it.

**Environment:**
- Chart installation dry-run produces unexpected ConfigMap data

**Task:** Run a dry-run with maximum verbosity to trace how values flow into templates.

**Solution:**
```bash
helm install myapp myorg/app --dry-run --debug
```
Combining `--dry-run --debug` shows:
- The final merged values map
- Each template file as it's processed
- The rendered output per template
- Any warnings or errors
This is the most powerful diagnostic command for understanding exactly what Helm will produce.

---

### Exercise H-3: Find Why a Release Failed

**Scenario:** You ran `helm install` and it failed. The error message just says "Error: INSTALLATION FAILED". You need more information.

**Environment:**
- Release `broken-app` is in `failed` state

**Task:** Diagnose why the `broken-app` release failed.

**Solution:**
```bash
helm status broken-app
```
The status output shows:
- The last deployment stage reached
- Hook execution results
- Any error messages from the failed operation
- Resources that were created before the failure

For even more detail:
```bash
helm history broken-app
kubectl describe <failing-resource> -n <namespace>
helm get manifest broken-app | kubectl apply --dry-run=client -f -
```

---

### Exercise H-4: Debug an Immutable Field Error

**Scenario:** An upgrade fails with `field is immutable` because the chart attempts to change a Selector or other immutable field on an existing resource.

**Environment:**
- Release `svc-discovery` fails upgrade with "field is immutable"

**Task:** Identify the immutable field change and execute a workaround.

**Solution:**
First, inspect the diff:
```bash
helm get manifest svc-discovery > current.yaml
helm upgrade svc-discovery myorg/chart --dry-run > proposed.yaml
diff current.yaml proposed.yaml
```

Once you identify the immutable field change, you have two options:

Option 1 – Delete and recreate:
```bash
kubectl delete deployment svc-discovery -n <namespace>
helm upgrade svc-discovery myorg/chart --set correctedField=value
```

Option 2 – Use `--force` (destructive, use with caution):
```bash
helm upgrade svc-discovery myorg/chart --force
```

The proper long-term fix is to ensure the chart template does not modify immutable fields (like `spec.selector.matchLabels` on Deployments) between upgrades.

---

### Exercise H-5: Debug Missing Value Error

**Scenario:** Template rendering fails with an error: `"<.Values.serviceAccount.name>: nil pointer evaluating interface {}.name"`. A required value is missing.

**Environment:**
- Chart template references `.Values.serviceAccount.name`
- The value was not provided and has no default

**Task:** Fix the error by providing the missing value.

**Solution:**
First, check the chart's values schema:
```bash
helm show values myorg/myapp | grep -A5 serviceAccount
```

Then provide the missing value:
```bash
helm upgrade myapp myorg/myapp \
  --set serviceAccount.name=myapp-sa \
  --set serviceAccount.create=true \
  --reuse-values
```

For chart authors, the permanent fix is to either:
- Add a default in `values.yaml`
- Add a `required` function call with a clear message:
  ```
  {{ required "serviceAccount.name is required" .Values.serviceAccount.name }}
  ```
- Use the `default` function: `{{ default "default-sa" .Values.serviceAccount.name }}`

---

### Exercise H-6: Debug a Hook Failure

**Scenario:** A `pre-upgrade` hook Job failed, and the entire upgrade was rolled back. You need to understand why the hook failed.

**Environment:**
- Release `data-migrator` has a failed `pre-upgrade` hook
- The upgrade was rolled back

**Task:** Find the hook's Pod logs and diagnose the failure.

**Solution:**
```bash
helm get hooks data-migrator
```

Look for the hook Job name, then:
```bash
kubectl get pods -n <namespace> -l helm.sh/chart=data-migrator
kubectl logs <hook-pod-name> -n <namespace>
kubectl describe job <hook-job-name> -n <namespace>
```

Common hook failure causes:
- Missing environment variables or secrets
- Database connection failures
- Incorrect hook weight ordering
- Hook Pod lacking required RBAC permissions
- Image pull errors

Also check the hook's deletion policy: if `before-hook-creation`, Helm deletes the previous hook before creating the new one, so you may need to check historical logs.

---

### Exercise H-7: Debug an RBAC Issue

**Scenario:** A release's ServiceAccount lacks permissions to perform an action. Pods are crash-looping with "forbidden" errors.

**Environment:**
- Release `monitoring-agent` deployed with a ServiceAccount `monitoring-agent-sa`
- Pods log: `User "system:serviceaccount:monitoring:monitoring-agent-sa" cannot list resource "pods"`

**Task:** Diagnose and fix the RBAC permissions.

**Solution:**
First, check the current RBAC resources:
```bash
helm get manifest monitoring-agent | grep -A20 -E "ServiceAccount|ClusterRole|RoleBinding"
kubectl describe clusterrolebinding -l app.kubernetes.io/instance=monitoring-agent
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitoring-agent-sa -n monitoring
```

Then fix by adding the required permissions. Either:
1. Update the chart values to enable RBAC creation with sufficient rules
2. Create a custom values override:
   ```bash
   helm upgrade monitoring-agent myorg/agent \
     --set rbac.create=true \
     --set rbac.rules[0].apiGroups='{""}' \
     --set rbac.rules[0].resources='{pods,services}' \
     --set rbac.rules[0].verbs='{get,list,watch}' \
     --reuse-values
   ```

---

### Exercise H-8: Debug Dependency Issues

**Scenario:** Your chart depends on another chart, but the dependency is not being installed. The sub-chart resources are missing from the cluster.

**Environment:**
- `user-service` chart declares a dependency on `bitnami/redis`
- Redis resources are not showing up

**Task:** Diagnose and fix the missing dependency.

**Solution:**
First, check if dependencies are downloaded:
```bash
helm dependency list user-service/
ls user-service/charts/
```

If the `charts/` directory is empty:
```bash
helm dependency update user-service/
```

If dependencies exist but are not deployed, check for a condition flag:
```bash
grep -r "condition\|enabled" user-service/Chart.yaml user-service/values.yaml
```

A condition like `condition: redis.enabled` in Chart.yaml means the dependency is only installed when `redis.enabled` is `true`. If `values.yaml` has `redis.enabled: false`, set it:
```bash
helm upgrade user-service myorg/user-service --set redis.enabled=true --reuse-values
```

Also check that the dependency's template does not have its own conditions that might skip resource creation.

---

## CATEGORY I: Release Management (8 Exercises)

### Exercise I-1: List All Releases

**Scenario:** You need an inventory of all Helm-managed applications in the current namespace.

**Environment:**
- Several releases installed: `nginx`, `redis`, `postgresql`, `wordpress`

**Task:** List all Helm releases in the current namespace.

**Solution:**
```bash
helm list
```
Output:
```
NAME        NAMESPACE   REVISION    UPDATED                                 STATUS      CHART               APP VERSION
nginx       default     1           2026-07-10 09:15:00.000 +0000 UTC     deployed    nginx-15.6.0        1.25.4
redis       default     3           2026-07-09 14:30:00.000 +0000 UTC     deployed    redis-18.2.0        7.2.4
```
`helm list` (or `helm ls`) shows all releases in the current namespace with their revision, status, chart, and app version.

---

### Exercise I-2: List Releases in All Namespaces

**Scenario:** You need a cluster-wide view of all Helm releases across every namespace.

**Environment:**
- Releases scattered across `default`, `monitoring`, `staging`, and `production` namespaces

**Task:** List all Helm releases in the entire cluster, across all namespaces.

**Solution:**
```bash
helm list --all-namespaces
```
Or:
```bash
helm list -A
```
`--all-namespaces` (or `-A`) queries every namespace and displays all Helm releases. This is essential for cluster-wide auditing and inventory management.

---

### Exercise I-3: List Only Failed Releases

**Scenario:** A recent batch deployment had issues. You need to quickly find all releases that are in a failed state.

**Environment:**
- Many releases exist; some may have failed

**Task:** List only releases that are in a `failed` state.

**Solution:**
```bash
helm list --failed
```
This filters the output to releases with `STATUS: failed`. Other status filters include `--deployed`, `--pending`, `--pending-install`, `--pending-upgrade`, `--pending-rollback`, `--superseded`, and `--uninstalling`.

---

### Exercise I-4: List Releases with a Selector

**Scenario:** Your team labels releases with `owner=platform-team`. You need to list only the releases owned by your team.

**Environment:**
- Releases are labeled with various `owner` labels

**Task:** List all Helm releases labeled `owner=platform-team`.

**Solution:**
```bash
helm list --selector owner=platform-team
```
Or:
```bash
helm list -l owner=platform-team
```
`--selector` (or `-l`) filters releases by Kubernetes labels. Labels are set with `--labels owner=platform-team` during `helm install`. This enables multi-tenant release management.

---

### Exercise I-5: Check Release Status

**Scenario:** You need detailed status of a specific release, including the resources it manages, their health, and hook execution results.

**Environment:**
- Release `payment-api` is deployed

**Task:** Display the full status of the `payment-api` release.

**Solution:**
```bash
helm status payment-api
```
Output includes:
- Last deployment time
- Namespace
- Status (deployed, failed, pending)
- Revision number
- List of resources created (Deployment, Service, ConfigMap, etc.)
- Release notes (NOTES.txt)
- Hook execution results (if any)

For even more detail (including computed values):
```bash
helm status payment-api --show-resources --show-desc
```

---

### Exercise I-6: Get Detailed Release Information

**Scenario:** You need to inspect every aspect of a release: its values, manifest, hooks, notes, and history, all for auditing purposes.

**Environment:**
- Release `critical-app` needs a full audit

**Task:** Retrieve all available information about `critical-app` in one go.

**Solution:**
Since `helm get` does not have an "all" subcommand, run each individually:
```bash
helm get values critical-app --all > audit-values.yaml
helm get manifest critical-app > audit-manifest.yaml
helm get hooks critical-app > audit-hooks.yaml
helm get notes critical-app > audit-notes.txt
helm history critical-app > audit-history.txt
helm status critical-app > audit-status.txt
```

Or in a script:
```bash
for cmd in "values --all" manifest hooks notes; do
  echo "=== helm get $cmd critical-app ==="
  helm get $cmd critical-app
  echo
done
helm history critical-app
helm status critical-app
```

---

### Exercise I-7: Uninstall a Release

**Scenario:** The `test-app` release was a temporary deployment and is no longer needed. You must clean up all its resources.

**Environment:**
- Release `test-app` is deployed in the `dev` namespace

**Task:** Uninstall `test-app` and remove all associated Kubernetes resources.

**Solution:**
```bash
helm uninstall test-app
```
Or in a specific namespace:
```bash
helm uninstall test-app --namespace dev
```
This removes all Kubernetes resources that were created by the release. The release record is completely deleted (unlike `delete` in Helm v2, which kept a tombstone). There is no way to rollback after uninstall.

---

### Exercise I-8: Uninstall Keeping History

**Scenario:** You need to uninstall a release but want to preserve the release history in case you need to review or restore it later. This is important for audit trails.

**Environment:**
- Release `legacy-service` is being decommissioned
- Compliance requires keeping deployment records

**Task:** Uninstall `legacy-service` but retain its deployment history.

**Solution:**
```bash
helm uninstall legacy-service --keep-history
```
`--keep-history` removes the Kubernetes resources but retains the release's revision history as Secrets in the namespace. This allows `helm history legacy-service` to still work after uninstall. To later restore:
```bash
helm rollback legacy-service <last-revision>
```
Note that this restores the Kubernetes resources from the stored manifests.

---

## CATEGORY J: Production (11 Exercises)

### Exercise J-1: Production Install with All Best Practices

**Scenario:** You are deploying a mission-critical application to production. You must apply all Helm best practices: atomic deployment, resource limits, probes, specific version, wait for readiness, and proper release naming.

**Environment:**
- Production Kubernetes cluster
- Chart `myorg/orders-api` version `2.1.0`
- Production values file: `prod-values.yaml`
- Target namespace: `production`

**Task:** Install `orders-api` in production with all recommended safety flags.

**Solution:**
```bash
helm install orders-api-prod myorg/orders-api \
  --version 2.1.0 \
  --namespace production \
  --create-namespace \
  --values prod-values.yaml \
  --atomic \
  --wait \
  --timeout 15m \
  --description "Production deployment of Orders API v2.1.0" \
  --labels "env=production,team=platform,tier=critical"
```

Best practices applied:
- `--version`: pinned chart version prevents unexpected changes
- `--atomic`: auto-cleanup on failure prevents orphaned resources
- `--wait`: blocks until all Pods are ready before pipeline continues
- `--timeout 15m`: realistic timeout for application startup
- `--description`: adds context to release metadata for auditing
- `--labels`: enables filtering and multi-tenancy
- `--namespace`: isolates production workloads

---

### Exercise J-2: Rollback a Production Deployment

**Scenario:** A production upgrade introduced a performance regression. The on-call engineer must safely roll back with minimal downtime.

**Environment:**
- Release `orders-api-prod` at revision 12 (bad)
- Revision 11 was healthy
- Production namespace `production`

**Task:** Perform a safe production rollback and verify service health.

**Solution:**
```bash
helm history orders-api-prod --namespace production

helm rollback orders-api-prod 11 \
  --namespace production \
  --wait \
  --timeout 10m

helm status orders-api-prod --namespace production

kubectl get pods -n production -l app.kubernetes.io/instance=orders-api-prod

kubectl logs -n production -l app.kubernetes.io/instance=orders-api-prod --tail=20
```

The safe rollback workflow:
1. Check history to confirm the target revision
2. Rollback with `--wait` to block until Pods are ready
3. Verify status shows `deployed`
4. Check Pod health manually
5. Tail logs to confirm application is processing requests normally

---

### Exercise J-3: Migrate a Database Using Hooks

**Scenario:** Before upgrading your application, you must run a database schema migration. The migration Job must complete successfully before the app Pods are updated.

**Environment:**
- Application release `app-with-db` uses PostgreSQL
- A migration Job needs to run as part of the upgrade process
- The chart supports migration hooks

**Task:** Upgrade the release, ensuring the database migration runs first and the upgrade fails if migration fails.

**Solution:**

First, ensure the chart defines a migration hook in `templates/migration-job.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app-with-db.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["/app/migrate.sh"]
```

Then upgrade:
```bash
helm upgrade app-with-db myorg/app \
  --namespace production \
  --set image.tag=3.0.0 \
  --atomic \
  --wait \
  --timeout 15m
```

The hook annotations:
- `helm.sh/hook: pre-upgrade`: runs before the upgrade applies
- `helm.sh/hook-weight: "0"`: controls execution order among hooks (lower first)
- `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded`: cleans up old Jobs but keeps the successful one for inspection
- `--atomic`: rolls back if the migration (hook) or upgrade fails

---

### Exercise J-4: Canary Deployment Pattern

**Scenario:** You want to deploy a new version to a small subset of users before rolling it out to everyone. You need a canary deployment using Helm.

**Environment:**
- Existing release `web-frontend` serving production traffic (stable version)
- New version `2.0.0` needs canary testing
- Your chart supports `canary.enabled` and `canary.weight` values

**Task:** Deploy a canary release that receives 10% of traffic alongside the stable release.

**Solution:**
```bash
helm install web-frontend-canary myorg/webapp \
  --namespace production \
  --set image.tag=2.0.0 \
  --set canary.enabled=true \
  --set canary.weight=10 \
  --set replicaCount=1
```

The chart's service template would conditionally create a canary Service:
```yaml
{{ if .Values.canary.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "webapp.fullname" . }}-canary
spec:
  selector:
    app.kubernetes.io/instance: {{ .Release.Name }}
    canary: "true"
  ports:
    - port: 80
{{ end }}
```

Monitor the canary:
```bash
helm status web-frontend-canary
kubectl logs -n production -l canary=true --tail=50
```

After validation, promote to stable:
```bash
helm upgrade web-frontend myorg/webapp \
  --set image.tag=2.0.0 \
  --reuse-values \
  --wait
helm uninstall web-frontend-canary
```

---

### Exercise J-5: Blue/Green Deployment Pattern

**Scenario:** You need zero-downtime deployments by running two complete environments (blue and green) and switching traffic between them.

**Environment:**
- Blue environment: release `app-blue` serving all production traffic
- Green environment: release `app-green` ready to be activated
- Ingress or Service selector points to the active environment

**Task:** Deploy a new version to the green environment, validate it, then switch production traffic from blue to green.

**Solution:**
```bash
helm upgrade --install app-green myorg/application \
  --namespace production \
  --set image.tag=3.0.0 \
  --set environment=green \
  --set service.selector.environment=green \
  -f prod-values.yaml \
  --wait

kubectl port-forward -n production svc/app-green 8080:80 &
curl http://localhost:8080/health
kill %1

kubectl patch service app-production -n production \
  -p '{"spec":{"selector":{"app.kubernetes.io/name":"app","environment":"green"}}}'

helm status app-green --namespace production

kubectl get pods -n production -l environment=green

helm uninstall app-blue --namespace production
```

The workflow:
1. Deploy green environment with new version
2. Run validation tests against the green service directly
3. Switch the production Service selector from `blue` to `green`
4. Monitor green for errors
5. Remove blue (or keep as rollback target)

---

### Exercise J-6: Deploy to Multiple Environments

**Scenario:** You need to deploy the same application to `dev`, `staging`, and `production` environments, each with different configurations.

**Environment:**
- Three environments with different resource requirements, replica counts, and feature flags
- Values files: `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`
- Common base: `values-common.yaml`

**Task:** Deploy the application to all three environments using environment-specific configurations.

**Solution:**

Deploy to dev:
```bash
helm upgrade --install myapp-dev myorg/app \
  --namespace dev \
  --create-namespace \
  -f values-common.yaml \
  -f values-dev.yaml \
  --wait
```

Deploy to staging:
```bash
helm upgrade --install myapp-staging myorg/app \
  --namespace staging \
  --create-namespace \
  -f values-common.yaml \
  -f values-staging.yaml \
  --wait
```

Deploy to production:
```bash
helm upgrade --install myapp-prod myorg/app \
  --namespace production \
  --create-namespace \
  --version 2.0.0 \
  -f values-common.yaml \
  -f values-prod.yaml \
  --atomic \
  --wait \
  --timeout 15m
```

Key points:
- `values-common.yaml` is loaded first, environment-specific files override it
- `--create-namespace` avoids pre-creating namespaces
- `--version` is pinned in production, flexible in dev/staging
- `--atomic` is used in production for safety
- `--upgrade --install` makes the commands idempotent across CI runs

---

### Exercise J-7: Handle Secrets Properly

**Scenario:** Your application requires database credentials and API keys. You must not store secrets in plaintext values files or pass them on the command line (which appears in shell history).

**Environment:**
- External Secrets Manager (e.g., Vault) already created Kubernetes Secrets
- Secret `app-db-creds` exists in the target namespace with keys `username` and `password`
- Chart supports referencing external secrets

**Task:** Deploy the application referencing the existing Kubernetes Secret without exposing secrets in Helm values.

**Solution:**

Approach 1 – Reference existing secret in values:
```bash
helm upgrade --install myapp myorg/app \
  --set existingSecret.name=app-db-creds \
  --set db.secretKeys.username=username \
  --set db.secretKeys.password=password \
  -f values-prod.yaml
```

The chart template references the secret:
```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: {{ .Values.existingSecret.name }}
        key: {{ .Values.db.secretKeys.username }}
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.existingSecret.name }}
        key: {{ .Values.db.secretKeys.password }}
```

Approach 2 – Use the Secrets Store CSI Driver:
```bash
helm upgrade --install myapp myorg/app \
  --set secretsStoreCSI.enabled=true \
  --set secretsStoreCSI.provider=vault \
  --set secretsStoreCSI.secretObjects[0].secretName=app-db-creds \
  --set secretsStoreCSI.secretObjects[0].data[0].key=username
```

Best practices:
- Never put secrets in `values.yaml` or `--set` arguments
- Use Kubernetes Secret resources created by external tools
- Use a secrets management operator (External Secrets Operator, Vault, Sealed Secrets)
- Use `.helmignore` to exclude files that might contain secrets
- Encrypt values files with `sops` or `helm-secrets` plugin if they must contain sensitive data

---

### Exercise J-8: OCI Push and Deploy

**Scenario:** You have built and packaged a chart. You need to push it to an OCI registry and then deploy from that registry into a cluster.

**Environment:**
- Chart packaged as `myapp-1.0.0.tgz`
- OCI registry: `registry.internal.example.com`
- You have registry credentials

**Task:** Push the chart to the OCI registry, then deploy it from the registry.

**Solution:**

Step 1 – Login to the OCI registry:
```bash
helm registry login registry.internal.example.com \
  --username helm-publisher \
  --password "$REGISTRY_TOKEN"
```

Step 2 – Push the chart:
```bash
helm push myapp-1.0.0.tgz oci://registry.internal.example.com/charts
```

Step 3 – Verify the chart is published:
```bash
helm show chart oci://registry.internal.example.com/charts/myapp --version 1.0.0
```

Step 4 – Deploy from the OCI registry:
```bash
helm install myapp-prod oci://registry.internal.example.com/charts/myapp \
  --version 1.0.0 \
  --namespace production \
  --create-namespace \
  -f prod-values.yaml \
  --atomic \
  --wait \
  --timeout 15m
```

The `helm push` command is available as a built-in in Helm v3.8+. OCI registries provide better scalability, cross-cloud portability, and native integration with existing container image toolchains.

---

### Exercise J-9: Set Up a CI/CD Pipeline Step

**Scenario:** You are configuring a GitHub Actions workflow that needs to lint, package, and deploy a Helm chart when changes are pushed to `main`.

**Environment:**
- GitHub repository with a Helm chart at `charts/myapp/`
- Kubernetes cluster accessible via a kubeconfig stored as a GitHub Secret
- Target namespace: `production`

**Task:** Write the CI/CD pipeline step(s) for Helm deployment.

**Solution:**

GitHub Actions workflow (`.github/workflows/deploy.yml`):
```yaml
name: Deploy Helm Chart

on:
  push:
    branches: [main]
    paths:
      - 'charts/myapp/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.14.0

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Lint Chart
        run: helm lint charts/myapp/

      - name: Package Chart
        run: |
          helm package charts/myapp/ --destination .packages/
          echo "CHART_ARCHIVE=$(ls .packages/*.tgz)" >> $GITHUB_ENV

      - name: Push to OCI Registry
        run: |
          helm registry login ${{ vars.REGISTRY_HOST }} \
            --username ${{ vars.REGISTRY_USER }} \
            --password "${{ secrets.REGISTRY_PASSWORD }}"
          helm push ${{ env.CHART_ARCHIVE }} oci://${{ vars.REGISTRY_HOST }}/charts

      - name: Deploy to Production
        run: |
          helm upgrade --install myapp-prod \
            oci://${{ vars.REGISTRY_HOST }}/charts/myapp \
            --version ${{ github.ref_name }} \
            --namespace production \
            --create-namespace \
            --values charts/myapp/values-prod.yaml \
            --atomic \
            --wait \
            --timeout 15m

      - name: Verify Deployment
        run: |
          helm status myapp-prod --namespace production
          helm history myapp-prod --namespace production --max 3
```

This pipeline:
1. Checks out the code on push to main
2. Sets up Helm and kubectl
3. Configures cluster access from a secret
4. Lints the chart for errors
5. Packages the chart
6. Pushes to an OCI registry
7. Deploys to production with safety flags
8. Verifies the deployment

---

### Exercise J-10: Disaster Recovery — Restore from Release History

**Scenario:** A cluster outage caused a namespace to be accidentally deleted. All resources in the `production` namespace are gone, but the Helm release history (stored as Secrets) survived in a backup. You need to restore the release.

**Environment:**
- Production namespace was deleted and recreated
- Helm release Secrets for `critical-api` still exist in the cluster (if namespace was restored from backup)
- The release used `--keep-history` during previous operations

**Task:** Restore the `critical-api` release to its last known state.

**Solution:**

Step 1 – Check if release history exists:
```bash
helm history critical-api --namespace production
```

If history exists, rollback to the last deployed revision:
```bash
LAST_REV=$(helm history critical-api --namespace production --max 1 -o json | jq -r '.[0].revision')
helm rollback critical-api $LAST_REV --namespace production --wait --timeout 10m
```

Step 2 – If history Secrets were lost but you have a manifest backup:
```bash
helm install critical-api myorg/api \
  --namespace production \
  -f /backups/critical-api-values.yaml \
  --version 2.1.0 \
  --atomic \
  --wait
```

Step 3 – If the entire cluster was lost, restore from external backup:
```bash
kubectl apply -f /backups/helm-release-secrets/critical-api.yaml -n production

helm rollback critical-api --namespace production --wait
```

Disaster recovery best practices:
- Regularly back up Helm release Secrets:
  ```bash
  kubectl get secrets -n production -l owner=helm -o yaml > helm-releases-backup.yaml
  ```
- Export values and manifests periodically:
  ```bash
  helm get values critical-api -n production --all > backup-values.yaml
  helm get manifest critical-api -n production > backup-manifest.yaml
  ```
- Use `--keep-history` when uninstalling temporarily
- Store backups in an external object store (S3, GCS)

---

### Exercise J-11: Production Troubleshooting Scenario

**Scenario:** A production release `payment-gateway` is not processing requests after an upgrade. The upgrade appeared to succeed (`STATUS: deployed`), but health checks fail.

**Environment:**
- Release `payment-gateway` was upgraded from v1.5.0 to v1.6.0
- Upgrade used `--wait` and completed without errors
- Application logs show `Connection refused` to the database

**Task:** Systematically diagnose and fix the issue using Helm and Kubernetes tooling.

**Solution:**

Step 1 – Assess the current state:
```bash
helm status payment-gateway --namespace production
helm history payment-gateway --namespace production
helm get values payment-gateway --namespace production --all > current-values.yaml
```

Step 2 – Compare with the previous revision:
```bash
helm get manifest payment-gateway --namespace production --revision <previous>
helm get manifest payment-gateway --namespace production --revision <current>
diff <(helm get manifest payment-gateway --revision <previous>) \
     <(helm get manifest payment-gateway --revision <current>)
```

Step 3 – Check the application state:
```bash
kubectl get pods -n production -l app.kubernetes.io/instance=payment-gateway
kubectl describe pod -n production -l app.kubernetes.io/instance=payment-gateway
kubectl logs -n production -l app.kubernetes.io/instance=payment-gateway --tail=100
kubectl get events -n production --sort-by='.lastTimestamp'
```

Step 4 – Diagnose the `Connection refused` error:
```bash
kubectl get svc -n production
kubectl get endpoints -n production -l app.kubernetes.io/instance=payment-gateway
```

Common causes and fixes:

Issue: Database hostname changed between versions.
```bash
helm upgrade payment-gateway myorg/gateway \
  --set database.host=postgresql-primary.production.svc.cluster.local \
  --namespace production \
  --reuse-values \
  --wait
```

Issue: Database credentials mapping changed in chart template.
```bash
helm get manifest payment-gateway --namespace production | grep -A5 DB_HOST
```
Fix by providing the correct secret reference:
```bash
helm upgrade payment-gateway myorg/gateway \
  --set existingSecret.name=payment-db-creds-v2 \
  --namespace production \
  --reuse-values \
  --wait
```

Issue: New version needs a newer database schema (missing migration).
```bash
helm rollback payment-gateway --namespace production --wait
```
Then run the migration separately before re-attempting the upgrade:
```bash
kubectl create job --from=cronjob/db-migration migration-manual -n production
kubectl wait --for=condition=complete job/migration-manual -n production --timeout=5m
helm upgrade payment-gateway myorg/gateway --version 1.6.0 ...
```

Diagnostic commands summary:
| Command | Purpose |
|---|---|
| `helm status` | Overall release health |
| `helm history` | Revision timeline |
| `helm get values --all` | Effective values |
| `helm get manifest --revision N` | Manifest at specific revision |
| `kubectl describe pod` | Pod events and conditions |
| `kubectl logs` | Application output |
| `kubectl get events` | Cluster-level events |
| `kubectl get endpoints` | Service backend health |

---

## Summary by Category

| Category | Exercises | Key Skills |
|---|---|---|
| A: Chart Installation | 16 | `install`, `--set`, `--values`, `--atomic`, `--wait`, OCI, local charts |
| B: Release Upgrade | 12 | `upgrade`, `--reuse-values`, `--reset-values`, `--install`, `--force`, `--dry-run` |
| C: Rollback | 8 | `rollback`, `history`, `--dry-run`, `--wait`, revision management |
| D: Repository Management | 10 | `repo add/remove/update/list`, `search repo/hub`, auth, TLS, indexing |
| E: Chart Inspection | 10 | `show chart/values/readme/all`, `get values/manifest/hooks/notes` |
| F: Chart Development | 10 | `create`, `lint`, `package`, `pull`, `dependency update/list/build`, verification |
| G: Templating | 8 | `template`, `--show-only`, `--api-versions`, `--kube-version`, `--debug` |
| H: Debugging | 8 | `--dry-run --debug`, hook debugging, RBAC, immutable fields, dependency issues |
| I: Release Management | 8 | `list`, `status`, `uninstall`, `--all-namespaces`, `--selector`, `--keep-history` |
| J: Production | 11 | Best practices, canary, blue/green, multi-environment, secrets, OCI, CI/CD, DR |

**Total: 105 exercises**

Use `helm help` and `helm help <command>` for authoritative reference on any of the flags and subcommands used in these exercises.
