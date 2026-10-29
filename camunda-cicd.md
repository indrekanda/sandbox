https://docs.camunda.io/docs/8.8/apis-tools/working-with-apis-tools/
https://www.postman.com/camundateam/camunda-8-postman/collection/e62j83g/web-modeler-public-api-rest?action=share&creator=11465105&sideView=agentMode

# Camunda 8.8 — Web Modeler & CI/CD (GitHub pipelines)

## Official docs

- [Integrate Web Modeler into CI/CD (8.8)](https://docs.camunda.io/docs/8.8/components/modeler/web-modeler/integrate-web-modeler-in-ci-cd/)
- [Web Modeler API overview](https://docs.camunda.io/docs/apis-tools/web-modeler-api/overview/)
- [Web Modeler API authentication](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/authentication/)
- [Web Modeler API tutorial](https://docs.camunda.io/docs/8.8/apis-tools/web-modeler-api/tutorial/)
- [Web Modeler OpenAPI (SaaS Swagger UI)](https://modeler.camunda.io/swagger-ui/index.html)
- [Deploy resources — Orchestration Cluster REST API](https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/specifications/create-deployment/)
- [c8ctl CLI](https://docs.camunda.io/docs/apis-tools/c8ctl/getting-started/)
- [c8ctl development workflows (deploy)](https://docs.camunda.io/docs/apis-tools/c8ctl/development-workflows/)

## Tools & examples

- [camunda-platform-8-github-action](https://github.com/camunda-community-hub/camunda-platform-8-github-action)
- [Camunda CI/CD pipeline tutorial (GitLab blueprint)](https://github.com/camunda/camunda-platform-tutorials/blob/main/solutions/ci-cd/README.md)

## Quick reference

| Goal | Approach |
|------|----------|
| Git → cluster deploy | `POST /deployments` or `c8 deploy ./processes --yes` |
| Update Modeler file | `PATCH /api/v1/files/:id` (include `revision`) |
| Find changed Modeler files | `POST /api/v1/files/search` (poll; no webhooks) |
| Listen for versions | `POST /api/v1/versions/search` + `GET /api/v1/versions/:id` |

