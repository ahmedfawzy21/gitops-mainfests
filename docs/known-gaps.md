# Known gaps — movies-app data path

Three gaps currently stop the loader → SQS → parser → MongoDB → store path from
working in-cluster. They are documented here and are **not** implemented yet.
Each one needs infrastructure work (IAM, Terraform outputs, secret delivery)
that is outside this repo.

Data flow these gaps affect:

```
movies-loader --SendMessage--> SQS movies-loader-<env> --Receive/Delete--> movies-parser
movies-parser --InsertOne--> MongoDB (mongodb-service.movies-app:27017) <--find-- movies-store
```

Already fixed on branch `config-fixes`:

- movies-parser now receives `HEALTH_PORT`, `MONGO_URI` and `MONGO_DATABASE`,
  the names its code reads.
- A `movies-store` Service now exists, so the marketplace NGINX `/api/` proxy resolves.

---

## Gap 1 — `SQS_URL` is not set for movies-loader or movies-parser

**Where:** `base/movies-loader-rollout.yaml` and `base/movies-parser-rollout.yaml`
(`env:` blocks). Both services read `SQS_URL`
(`movies-loader/main.py:68`, `movies-parser/main.go:202,318`).

**Effect:** Neither service touches SQS. When `SQS_URL` is empty:

- the loader logs `SQS_URL not set, running in mock mode` and sends nothing
  (`main.py:70-73`);
- the parser logs `SQS_URL not set, running in health-check only mode` and never
  connects to SQS or MongoDB (`main.go:318-322`).

Both pods stay Ready, so nothing alerts on this.

**Planned fix:** Take the URL from Terraform outputs; don't hardcode it.

- The `sqs` module already exports `movies_loader_queue_url`
  (`terragrunt-structure/modules/sqs/outputs.tf`). Loader and parser must both use
  this **same** queue. The loader writes to it and the parser consumes it.
- Publish the output into the cluster, e.g. a Terraform-managed SSM parameter
  `/movies-app/<env>/sqs_url` synced by External Secrets into
  `movies-app-config` / `movies-app-secrets`. Reference it with `configMapKeyRef`
  (or `secretKeyRef`) as `SQS_URL` in both rollouts.
- Separately, decide what to do with `movies-parser-<env>`
  (`create_parser_queue`, default `true`). Nothing writes to it today. Either
  drop it (`create_parser_queue = false`) or give it a real producer.

## Gap 2 — no AWS identity for SQS calls

**Where:** No rollout sets `serviceAccountName`, so the pods run as the
namespace `default` ServiceAccount with no AWS role. The IAM policy
`<project>-<env>-sqs-access` exists (`modules/sqs/main.tf:100-135`) but is not
attached to any workload identity.

**Effect:** After gap 1 is fixed, boto3 (loader) and aws-sdk-go-v2 (parser) will
fall back to the node instance role, or fail with `NoCredentialProviders` /
`AccessDenied`. Either outcome is wrong: the node role should not hold
application SQS permissions.

**Planned fix:** EKS Pod Identity.

- Install the `eks-pod-identity-agent` add-on on the cluster (Terraform, `eks` module).
- Create one IAM role per service, trusted by `pods.eks.amazonaws.com`, with
  least-privilege policies:
  - movies-loader: `sqs:SendMessage` on the loader queue only.
  - movies-parser: `sqs:ReceiveMessage`, `sqs:DeleteMessage`,
    `sqs:ChangeMessageVisibility` and `sqs:GetQueueAttributes` on the loader queue only.

  This replaces the single shared `sqs_access` policy.
- Add `aws_eks_pod_identity_association` resources mapping
  `movies-app/movies-loader` and `movies-app/movies-parser` ServiceAccounts to those roles.
- In this repo, add the two ServiceAccounts and set `serviceAccountName` on both
  rollouts. No annotations are needed with Pod Identity. Both SDKs pick up the
  credentials automatically.

## Gap 5 — `MONGO_URI` key is not delivered to `movies-app-secrets`

**Where:** movies-store requires `secretKeyRef: movies-app-secrets / MONGO_URI`
(`base/movies-store-rollout.yaml`). movies-parser now references the same key
(marked `optional: true`). Neither `k8s/base/secrets.yaml` nor
`k8s/base/external-secrets.yaml` in the infra repo defines a `MONGO_URI` key.

**Effect:**

- movies-store: if the key is missing, the pod fails with
  `CreateContainerConfigError` and never starts. If the key comes from some
  manual secret, it isn't tracked in Git and can drift.
- movies-parser: it starts (the key is optional) but would have no database once
  `SQS_URL` is set. Remove `optional: true` when this gap is closed.

**Planned fix:** Deliver `MONGO_URI` via External Secrets.

- Store the connection string in AWS Secrets Manager. With DocumentDB this
  already exists as `<project>/mongodb/<env>` → `connection_string`
  (`modules/mongodb/main.tf:122-143`). For the in-cluster StatefulSet, create the
  equivalent secret with
  `mongodb://<user>:<pass>@mongodb-service.movies-app.svc.cluster.local:27017/movies?authSource=admin`.
- Add an `ExternalSecret` in this repo targeting `movies-app-secrets` that maps
  that secret's `connection_string` property to the `MONGO_URI` key. Use the
  existing ClusterSecretStore, which authenticates with External Secrets'
  own Pod Identity / IRSA role.
- The URI must include the database name (`/movies`). movies-store does not set
  a database explicitly (`dao.js:29`) and falls back to `test` without it.
- When moving to DocumentDB, the URI must keep `tls=true&retryWrites=false`, and
  the images need the Amazon RDS CA bundle.

---

## Also found (not yet fixed): every service renders the movies-loader image

**Where:** `overlays/{dev,staging,prod}/kustomization.yaml` `images:`. All four
rollouts use `image: IMAGE_PLACEHOLDER`, and each overlay lists four `images:`
entries with that same `name`. Kustomize applies the first match, so
`kustomize build overlays/<env>` gives movies-loader, movies-parser,
movies-store and movies-marketplace the image `.../movies-loader:<env>-latest`.

**Effect:** Any sync from these overlays as written deploys the wrong
container for three of the four services. Whatever is running now must have
been set some other way (CI `kustomize edit` / `kubectl set image`), and Git
does not reflect it.

**Planned fix:** Give each rollout a unique placeholder (e.g.
`image: movies-parser`) and key each overlay's `images:` entry on that name.
