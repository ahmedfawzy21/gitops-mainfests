# Known gaps — movies-app data path

Gaps 1 and 2 still stop the loader → SQS → parser → MongoDB → store path from
working in-cluster; gap 5 is closed for dev only. They are documented here and,
except where marked, are **not** implemented yet. Each one needs infrastructure
work (IAM, Terraform outputs, secret delivery) that is outside this repo.

Data flow these gaps affect:

```
movies-loader --SendMessage--> SQS movies-loader-<env> --Receive/Delete--> movies-parser
movies-parser --InsertOne--> MongoDB (mongodb.movies-app:27017, dev) <--find-- movies-store
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

**Status — closed for dev, open for staging/prod:** dev runs its own MongoDB
(`overlays/dev/mongodb.yaml`: one node, auth on, `mongo:7.0`). The
terragrunt-structure bootstrap (`scripts/argocd-bootstrap.sh app-secrets`)
reads Secrets Manager `movies-app/dev/mongodb` (`root_password`,
`app_password`) and creates, without printing or storing any value:

- `mongodb-auth`: `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD`,
  `MONGO_APP_PASSWORD`;
- `movies-app-secrets`: `MONGO_URI` =
  `mongodb://movies:<app_password>@mongodb.movies-app.svc.cluster.local:27017/movies?authSource=movies`.

The parser's reference is no longer optional. Notes:

- MongoDB creates the users only on its first start (empty volume). Changing
  a password later is manual: `db.changeUserPassword` in the pod, update the
  Secrets Manager secret, re-run the bootstrap, restart store and parser.
