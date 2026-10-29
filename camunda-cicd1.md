# Camunda 8.8 — Git → deploy + replace Web Modeler (atomic release)

**Goal:** Git is the source of truth. On merge/release, the pipeline:

1. **Deploys** all process resources from the repo to the Camunda 8.8 cluster (atomic).
2. **Replaces** the matching diagrams in Web Modeler with the exact files from Git (so Modeler matches what runs).

```
Git repo (BPMN/DMN/forms)
        │
        ├──────────────────────► Cluster deploy   (POST /deployments — atomic)
        │
        └──────────────────────► Web Modeler replace (PATCH files from Git)
```

---

## Why this pattern

| Concern | Approach |
|---------|----------|
| Source of truth | Git (PRs, reviews, history) |
| What the engine runs | Cluster deployment from Git |
| What people see in Web Modeler | Overwritten from Git after a successful deploy |
| Accidental Modeler edits | Treated as drafts; next pipeline run overwrites them (or disable Modeler deploy) |

Built-in [Git sync](https://docs.camunda.io/docs/components/modeler/web-modeler/process-applications/git-sync/) is UI-driven. For GitHub Actions you drive the same idea with the **Web Modeler API** + **Orchestration Cluster API**.

**Deeper reading:**
- [Integrate Web Modeler into CI/CD (8.8)](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)
- [Git sync (process applications)](https://docs.camunda.io/docs/components/modeler/web-modeler/process-applications/git-sync/)
- [Web Modeler API overview](https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/)

---

## Credentials

| Secret / client | Used for |
|-----------------|----------|
| Cluster / Zeebe OIDC client | `POST /deployments` or `c8 deploy` |
| Web Modeler API client (`update` permission) | List + `PATCH` / `POST` files |

**Deeper reading:**
- [Cluster API authentication](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/orchestration-cluster-api-rest-authentication/)
- [Web Modeler API authentication](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/)
- [c8ctl getting started](https://docs.camunda.io/docs/apis-tools/c8ctl/getting-started/)

Store both as GitHub Actions secrets. Optionally disable one-click deploy from Modeler so only CI deploys (Self-Managed: `ZEEBE_BPMN_DEPLOYMENT_ENABLED` / `ZEEBE_DMN_DEPLOYMENT_ENABLED`) — see the [CI/CD guide](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/).

---

## Step 1 — Atomic deploy to the cluster

Cluster deploy is **truly atomic**: either all resources in the request succeed, or none do.

```bash
# Directory of BPMN/DMN/forms from the repo
c8 deploy ./processes --profile=prod --yes
```

Or REST: [`POST /deployments`](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/) with every file that belongs to this release in **one** request.

**Deeper reading:**
- [Create deployment](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/)
- [c8ctl deploy workflows](https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/)

**Pipeline rule:** only sync Web Modeler **after** this step succeeds. That way Modeler never shows a version the cluster rejected.

---

## Step 2 — Replace Web Modeler files from Git (“atomic” update)

Web Modeler has **no multi-file transaction API**. Updates are per file via `PATCH`, with optimistic locking (`revision`).

To get an **atomic release feel** in CI:

1. Maintain a **mapping** Git path → Modeler `fileId` (and `projectId` / process application).
2. For each mapped file: `GET` current metadata → read `revision` → `PATCH` with Git content + that `revision`.
3. Fail the job if **any** PATCH fails (conflict / 409 / network). Do not partially “declare success”.
4. Optionally create Modeler **versions** after all PATCHes succeed, so the synced state is labeled.

OpenAPI / Swagger: [modeler.camunda.io/swagger-ui](https://modeler.camunda.io/swagger-ui/index.html)  
SDK reference: [`updateFile`](https://camunda.github.io/camunda-8-js-sdk/classes/index.Modeler.ModelerApiClient.html)

### 2.1 Discover / map files

List files in the Modeler project (paginate all pages):

```http
POST /api/v1/files/search
Authorization: Bearer <MODELER_TOKEN>
Content-Type: application/json

{
  "filter": { "projectId": "<PROJECT_ID>" },
  "page": 0,
  "size": 50
}
```

Match by `name` / `simplePath` to repo paths, or keep an explicit map in the repo, e.g. `modeler-sync.json`:

```json
{
  "projectId": "fb928277-6268-44bb-b3e6-1925fa730ecf",
  "files": [
    { "path": "processes/order.bpmn", "fileId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" },
    { "path": "processes/pricing.dmn", "fileId": "ffffffff-1111-2222-3333-444444444444" }
  ]
}
```

**Deeper reading:** [Integrate Web Modeler into CI/CD — sync files](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)

### 2.2 Replace one file (overwrite with Git content)

```http
GET /api/v1/files/<fileId>
Authorization: Bearer <MODELER_TOKEN>
```

Response includes `revision` and current `content`. Then:

```http
PATCH /api/v1/files/<fileId>
Authorization: Bearer <MODELER_TOKEN>
Content-Type: application/json

{
  "revision": 12,
  "content": "<exact XML/JSON string from the Git file>",
  "name": "order.bpmn"
}
```

- `revision` **must** be the current value from `GET`. Wrong revision → conflict (someone edited in Modeler meanwhile).
- On conflict in CI: fail, or retry once with a fresh `GET` if you intentionally force Git over Modeler.

**Deeper reading:**
- [patch File (Postman)](https://www.postman.com/camundateam/camunda-8-postman/request/t5u0rh0/patch-file)
- [Web Modeler API tutorial](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/tutorial/)

### 2.3 Create missing files / remove orphans (optional)

| Situation | Action |
|-----------|--------|
| File in Git, not in Modeler | `POST /api/v1/files` with `name`, `projectId`, `content`, `fileType` (`BPMN` / `DMN` / …); save returned `id` into the map |
| File in Modeler, not in Git | Decide policy: leave alone, or `DELETE` (irreversible — API does not warn) |

Create example shape (see also process-application fields on newer versions):

```http
POST /api/v1/files
{
  "name": "order.bpmn",
  "projectId": "<PROJECT_ID>",
  "content": "...",
  "fileType": "BPMN"
}
```

**Deeper reading:** [Process application notes for Web Modeler API v1](https://docs.camunda.io/docs/apis-tools/migration-manuals/web-modeler-v1-apis/) (e.g. `processApplicationId` vs `folderId` on newer platforms)

### 2.4 Make the Modeler update “all or nothing” in practice

Recommended CI logic:

```text
1. Load mapping + Git file contents
2. GET all target files → collect revisions (abort if any missing without create-policy)
3. PATCH every file with Git content
4. If any PATCH fails → fail job (Modeler may be partially updated; re-run after fixing)
5. On full success → optionally POST /api/v1/versions for each file (label = git SHA / tag)
6. Only then mark the GitHub job green for "modeler-sync"
```

There is no server-side rollback of a partial PATCH set. Keeping the set small (one process application / folder per pipeline) and failing loudly is the practical “atomic” strategy. The **cluster** side remains the hard atomic boundary via a single `POST /deployments`.

---

## Suggested GitHub Actions flow

```yaml
# Conceptual — secrets and auth omitted
on:
  push:
    branches: [main]
    paths: ['processes/**', 'modeler-sync.json']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint
        run: npx bpmnlint processes/**/*.bpmn   # add dmnlint as needed

      - name: Deploy to cluster (atomic)
        run: c8 deploy ./processes --profile=prod --yes

      - name: Replace Web Modeler from Git
        run: python scripts/sync_modeler_from_git.py
        # script: map paths → fileIds, GET revision, PATCH content; exit non-zero on any failure
```

Order matters: **deploy first, then overwrite Modeler**. If deploy fails, leave Modeler unchanged.

Community helper for deploy only: [camunda-platform-8-github-action](https://github.com/camunda-community-hub/camunda-platform-8-github-action) — Modeler replace is custom script/curl against the Modeler API.

---

## Operational tips

- **Force Git over Modeler:** treat Modeler as a mirror. On `revision` conflict, re-GET and PATCH again (or block Modeler edits via process/governance).
- **One mapping per environment:** same Git files can sync to different Modeler projects (dev vs shared design space) if needed; cluster profiles stay separate.
- **Version label:** after sync, create Modeler versions named with the Git commit SHA so UI history matches Git.
- **Payload size:** built-in Git sync has size limits (e.g. SaaS ~4MB); large apps may need file-by-file API updates anyway — see [Git sync limitations](https://docs.camunda.io/docs/components/modeler/web-modeler/process-applications/git-sync/).

---

## Link index

| Topic | Link |
|-------|------|
| Web Modeler + CI/CD (8.8) | https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/ |
| Git sync (UI) | https://docs.camunda.io/docs/components/modeler/web-modeler/process-applications/git-sync/ |
| Web Modeler API overview | https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/ |
| Web Modeler API auth | https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/ |
| Web Modeler API tutorial | https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/tutorial/ |
| OpenAPI (Swagger) | https://modeler.camunda.io/swagger-ui/index.html |
| patch File | https://www.postman.com/camundateam/camunda-8-postman/request/t5u0rh0/patch-file |
| Process application / API v1 notes | https://docs.camunda.io/docs/apis-tools/migration-manuals/web-modeler-v1-apis/ |
| Create deployment (atomic) | https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/ |
| Cluster API auth | https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/orchestration-cluster-api-rest-authentication/ |
| c8ctl deploy | https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/ |
| GitHub Action (deploy) | https://github.com/camunda-community-hub/camunda-platform-8-github-action |
