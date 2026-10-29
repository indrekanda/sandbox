# Camunda 8.8 — Git → test → (approve) syst → (approve) prod + Modeler mirror

**Goal**

- On **merge to `main`**: deploy the Git folder to **test**, then sync Web Modeler so that folder is **identical** to Git.
- Promote the **same Git commit** to **syst** and **prod** only after **approvals** (GitHub Environments).

```
merge → main
   │
   ├─► deploy to TEST (atomic)     ──► then wipe+rewrite Modeler folder from Git
   │
   ├─► approval ──► deploy same commit to SYST
   │
   └─► approval ──► deploy same commit to PROD
```

Git is source of truth. Cluster runs what Git has. Web Modeler is a **mirror** of one designated repo folder (not the other way around).

**Deeper reading:**
- [Integrate Web Modeler into CI/CD (8.8)](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)
- [Create deployment (atomic)](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/)
- [Web Modeler API overview](https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/)
- [OpenAPI / Swagger](https://modeler.camunda.io/swagger-ui/index.html)

---

## Repo layout

Keep one folder that is the unit of deploy + Modeler sync, e.g.:

```text
processes/                 ← identical to Web Modeler folder
  order.bpmn
  pricing.dmn
  start.form
  subprocesses/
    ship.bpmn
```

Config (IDs from Modeler URL / API), e.g. `modeler-sync.json`:

```json
{
  "projectId": "<project-uuid>",
  "folderId": "<folder-uuid>",
  "gitRoot": "processes"
}
```

That Modeler `folderId` is the mirror target. After sync it must contain the same relative paths/names as `processes/` in Git (including subfolders if you recreate them).

---

## Credentials & environments

| GitHub Environment | Cluster client | When |
|--------------------|----------------|------|
| `test` | test cluster creds | auto on merge to `main` |
| `syst` | syst cluster creds | after required reviewers approve |
| `prod` | prod cluster creds | after required reviewers approve |

Plus org-wide (or env-scoped) **Web Modeler** client with create/update/delete on files (and folders if you recreate nested dirs).

**Deeper reading:**
- [Cluster API authentication](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/orchestration-cluster-api-rest-authentication/)
- [Web Modeler API authentication](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/)
- [c8ctl profiles](https://docs.camunda.io/docs/apis-tools/c8ctl/getting-started/)
- [GitHub Environment protection rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

Optional: disable one-click deploy from Modeler so only CI deploys — [CI/CD guide](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/).

---

## Stage A — Deploy to test (on merge to main)

Trigger: `push` to `main` (after PR merge). Deploy **all** resources under `processes/` in **one** atomic call.

```bash
c8 deploy ./processes --profile=test --yes
```

Or [`POST /deployments`](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/) with every file in that folder in a single request.

**Rule:** only run the Modeler wipe/rewrite after this succeeds.

**Deeper reading:** [c8ctl deploy](https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/)

---

## Stage B — Make Web Modeler folder identical to Git (wipe + rewrite)

Strategy: **delete every file in the target Modeler folder, then create all files from Git**. That avoids revision conflicts and leftover orphans; the folder matches Git by construction.

> **API constraint:** `DELETE /api/v1/folders/{folderId}` only works on an **empty** folder. You must delete **files** (and nested empty folders) first — you cannot delete a non-empty folder in one call.  
> Docs note: deletes are **permanent** (files, comments, milestones, shares).  
> See [Web Modeler API usage notes](https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/) and SDK [`deleteFile` / `deleteFolder`](https://camunda.github.io/camunda-8-js-sdk/classes/index.Modeler.ModelerApiClient.html).

### B.1 List everything currently in the folder

```http
POST /api/v1/files/search
Authorization: Bearer <MODELER_TOKEN>
Content-Type: application/json

{
  "filter": {
    "projectId": "<PROJECT_ID>",
    "folderId": "<FOLDER_ID>"
  },
  "page": 0,
  "size": 50
}
```

Paginate until all pages are consumed. For nested Git trees, also list/create subfolders under this root (folders API: create / delete empty).

**Deeper reading:** [CI/CD — sync files](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/), [Swagger](https://modeler.camunda.io/swagger-ui/index.html)

### B.2 Delete all files in that folder (and nested content)

For each file id from search:

```http
DELETE /api/v1/files/<fileId>
Authorization: Bearer <MODELER_TOKEN>
```

If you mirrored subfolders: after files are gone, delete empty nested folders bottom-up with `DELETE /api/v1/folders/<folderId>`, then recreate folder tree from Git paths. **Keep the configured root `folderId`** (empty it; do not delete the root folder you configured in `modeler-sync.json`).

### B.3 Write all files from Git into the (empty) folder

For each file under `processes/` (recursive):

```http
POST /api/v1/files
Authorization: Bearer <MODELER_TOKEN>
Content-Type: application/json

{
  "name": "order.bpmn",
  "projectId": "<PROJECT_ID>",
  "folderId": "<FOLDER_ID_OR_NESTED_FOLDER_ID>",
  "content": "<file contents from Git>",
  "fileType": "BPMN"
}
```

Map extensions → `fileType` (`BPMN`, `DMN`, form types as per OpenAPI). Recreate relative subfolders first if Git has `processes/subprocesses/...`.

**Deeper reading:**
- [create File / create Folder (Postman)](https://www.postman.com/camundateam/camunda-8-postman/collection/e62j83g/web-modeler-public-api-rest)
- [Process application / folderId notes](https://docs.camunda.io/docs/apis-tools/migration-manuals/web-modeler-v1-apis/) (use real folder IDs; on newer platforms you may also need `processApplicationId`)

### B.4 Sync algorithm (identical folder)

```text
1. Read gitRoot from repo (e.g. processes/)
2. Auth Modeler API
3. Search all files with folderId = configured root (and descendants if needed)
4. DELETE each file
5. DELETE empty nested folders (keep root folderId)
6. Recreate nested folders from Git relative paths
7. POST each Git file into the matching folderId
8. Fail the job if any step fails mid-way (re-run is safe: wipe again, rewrite again)
```

Mid-failure can leave Modeler empty or partial until the job is re-run — that is expected with wipe+rewrite. Prefer running Modeler sync only after test deploy succeeded, and make the sync job **idempotent** (always wipe then rewrite).

Optional check: after POST, search folder again and assert file **names/paths** match the Git tree (count + relative paths).

---

## Stage C — Promote to syst / prod (approvals)

Do **not** rebuild from a different commit. Promote the **same SHA** that landed on `main`.

Use GitHub **Environments** with required reviewers:

| Job | Environment | Protection |
|-----|-------------|------------|
| `deploy-test` | `test` | none (or optional) — runs on merge |
| `deploy-syst` | `syst` | required reviewers |
| `deploy-prod` | `prod` | required reviewers (+ wait timer if you want) |

Each promote job:

```bash
c8 deploy ./processes --profile=syst --yes   # or prod
```

**Modeler:** sync once after **test** (mirror of what merged). You usually do **not** wipe Modeler again on syst/prod promote — Modeler is design visibility, not per-environment. If you ever need Modeler to show “what’s in prod”, use versions/labels with the Git SHA instead of a second wipe.

---

## Example GitHub Actions shape

```yaml
name: Camunda release

on:
  push:
    branches: [main]
    paths: ['processes/**', 'modeler-sync.json']

jobs:
  deploy-test:
    runs-on: ubuntu-latest
    environment: test
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to test (atomic)
        run: c8 deploy ./processes --profile=test --yes
        # env: cluster credentials for test
      - name: Mirror processes/ → Web Modeler folder (wipe + rewrite)
        run: python scripts/mirror_modeler_folder.py
        # env: MODELER_*; reads modeler-sync.json

  deploy-syst:
    needs: deploy-test
    runs-on: ubuntu-latest
    environment: syst   # required reviewers in GitHub settings
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to syst (atomic)
        run: c8 deploy ./processes --profile=syst --yes

  deploy-prod:
    needs: deploy-syst
    runs-on: ubuntu-latest
    environment: prod   # required reviewers
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to prod (atomic)
        run: c8 deploy ./processes --profile=prod --yes
```

Configure `syst` / `prod` environments under **Settings → Environments** with required reviewers. The workflow waits at those jobs until someone approves.

**Deeper reading:** [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

---

## What “identical” means here

| Git (`processes/`) | Web Modeler (`folderId`) |
|--------------------|---------------------------|
| File present | Same name (and relative subfolder) present after sync |
| File removed in Git | Gone after wipe (not left behind) |
| Content changed | New content from `POST` after delete |
| Extra Modeler-only edits | Discarded on next merge sync |

Cluster **test/syst/prod** each get the same Git tree via separate atomic deploys; only **test** job drives the Modeler wipe+rewrite in this design.

---

## Link index

| Topic | Link |
|-------|------|
| Web Modeler + CI/CD (8.8) | https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/ |
| Web Modeler API overview | https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/ |
| Web Modeler API auth | https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/ |
| OpenAPI (Swagger) | https://modeler.camunda.io/swagger-ui/index.html |
| SDK (deleteFile / createFile / searchFiles) | https://camunda.github.io/camunda-8-js-sdk/classes/index.Modeler.ModelerApiClient.html |
| Process application / folderId | https://docs.camunda.io/docs/apis-tools/migration-manuals/web-modeler-v1-apis/ |
| Create deployment (atomic) | https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/ |
| c8ctl deploy | https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/ |
| GitHub Environments / approvals | https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment |
| Git sync (UI alternative) | https://docs.camunda.io/docs/components/modeler/web-modeler/process-applications/git-sync/ |
