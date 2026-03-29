---
name: backstage-template-writer
description: >-
  Write and maintain Backstage / RHDH software templates (template.yaml).
  Covers scaffolder v1beta3 parameters, steps, built-in and custom actions,
  Nunjucks templating, UI field extensions, output links, and RHDH-specific
  dynamic-plugin actions. Use when creating, editing, reviewing, or debugging
  any Backstage or RHDH scaffolder template, golden path template, or
  template.yaml file.
---

# Backstage / RHDH Template Writer

## API Version

Always use `scaffolder.backstage.io/v1beta3`. Never use v1beta2.

## Template Anatomy

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: kebab-case-name        # unique across catalog
  title: Human Readable Title
  description: What this template creates
  tags: [recommended, python]  # optional; drives filtering in the UI
  annotations:                 # optional
    backstage.io/techdocs-ref: dir:.
spec:
  owner: group:platform-team   # entity ref
  type: service                # service | website | library | documentation | other
  parameters: [...]            # form definition (JSON Schema + ui: extensions)
  steps: [...]                 # backend actions executed in order
  output: { links: [...] }     # shown on completion page
```

## Parameters

Parameters use **JSON Schema draft-07** with `react-jsonschema-form` (rjsf) `ui:` extensions.

### Multi-page forms

Wrap each page in a list item under `parameters`:

```yaml
parameters:
  - title: Project Info
    required: [name]
    properties:
      name:
        title: Name
        type: string
        pattern: '^[a-z][a-z0-9-]*$'
        ui:autofocus: true
  - title: Repository
    required: [repoUrl]
    properties:
      repoUrl:
        title: Repo Location
        type: string
        ui:field: RepoUrlPicker
        ui:options:
          allowedHosts: [github.com]
```

### Common property patterns

| Type | Schema | Notes |
|------|--------|-------|
| Free text | `type: string` | Add `pattern`, `minLength`, `maxLength` as needed |
| Dropdown | `type: string, enum: [a,b]` | Or `enumNames` for labels |
| Boolean | `type: boolean` | Renders as checkbox |
| Number | `type: integer` or `type: number` | Add `minimum` / `maximum` |
| Array | `type: array, items: { type: string }` | Multi-value input |
| Object | `type: object, properties: {...}` | Nested group |

### Conditional fields

Use `if/then/else` or `dependencies`:

```yaml
properties:
  deployTarget:
    type: string
    enum: [kubernetes, vm]
  namespace:
    type: string
dependencies:
  namespace:
    oneOf:
      - properties:
          deployTarget:
            const: kubernetes
          namespace:
            title: K8s Namespace
        required: [namespace]
      - properties:
          deployTarget:
            const: vm
```

### UI field extensions

| Field | Purpose | Key options |
|-------|---------|-------------|
| `RepoUrlPicker` | Git host + owner + repo | `allowedHosts`, `allowedOwners`, `allowedOrganizations` |
| `OwnerPicker` | Select catalog owner entity | `catalogFilter: { kind: [Group, User] }` |
| `EntityPicker` | Any catalog entity | `catalogFilter: { kind: Component, spec.type: service }` |
| `OwnedEntityPicker` | Entities owned by current user | Same filters as EntityPicker |
| `MyGroupsPicker` | Groups the user belongs to | — |
| `Secret` | Masked input | Value available via `${{ secrets.fieldName }}` |

### Secrets

Fields with `ui:field: Secret` are masked in the review step and referenced as `${{ secrets.fieldName }}` (not `parameters`).

## Steps

Steps execute sequentially in the scaffolder backend. Each step:

```yaml
steps:
  - id: uniqueStepId          # referenced by later steps
    name: Human-readable name  # shown in progress UI
    action: action:name        # registered action identifier
    input:                     # action-specific inputs
      key: ${{ parameters.name }}
    if: ${{ parameters.deploy === true }}  # optional conditional
