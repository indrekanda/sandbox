https://docs.camunda.io/docs/8.8/apis-tools/working-with-apis-tools/
https://www.postman.com/camundateam/camunda-8-postman/collection/e62j83g/web-modeler-public-api-rest?action=share&creator=11465105&sideView=agentMode

# Camunda 8.8 — Automating BPMN updates & deployments (GitHub pipelines)

Guide for updating BPMN/DMN/forms (and related resources) automatically and deploying new versions to Camunda 8.8 from CI/CD — especially GitHub Actions.

**Start here (full official guide):**  
[Integrate Web Modeler into CI/CD (8.8)](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)

---

## 1. Choose an approach

There are two main patterns. Pick based on where diagrams should live as the source of truth.

### Pattern A — Git is source of truth (recommended for GitHub)

Store `.bpmn` / `.dmn` / `.form` files in the repository. On push or merge, the pipeline deploys them to the Camunda cluster. Web Modeler is optional (edit locally with Desktop Modeler, or sync later).

**Why use it:** Simple, reviewable diffs in PRs, natural fit for GitHub Actions, no polling of Modeler.

**Deeper reading:**
- [Deploy resources — Orchestration Cluster REST API](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/)
- [c8ctl development workflows (deploy)](https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/)
- [Community GitHub Action](https://github.com/camunda-community-hub/camunda-platform-8-github-action)

### Pattern B — Web Modeler ↔ Git ↔ deploy

Use Web Modeler as the modeling UI. Sync files/versions to Git via the **Web Modeler API**, then let the pipeline lint, review, and deploy. Camunda’s Marketplace blueprint follows this idea (GitLab-oriented; same APIs work for GitHub).

**Why use it:** Business users stay in Web Modeler; engineering owns release gates in Git/CI.

**Deeper reading:**
- [Integrate Web Modeler into CI/CD (8.8)](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)
- [CI/CD pipeline tutorial / blueprint](https://github.com/camunda/camunda-platform-tutorials/blob/main/solutions/ci-cd/README.md)
- [Web Modeler API overview](https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/)

> **Note:** There are no change webhooks for Web Modeler today. You **poll** search endpoints for updated files or new versions.

---

## 2. Credentials you need

| Client | Purpose |
|--------|---------|
| **Web Modeler API token** | Create/read/update projects, files, versions in Modeler |
| **Cluster / Zeebe client** | Deploy resources to a Camunda 8 cluster (dev/stage/prod) |

**Deeper reading:**
- [Web Modeler API authentication](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/)
- [Orchestration Cluster API authentication](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/orchestration-cluster-api-rest-authentication/)
- [c8ctl getting started (profiles & auth)](https://docs.camunda.io/docs/apis-tools/c8ctl/getting-started/)

Optional: disable one-click deploy from Web Modeler so only the pipeline can deploy (Self-Managed env vars `ZEEBE_BPMN_DEPLOYMENT_ENABLED` / `ZEEBE_DMN_DEPLOYMENT_ENABLED`). Details in the [CI/CD integration guide](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/).

---

## 3. Web Modeler API — update & sync diagrams

Base path: `/api/v1/...` (SaaS: `https://modeler.cloud.camunda.io`). Interactive OpenAPI: [Swagger UI](https://modeler.camunda.io/swagger-ui/index.html).

| Goal | Endpoint | Notes |
|------|----------|--------|
| Search files (e.g. changed since last sync) | `POST /api/v1/files/search` | Filter by `projectId`, `updated`; pagination required |
| Get file content | `GET /api/v1/files/:id` | Latest edit state |
| Create file | `POST /api/v1/files` | Name, project, content, file type |
| Update file | `PATCH /api/v1/files/:id` | Must send current `revision` to avoid conflicts |
| Search versions | `POST /api/v1/versions/search` | Use to trigger CI when a “ready” version is created |
| Get a version’s content | `GET /api/v1/versions/:id` | Snapshot for deploy or Git sync |
| Visual diff for review | `GET /api/v1/versions/compare/{v1}...{v2}` | Link for PR review |

**Hands-on walkthrough:** [Web Modeler API tutorial](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/tutorial/)

**Example — find files updated after last sync:**

```json
POST /api/v1/files/search
{
  "filter": {
    "projectId": "(PROJECT TO SYNC)",
    "updated": ">(LAST SYNC DATE)"
  },
  "page": 0,
  "size": 50
}
```

**Example — listen for new versions:**

```json
POST /api/v1/versions/search
{
  "filter": {
    "created": ">(YOUR LAST SYNC DATE)"
  },
  "page": 0,
  "size": 50
}
```

Then push content into Git (commit/PR) and continue the pipeline.

---

## 4. Deploy to Camunda 8.8

Deployment goes to the **orchestration cluster**, not into Web Modeler. Updating Modeler and deploying to the engine are separate steps.

### Option A — REST API

`POST /deployments` — atomic deploy of BPMN, DMN, forms, and other resources.

**Reference:** [Create deployment](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/)

### Option B — c8ctl (good for scripts / Actions)

```bash
c8 deploy ./processes --profile=prod --yes
```

Supports single files, multiple files, or a directory (`.bpmn`, `.dmn`, `.form` by default).

**Reference:** [c8ctl development workflows](https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/)

### Option C — Community GitHub Action

```yaml
- uses: camunda-community-hub/camunda-platform-8-github-action@master
  with:
    clientConfig: ${{ secrets.ZEEBE_CLIENT_CONFIG }}
    operation: deployProcess
    bpmnDirectory: bpmn
```

**Reference:** [camunda-platform-8-github-action](https://github.com/camunda-community-hub/camunda-platform-8-github-action)

> Prefer the official REST API or `c8ctl` for long-term 8.8 pipelines; the Action is convenient but community-maintained. `zbctl` is deprecated for new work.

---

## 5. Suggested GitHub Actions pipeline

1. **Trigger** — push/PR on `bpmn/**` (or scheduled poll of Modeler versions if using Pattern B).
2. **Checkout** — get diagram files from Git.
3. **Lint** — `bpmnlint` / `dmnlint` (same family of rules Web Modeler uses).
4. **Test** — e.g. [camunda-process-test](https://docs.camunda.io/docs/) / client-based tests against a preview cluster if needed.
5. **Review** — optional: Modeler visual diff links in the PR ([compare versions](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)).
6. **Deploy** — `c8 deploy ... --yes` or `POST /deployments` to the target environment (dev → stage → prod).

Camunda’s end-to-end write-up of build/test/review/publish stages:  
[Integrate Web Modeler into CI/CD — pipeline stages](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)

Blueprint (GitLab; adapt steps for GitHub):  
[camunda-platform-tutorials / solutions/ci-cd](https://github.com/camunda/camunda-platform-tutorials/blob/main/solutions/ci-cd/README.md)

---

## 6. Link index

| Topic | Link |
|-------|------|
| Web Modeler + CI/CD (8.8) | https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/ |
| Web Modeler API overview | https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/ |
| Web Modeler API auth | https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/ |
| Web Modeler API tutorial | https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/tutorial/ |
| Web Modeler OpenAPI (Swagger) | https://modeler.camunda.io/swagger-ui/index.html |
| Deploy resources (REST) | https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/ |
| Cluster API auth | https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/orchestration-cluster-api-rest-authentication/ |
| c8ctl getting started | https://docs.camunda.io/docs/apis-tools/c8ctl/getting-started/ |
| c8ctl deploy workflows | https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/ |
| GitHub Action (community) | https://github.com/camunda-community-hub/camunda-platform-8-github-action |
| CI/CD blueprint (GitLab) | https://github.com/camunda/camunda-platform-tutorials/blob/main/solutions/ci-cd/README.md |


