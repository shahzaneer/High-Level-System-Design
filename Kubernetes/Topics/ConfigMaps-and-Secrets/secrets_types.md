There are **many built-in Secret types** in Kubernetes, but in practice we'll use only a handful.

## 1. Opaque (Most Common)

This is the default type.

Used for:

* Database passwords
* API keys
* Tokens
* Connection strings
* Environment variables

Example:

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123!
```

YAML:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
data:
  username: YWRtaW4=
  password: U3VwZXJTZWNyZXQxMjMh
```

> `kubectl create secret generic` creates an **Opaque** Secret.

---

## 2. TLS Secret

Specifically for TLS certificates.

Contains:

* Certificate
* Private Key

Example:

```bash
kubectl create secret tls api-tls \
  --cert=fullchain.pem \
  --key=privkey.pem
```

YAML:

```yaml
type: kubernetes.io/tls
```

Data:

```text
tls.crt
tls.key
```

Commonly used by:

* Ingress Controllers
* NGINX
* Traefik
* Gateway API

---

## 3. Docker Registry Secret

Stores credentials for pulling private container images.

Example:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass
```

Type:

```yaml
type: kubernetes.io/dockerconfigjson
```

Usually attached like:

```yaml
imagePullSecrets:
- name: regcred
```

Used for:

* Docker Hub Private
* GitHub Container Registry
* Amazon ECR
* Google Artifact Registry
* Azure Container Registry

---

## 4. Basic Authentication Secret

Stores username/password.

```yaml
type: kubernetes.io/basic-auth
```

Keys:

```text
username
password
```

Example:

```yaml
stringData:
  username: admin
  password: secret
```

---

## 5. SSH Authentication Secret

Stores SSH private keys.

```yaml
type: kubernetes.io/ssh-auth
```

Key:

```text
ssh-privatekey
```

Useful for:

* Git access
* SSH automation
* CI/CD pipelines

---

## 6. Service Account Token Secret

Automatically created (or requested) for Service Accounts.

```yaml
type: kubernetes.io/service-account-token
```

Contains:

* JWT token
* CA certificate
* Namespace

Modern Kubernetes versions typically use **projected service account tokens** instead of long-lived Secret objects, but you should still recognize this Secret type.

---

## 7. Bootstrap Token Secret

Used when joining worker nodes to the cluster with `kubeadm`.

Type:

```yaml
bootstrap.kubernetes.io/token
```

Mostly relevant to cluster administrators.

---

# Other Built-in Secret Types

Kubernetes also defines some specialized Secret types:

| Type                                  | Purpose                         |
| ------------------------------------- | ------------------------------- |
| `Opaque`                              | Generic secret (default)        |
| `kubernetes.io/tls`                   | TLS certificates                |
| `kubernetes.io/dockerconfigjson`      | Docker/OCI registry credentials |
| `kubernetes.io/basic-auth`            | Username/password               |
| `kubernetes.io/ssh-auth`              | SSH private key                 |
| `kubernetes.io/service-account-token` | Service account token           |
| `bootstrap.kubernetes.io/token`       | `kubeadm` bootstrap token       |

---


Focus on these four:

| Secret Type               | How it's created                        | Common Use                               |
| ------------------------- | --------------------------------------- | ---------------------------------------- |
| **Opaque**                | `kubectl create secret generic`         | Database passwords, API keys, tokens     |
| **TLS**                   | `kubectl create secret tls`             | Ingress, HTTPS certificates              |
| **Docker Registry**       | `kubectl create secret docker-registry` | Pulling images from private registries   |
| **Service Account Token** | Usually managed by Kubernetes           | Pod authentication to the Kubernetes API |


### A useful mental shortcut

Think of Secrets in two categories:

* **General-purpose:** `Opaque` (stores arbitrary key/value pairs)
* **Special-purpose:** Everything else (TLS, Docker registry, SSH, Basic Auth, Service Account, Bootstrap), where Kubernetes expects specific keys and formats and can validate or use them accordingly.
