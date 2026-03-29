# Golden Path Template Examples

## 1. Microservice (GitHub + ArgoCD)

A production golden path template that scaffolds a new microservice with CI/CD.

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: microservice-golden-path
  title: Create a Microservice
  description: Scaffold a production-ready microservice with CI/CD pipeline
  tags: [recommended, microservice, golden-path]
spec:
  owner: group:default/platform-team
  type: service

  parameters:
    - title: Service Details
      required: [name, owner, language]
      properties:
        name:
          title: Service Name
          type: string
          description: Unique name (kebab-case)
          pattern: '^[a-z][a-z0-9-]{2,62}$'
          ui:autofocus: true
        description:
          title: Description
          type: string
          maxLength: 200
        owner:
          title: Owner
          type: string
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
        language:
          title: Language
          type: string
          enum: [java, python, nodejs, go]
          enumNames: [Java (Spring Boot), Python (FastAPI), Node.js (Express), Go (Gin)]
          default: java

    - title: Infrastructure
      required: [repoUrl]
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]
            allowedOrganizations: [my-org]
        port:
          title: Service Port
          type: integer
          default: 8080
          minimum: 1024
          maximum: 65535
        namespace:
          title: Kubernetes Namespace
          type: string
          default: default
        enableMonitoring:
          title: Enable Prometheus Monitoring
          type: boolean
          default: true

  steps:
    - id: template
      name: Fetch Service Skeleton
      action: fetch:template
      input:
        url: ./skeletons/${{ parameters.language }}
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          port: ${{ parameters.port }}
          namespace: ${{ parameters.namespace }}
          enableMonitoring: ${{ parameters.enableMonitoring }}

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        repoUrl: ${{ parameters.repoUrl }}
        description: ${{ parameters.description }}
        defaultBranch: main
        repoVisibility: internal
        topics:
          - microservice
          - ${{ parameters.language }}
          - golden-path

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

---

## 2. Library / Shared Package

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: shared-library
  title: Create a Shared Library
  description: Scaffold a reusable library with publishing pipeline
  tags: [library, npm, pypi]
spec:
  owner: group:default/platform-team
  type: library

  parameters:
    - title: Library Details
      required: [name, owner, ecosystem]
      properties:
        name:
          title: Package Name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
          ui:autofocus: true
        owner:
          title: Owner
          type: string
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
        ecosystem:
          title: Package Ecosystem
          type: string
          enum: [npm, pypi, maven]
          enumNames: [npm (TypeScript), PyPI (Python), Maven (Java)]
        scope:
          title: npm Scope
          type: string
          description: e.g. @my-org (npm only)
    - title: Repository
      required: [repoUrl]
      properties:
        repoUrl:
          title: Repository
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]

  steps:
    - id: template
      name: Fetch Library Skeleton
      action: fetch:template
      input:
        url: ./skeletons/${{ parameters.ecosystem }}
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          scope: ${{ parameters.scope }}

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        repoUrl: ${{ parameters.repoUrl }}
        description: Shared library — ${{ parameters.name }}
        defaultBranch: main
        repoVisibility: internal

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

---

## 3. Documentation Site (TechDocs)

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: techdocs-site
  title: Create a Documentation Site
  description: Scaffold a TechDocs site with MkDocs
  tags: [documentation, techdocs]
spec:
  owner: group:default/platform-team
  type: documentation

  parameters:
    - title: Documentation Details
      required: [name, owner]
      properties:
        name:
          title: Site Name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
          ui:autofocus: true
        owner:
          title: Owner
          type: string
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
        theme:
          title: MkDocs Theme
          type: string
          enum: [material, readthedocs]
          default: material
    - title: Repository
      required: [repoUrl]
      properties:
        repoUrl:
          title: Repository
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]

  steps:
    - id: template
      name: Fetch Docs Skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          theme: ${{ parameters.theme }}

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        repoUrl: ${{ parameters.repoUrl }}
        description: TechDocs site — ${{ parameters.name }}
        defaultBranch: main
        repoVisibility: internal

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: View Docs
        icon: docs
        entityRef: ${{ steps.register.output.entityRef }}
