# Crossplane GCP Service Accounts

This Crossplane project provisions Google Cloud service accounts with IAM role
bindings and optional key generation.

## What it does

A claim (`ServiceAccountClaim`) provisions:

- A Google Cloud **service account**.
- Up to **two IAM role bindings** on that service account (`roles[0]` and
  `roles[1]`).
- A **service account key**, whose private key is written to a Kubernetes
  Secret (referenced by `writeConnectionSecretToRef`).

The service account email, unique ID, and the key Secret reference are
published as outputs on the claim's `status`.

> **Fixed vs dynamic.** The included Composition is a *fixed example*: it
> provisions a single service account with up to two role bindings. For
> dynamic cases (many service accounts, an arbitrary number of roles, or a
> map of accounts to roles), use the direct managed resources in
> [`examples/service-accounts.yaml`](examples/service-accounts.yaml) as a
> starting point and reference them from your own Composition.

## Free tier

Google Cloud does **not charge** for service accounts or IAM role bindings.
Keep in mind that service account keys are long-lived credentials: prefer
Workload Identity or short-lived credentials where possible, and rotate keys
regularly.

## Prerequisites

- A GCP project with a service account that has sufficient IAM permissions to
  create service accounts, bind IAM roles, and create keys (e.g. `roles/iam.serviceAccountAdmin`
  plus `roles/iam.roleAdmin` for role bindings).
- A Kubernetes cluster with [Crossplane](https://www.crossplane.io) installed.
- `kubectl` configured to talk to the cluster.

## Project structure

```text
.
├── .github/workflows/
│   ├── crossplane-ci.yml            # CI: lint + validate manifests (ryl)
│   ├── crossplane-cd-apply.yml      # CD: deploy the claim (workflow_dispatch)
│   └── crossplane-cd-destroy.yml    # CD: tear down the claim
├── provider/
│   ├── provider.yaml                # upbound-provider-gcp-iam (v3.0.1)
│   ├── provider-config.yaml         # gcp-provider-config (PROJECT_ID placeholder)
│   └── credentials-secret.yaml      # K8s Secret template for GCP credentials
├── composition/
│   ├── xrd.yaml                     # ServiceAccount XRD + ServiceAccountClaim
│   ├── composition.yaml             # ServiceAccount + IAM roles + key
│   └── claim.yaml                   # Example claim (placeholders)
├── examples/
│   └── service-accounts.yaml        # Direct managed resources (dynamic cases)
├── .gitignore
├── .secrets.example
├── .yamllint
└── README.md
```

## Quick start

### 1. Install the provider

```bash
kubectl apply -f provider/provider.yaml
kubectl wait --for=condition=Healthy provider/upbound-provider-gcp-iam --timeout=300s
```

### 2. Configure GCP credentials

Create a service account key for a GCP account with the permissions above and
store it in a Secret named `gcp-creds` in the `crossplane-system` namespace:

```bash
gcloud iam service-accounts keys create creds.json \
  --iam-account=your-sa@your-project.iam.gserviceaccount.com

kubectl create secret generic gcp-creds \
  --namespace crossplane-system \
  --from-file=creds=creds.json
```

Then apply the ProviderConfig, replacing `PROJECT_ID`:

```bash
sed -e "s/PROJECT_ID/your-project/g" provider/provider-config.yaml | kubectl apply -f -
```

### 3. Install the XRD and Composition

```bash
kubectl apply -f composition/xrd.yaml
kubectl apply -f composition/composition.yaml
```

### 4. Create a claim

Edit `composition/claim.yaml` and replace the placeholders:

| Field        | Description                                        |
| ------------ | -------------------------------------------------- |
| `projectId`  | GCP project ID where resources are created.        |
| `region`     | GCP region (Free Tier: `us-west1`, `us-central1`, `us-east1`). |
| `accountId`  | Account ID of the service account.                 |
| `displayName`| Display name of the service account.               |
| `description`| Description of the service account.                |
| `roles`      | IAM roles to bind (`roles[0]`, `roles[1]`).        |

```bash
kubectl apply -f composition/claim.yaml
kubectl wait --for=condition=Ready \
  serviceaccountclaims.iam.gcp.example.org/example \
  --namespace default --timeout=300s
```

### 5. Read the outputs

```bash
kubectl get serviceaccountclaims.iam.gcp.example.org example -n default \
  -o jsonpath='Email: {.status.serviceAccountEmail}{"\n"}Unique ID: {.status.serviceAccountId}{"\n"}'
```

The generated key (a private key) is stored in a Kubernetes Secret whose name
and namespace are published in `status.serviceAccountKeySecretRef`. To inspect
it:

```bash
kubectl get secret my-sa-key -o jsonpath='{.data.privateKey}' | base64 --decode
```

> The key is a long-lived credential. Store it securely, restrict access to
> the Kubernetes Secret, and rotate it regularly.

## CI/CD

- **CI** (`.github/workflows/crossplane-ci.yml`): runs `ryl check --strict .`
  on pull requests and pushes to `main`.
- **CD Apply** (`.github/workflows/crossplane-cd-apply.yml`): triggered by
  `workflow_dispatch` with the claim inputs (`project_id`, `region`,
  `claim_name`, `claim_namespace`, `account_id`, `display_name`, `description`,
  `role_0`, `role_1`). It installs the provider, the credentials Secret, the
  ProviderConfig, the XRD/Composition, and the claim, then waits for readiness.
  Requires the `KUBECONFIG` and `GCP_SA_KEY` repository Secrets.
- **CD Destroy** (`.github/workflows/crossplane-cd-destroy.yml`): triggered by
  `workflow_dispatch` with the claim name/namespace (optionally removes the
  provider as well).

## Destroy

```bash
kubectl delete serviceaccountclaims.iam.gcp.example.org example -n default
```

This deletes the claim, the service account, its IAM role bindings, and the
generated key.