```

### Referencing outputs from previous steps

```yaml
${{ steps['stepId'].output.propertyName }}
${{ steps.stepId.output.propertyName }}    # also valid
```

### Core built-in actions

| Action | Purpose | Key inputs |
|--------|---------|------------|
| `fetch:template` | Render Nunjucks templates from a directory | `url`, `targetPath`, `values`, `copyWithoutTemplating` |
| `fetch:plain` | Copy files without templating | `url`, `targetPath` |
| `fetch:plain:file` | Copy a single file | `url`, `filename`, `targetPath` |
| `publish:github` | Create a GitHub repo and push | `repoUrl`, `description`, `defaultBranch`, `repoVisibility` |
| `publish:github:pull-request` | Open a PR against existing repo | `repoUrl`, `title`, `branchName`, `description` |
| `publish:gitlab` | Create a GitLab project and push | `repoUrl`, `description`, `defaultBranch` |
| `publish:gitlab:merge-request` | Open a merge request | `repoUrl`, `title`, `branchName`, `description` |
| `publish:bitbucketCloud` | Create a Bitbucket Cloud repo | `repoUrl`, `description` |
| `publish:azure` | Create an Azure DevOps repo | `repoUrl`, `description` |
| `catalog:register` | Register entity in catalog | `repoContentsUrl`, `catalogInfoPath` |
| `catalog:write` | Write a `catalog-info.yaml` to workspace | `entity` (object) |
| `catalog:fetch` | Fetch entity data from catalog | `entityRef` |
| `fs:delete` | Delete files from workspace | `files` (array of paths) |
| `fs:rename` | Rename files in workspace | `files` (array of `{from, to}`) |
| `fs:append` | Append content to a file | `file`, `content` |
| `debug:log` | Log message to scaffolder output | `message`, `listWorkspace` |
| `debug:wait` | Pause for N seconds | `seconds` |
| `github:actions:dispatch` | Trigger a GitHub Actions workflow | `repoUrl`, `workflowId`, `branchOrTagName` |

For the complete action reference with all inputs/outputs, see [reference.md](reference.md).

## Nunjucks Templating

`fetch:template` renders files using Nunjucks. Template expressions use `${{ }}` in YAML; inside fetched template files use `${{ }}` or standard `{{ }}` depending on context.

### Common filters

```yaml
${{ parameters.name | lower }}
${{ parameters.name | replace(" ", "-") }}
${{ parameters.entity | parseEntityRef | pick('name') }}
${{ parameters.repoUrl | parseRepoUrl | pick('owner') }}
${{ parameters.name | dump }}                  # JSON-encode
${{ parameters.tags | join(", ") }}
```

### In skeleton files

Inside `./template/` directory files rendered by `fetch:template`, use Nunjucks syntax:

```
# {{ values.name }}

Package owned by {{ values.owner }}.

{% if values.ciProvider == "github-actions" %}
ci: GitHub Actions
{% elif values.ciProvider == "tekton" %}
ci: Tekton
{% endif %}
```

The `values` object comes from `fetch:template`'s `input.values`.

### Excluding files from templating

```yaml
- action: fetch:template
  input:
    url: ./template
    copyWithoutTemplating: ['**/*.png', '**/*.ico', '.helmignore']
    values:
      name: ${{ parameters.name }}
```

## Output

Displayed on the completion page:

```yaml
output:
  links:
    - title: Repository
      url: ${{ steps.publish.output.remoteUrl }}
    - title: Open in catalog
      icon: catalog
      entityRef: ${{ steps.register.output.entityRef }}
    - title: Open CI/CD
      icon: github
      url: ${{ steps.publish.output.remoteUrl }}/actions
  text:
    - title: Component Name
      content: ${{ parameters.name }}