- movies-store uses mongoose 5.9.9 / driver 3.5.5, officially tested up to
  MongoDB 4.4. If it cannot talk to 7.0, use `mongo:6.0` (new volume: a
  data directory can't move back a major version).
- Staging/prod still have no MongoDB and no `MONGO_URI` (and share the
  namespace, gap 6). The External Secrets fix above remains the plan there.

## Gap 6 — all overlays deploy into the same namespace

**Where:** `base/kustomization.yaml` and every overlay set `namespace: movies-app`,
and all three ArgoCD Applications (`apps/movies-app-{dev,staging,prod}.yaml`)
target `movies-app` on the same cluster (`https://kubernetes.default.svc`).

**Effect:**

- Only one environment can run at a time. If the staging or prod Application
  is created next to dev, the Applications fight over the same Rollouts,
  Services and ConfigMap, and each sync undoes the other.
- A "promote staging" step would promote the dev Rollout, because it targets
  `movies-app/<service>`, which is the same object.

**Current state:** Only `apps/movies-app-dev.yaml` is applied (by the bootstrap
in terragrunt-structure, not `kubectl apply -f apps/`). **Staging and prod
promotion are disabled** in the Jenkins pipeline (the shared library stops
after the dev stages) until staging and prod get their own namespaces or
clusters.

**Blocks real staging and prod.** The public entry point is ready for them
by inputs alone (terragrunt-structure `app_hostnames`: `staging.gromaops.dev`;
`gromaops.dev` + `www`; certificate, WAF, ExternalDNS owner
`movies-app-<env>` per environment), but it can't serve them on a shared
cluster or namespace:

- One marketplace Ingress per namespace: the staging or prod Application
  would take over dev's `movies-marketplace` Ingress and swap its hosts.
- IngressClass `alb` belongs to one cluster's controller, with one set of
  IngressClassParams (one environment's certificate, WAF and load balancer
  name), and only lets namespace `movies-app` use it.
- Every environment's Terraform names its EKS cluster `movies-app`
  (`project_name`), so two environments can't both have a cluster in the
  same account and region.

Staging and prod need their **own clusters** (cluster name per environment,
then this repo needs no change for the entry point) **or their own
namespaces** (below, plus one IngressClass and IngressClassParams per
environment and the namespace in its namespaceSelector).

**Planned fix:** One namespace per environment (`movies-app-staging`,
`movies-app-prod`) set in each overlay and Application, with the Jenkins
Role/RoleBinding (`base/jenkins-rbac.yaml`) and the EKS access policy scope
extended to them. A separate cluster per environment is the alternative.
Then re-enable the staging and prod stages in the pipeline.

## Gap 7 — `k8s/base/secrets.yaml` holds a placeholder password in Git

**Where:** `k8s/base/secrets.yaml` in the old raw-manifests tree (`Project3/k8s`,
next to terragrunt-structure). It defines `movies-app-secrets` with
`MONGODB_PASSWORD: "CHANGE_ME_IN_PRODUCTION"` in plain text, and
`k8s/base/mongodb-statefulset.yaml` uses it as the MongoDB root password.

**Effect:** Anyone who applies that tree gets a MongoDB whose root password is
public. Because it also defines `movies-app-secrets`, applying it would replace
the bootstrap-created Secret in dev and drop `MONGO_URI`.

**Planned fix:** Delete the file, or replace it with an `ExternalSecret` (the
External Secrets Operator is not installed yet; Terraform only creates its IRSA
role). Do not use the old StatefulSet either: each pod generates its own
replica-set keyfile in an `emptyDir`, so its members cannot authenticate to
each other. Treat any value ever committed there as burned.

## Gap 8 — Terraform's `movies-app/mongodb` secret keeps its password in state

**Where:** terragrunt-structure `modules/secrets-manager/main.tf`
(`create_mongodb_secret`, default `true`). `random_password.mongodb` generates
the password and `aws_secretsmanager_secret_version.mongodb` writes it, so the
value is in Terraform state in plain text. It also names the database
`moviesdb`, while the apps use `movies`.

**Effect:** Anyone with read access to the state bucket can read the
password. Nothing in dev uses this secret: dev uses `movies-app/dev/mongodb`,
created with the AWS CLI outside Terraform.

**Planned fix:** Set `create_mongodb_secret = false` (Terraform deletes the
secret after its recovery window), or keep only the secret resource in
Terraform and write the value outside it, as for `movies-app/dev/mongodb`.

## Dev: all four services run

Each Rollout runs one replica in dev. A service whose pipeline has not yet
committed an image tag pulls `dev-latest`, which is not in ECR (ECR expires
images after 14 days): its pod stays in `ImagePullBackOff` until the first
`deploy: true` build commits a tag. MongoDB and its Secrets must exist first
(gap 5), because MongoDB's sync-wave -1 holds the whole dev Application until
it is healthy.

---

## Fixed: every service rendered the movies-loader image

**Where:** `overlays/{dev,staging,prod}/kustomization.yaml` `images:`. All four
rollouts use `image: IMAGE_PLACEHOLDER`, and each overlay lists four `images:`
entries with that same `name`. Kustomize applies the first match, so
`kustomize build overlays/<env>` gives movies-loader, movies-parser,
movies-store and movies-marketplace the image `.../movies-loader:<env>-latest`.

**Effect:** Any sync from these overlays as written deploys the wrong
container for three of the four services. Whatever is running now must have
been set some other way (CI `kustomize edit` / `kubectl set image`), and Git
does not reflect it.

**Fix (branch `config-fixes`):** Each rollout now has its own image name
(`movies-loader-image`, `movies-parser-image`, `movies-store-image`,
`movies-marketplace-image`), and each overlay's `images:` entry is keyed on it.
The shared library's `deployToK8sGitOps.groovy` updates `newTag` under
`- name: <service>-image` and fails if the tag was not written.

---

## movies-marketplace: Angular 8 and Node versions are end-of-life

**Where:** movies-marketplace (`package.json`: Angular 8.2, `@angular-devkit/build-angular`
0.803), `Dockerfile` (builder stage `node:16-alpine`) and `Dockerfile.test`
(`node:14-alpine`).

**Effect:**

- Angular 8 has been EOL since 2021 and gets no security fixes.
- Unit tests run on **Node 14 only, as a stopgap**. Angular 8's Karma plugin
  copies requests with `{ ...req }` (`build-angular/.../plugins/karma.js:269`).
  On Node >= 16, `IncomingMessage.headers` is a lazy getter, so the copy has
  no headers. `webpack-dev-middleware` then throws
  `Cannot read properties of undefined (reading 'range')` and the Karma server
  dies mid-run (on Node 20: 4 of 7 specs ran, 0 failed, the rest never ran).
- The production builder stage uses `node:16`, also EOL. Only the test image is
  on Node 14; it's not shipped.
- Node 14 ships npm 6, which can't read the v3 lockfile, so the test image
  installs npm 9.

**Planned fix:** Upgrade Angular one major version at a time (8 → 9 → … → 17+)
with `ng update`, fixing breaking changes and keeping the unit tests green at
each step. Then move every image (builder, test and runtime) to a supported
Node LTS, and remove the Node 14 / npm 9 stopgap from `Dockerfile.test`.