```

---

## 4. Pull Request Template (Add Config to Existing Repo)

Useful for adding a new config file, Helm chart, or manifest to an existing repository.

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: add-helm-chart
  title: Add Helm Chart to Existing Repo
  description: Opens a PR that adds a Helm chart to an existing service
  tags: [kubernetes, helm, pr]
spec:
  owner: group:default/platform-team
  type: service

  parameters:
    - title: Target Service
      required: [component, repoUrl]
      properties:
        component:
          title: Target Component
          type: string
          ui:field: EntityPicker
          ui:options:
            catalogFilter:
              kind: Component
              spec.type: service
        repoUrl:
          title: Repository
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]
    - title: Chart Settings
      properties:
        chartVersion:
          title: Helm Chart Version
          type: string
          default: '0.1.0'
        replicas:
          title: Default Replicas
          type: integer
          default: 2
          minimum: 1
          maximum: 10

  steps:
    - id: template
      name: Render Helm Chart
      action: fetch:template
      input:
        url: ./helm-skeleton
        targetPath: chart/
        values:
          name: ${{ parameters.component | parseEntityRef | pick('name') }}
          chartVersion: ${{ parameters.chartVersion }}
          replicas: ${{ parameters.replicas }}

    - id: pr
      name: Open Pull Request
      action: publish:github:pull-request
      input:
        repoUrl: ${{ parameters.repoUrl }}
        title: 'feat: add Helm chart'
        branchName: add-helm-chart
        description: |
          Adds a Helm chart for deployment.

          - Chart version: ${{ parameters.chartVersion }}
          - Default replicas: ${{ parameters.replicas }}

  output:
    links:
      - title: Pull Request
        url: ${{ steps.pr.output.remoteUrl }}
```

---

## 5. Skeleton catalog-info.yaml

Every template skeleton should include a `catalog-info.yaml`. Example:

```yaml
# template/catalog-info.yaml (inside skeleton)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name }}
  description: ${{ values.description }}
  annotations:
    github.com/project-slug: ${{ values.repoSlug }}
    backstage.io/techdocs-ref: dir:.
  tags:
    - ${{ values.language }}
spec:
  type: service
  lifecycle: experimental
  owner: ${{ values.owner }}
```

---

## 6. Multi-component Template (Monorepo)

When one template creates multiple catalog entities:

```yaml
steps:
  - id: template
    name: Fetch Monorepo Skeleton
    action: fetch:template
    input:
      url: ./skeleton
      values:
        name: ${{ parameters.name }}
        owner: ${{ parameters.owner }}

  - id: publish
    name: Publish
    action: publish:github
    input:
      repoUrl: ${{ parameters.repoUrl }}
      description: ${{ parameters.name }} monorepo
      defaultBranch: main

  - id: registerFrontend
    name: Register Frontend
    action: catalog:register
    input:
      repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
      catalogInfoPath: /packages/frontend/catalog-info.yaml

  - id: registerBackend
    name: Register Backend
    action: catalog:register
    input:
      repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
      catalogInfoPath: /packages/backend/catalog-info.yaml
```

---

## Common Patterns Cheat Sheet

### Dynamic step ID references
```yaml
${{ steps['publish'].output.remoteUrl }}
${{ steps.publish.output.repoContentsUrl }}
```

### Extract owner name from entity ref
```yaml
${{ parameters.owner | parseEntityRef | pick('name') }}
```

### Extract repo name from RepoUrlPicker
```yaml
${{ parameters.repoUrl | parseRepoUrl | pick('repo') }}
```

### Conditional step execution
```yaml
- id: createNamespace
  name: Create K8s Namespace
  action: kubernetes:create-namespace
  if: ${{ parameters.createNamespace === true }}
  input:
    namespace: ${{ parameters.namespace }}
```

### Delete unwanted skeleton files
```yaml
- id: cleanup
  name: Remove Unused Files
  action: fs:delete
  input:
    files:
      - README.template.md
      - .github/TEMPLATE_README.md
```