```

## RHDH-Specific Extensions

### Custom actions from RHDH dynamic plugins

RHDH ships additional scaffolder actions via dynamic plugins. Common ones:

| Action | Plugin | Purpose |
|--------|--------|---------|
| `quay:create-repository` | `@janus-idp/backstage-scaffolder-backend-module-quay` | Create Quay.io image repo |
| `argocd:create-resources` | `@roadiehq/scaffolder-backend-argocd` | Create ArgoCD Application |
| `kubernetes:create-namespace` | `@janus-idp/backstage-scaffolder-backend-module-kubernetes` | Create K8s namespace |
| `servicenow:now:table:createRecord` | `@janus-idp/backstage-scaffolder-backend-module-servicenow` | Create ServiceNow record |
| `sonarqube:create-project` | `@janus-idp/backstage-scaffolder-backend-module-sonarqube` | Create SonarQube project |
| `http:backstage:request` | `@roadiehq/scaffolder-backend-module-http-request` | Make HTTP calls to Backstage APIs |
| `github:actions:dispatch` | built-in | Trigger GitHub Actions workflow |

### Registering templates in RHDH

Templates are registered via `app-config.yaml` or the catalog:

```yaml
# app-config.yaml
catalog:
  locations:
    - type: url
      target: https://github.com/org/templates/blob/main/all-templates.yaml
      rules:
        - allow: [Template]
```

Or via a `Location` entity:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: org-templates
  description: Organization golden path templates
spec:
  type: url
  targets:
    - https://github.com/org/templates/blob/main/service-template/template.yaml
    - https://github.com/org/templates/blob/main/library-template/template.yaml
```

### Discovering available actions at runtime

Visit `/create/actions` in your Backstage/RHDH instance, or call the API:

```bash
curl -H "Authorization: Bearer $TOKEN" \
  https://backstage.example.com/api/scaffolder/v2/actions
```

## Template Testing & Validation

### Dry-run via API

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  https://backstage.example.com/api/scaffolder/v2/tasks \
  -d '{
    "templateRef": "template:default/my-template",
    "values": { "name": "test-service", "owner": "group:default/my-team" },
    "secrets": {},
    "dryRun": true
  }'
```

### Local validation checklist

- [ ] YAML is valid (use `yamllint` or IDE linter)
- [ ] `apiVersion` is `scaffolder.backstage.io/v1beta3`
- [ ] `kind: Template`
- [ ] `metadata.name` is kebab-case, unique
- [ ] All `required` fields have corresponding `properties`
- [ ] `repoUrl` fields use `ui:field: RepoUrlPicker`
- [ ] Owner fields use `ui:field: OwnerPicker`
- [ ] All `${{ }}` expressions reference valid parameters, secrets, or step outputs
- [ ] `steps[*].id` values are unique
- [ ] `fetch:template` `values` passes all variables used in skeleton files
- [ ] `catalog:register` uses correct `catalogInfoPath`
- [ ] Skeleton directory contains a valid `catalog-info.yaml`
- [ ] No Nunjucks template files in `copyWithoutTemplating` patterns are actual templates

### Testing skeleton files

Render the skeleton locally with Nunjucks CLI:

```bash
npx nunjucks-cli template/catalog-info.yaml \
  -D '{"values":{"name":"test","owner":"group:default/team"}}' \
  --out /tmp/rendered/
```

## Best Practices

1. **One template per concern**: Don't overload a single template with too many options. Create separate templates for service, library, website, etc.
2. **Sensible defaults**: Pre-fill common values. Use `default:` in properties.
3. **Validate early**: Add `pattern`, `minLength`, `enum` constraints to catch bad input in the form, not in steps.
4. **Descriptive step names**: Users see them in the progress UI.
5. **Idempotent steps**: Prefer actions that won't break if re-run (e.g., `publish:github:pull-request` over manual git commands).
6. **Keep skeletons minimal**: Only include files the project actually needs. Don't copy a bloated boilerplate.
7. **Tag templates**: Use `metadata.tags` for filtering in the "Create" page.
8. **Owner entity refs**: Use `group:default/team-name` format for `spec.owner`.
9. **Skeleton `catalog-info.yaml`**: Always include one in the template skeleton so the created component is auto-registered.
10. **Version template repos**: Pin template URLs to a branch or tag to avoid breaking changes.

## Additional Resources

- Built-in action inputs/outputs and filter details: [reference.md](reference.md)
- Golden path template examples: [examples.md](examples.md)
