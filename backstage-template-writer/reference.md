# Backstage Scaffolder Reference

## Built-in Actions — Full Input / Output

### fetch:template

Renders a directory of Nunjucks templates into the workspace.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | Relative or absolute URL to template directory |
| `targetPath` | string | no | Subdirectory in workspace to write to (default: root) |
| `values` | object | no | Variables passed into Nunjucks templates as `{{ values.key }}` |
| `copyWithoutTemplating` | string[] | no | Glob patterns for files copied without Nunjucks rendering |
| `cookiecutterCompat` | boolean | no | Enable cookiecutter variable syntax compatibility |
| `replace` | boolean | no | If true, replace existing files (default: false) |
| `templateFileExtension` | string | no | Only render files with this extension; strip it after render |
| `token` | string | no | Auth token for private URLs |

**Outputs**: none

---

### fetch:plain

Copies files without any templating.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | URL to file or directory |
| `targetPath` | string | no | Subdirectory in workspace to write to |
| `token` | string | no | Auth token for private URLs |

**Outputs**: none

---

### fetch:plain:file

Copies a single file without templating.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | URL to the file |
| `filename` | string | no | Override the filename |
| `targetPath` | string | no | Subdirectory in workspace |
| `token` | string | no | Auth token |

**Outputs**: none

---

### publish:github

Creates a GitHub repository and pushes workspace contents.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repoUrl` | string | yes | Format: `github.com?repo=name&owner=org` |
| `description` | string | no | Repository description |
| `defaultBranch` | string | no | Default branch name (default: `main`) |
| `repoVisibility` | string | no | `public`, `private`, or `internal` |
| `access` | string | no | `owner/team-slug` for push access |
| `deleteBranchOnMerge` | boolean | no | Auto-delete branches after merge |
| `gitAuthorName` | string | no | Git commit author name |
| `gitAuthorEmail` | string | no | Git commit author email |
| `allowMergeCommit` | boolean | no | Allow merge commits |
| `allowSquashMerge` | boolean | no | Allow squash merging |
| `allowRebaseMerge` | boolean | no | Allow rebase merging |
| `requireCodeOwnerReviews` | boolean | no | Require CODEOWNERS review |
| `requiredStatusCheckContexts` | string[] | no | Required status checks |
| `topics` | string[] | no | Repository topics |
| `collaborators` | array | no | `[{ user, access }]` or `[{ team, access }]` |
| `hasProjects` | boolean | no | Enable GitHub Projects |
| `hasWiki` | boolean | no | Enable wiki |
| `hasIssues` | boolean | no | Enable issues |
| `token` | string | no | GitHub token override |
| `gitCommitMessage` | string | no | Initial commit message |
| `sourcePath` | string | no | Subdirectory of workspace to publish |
| `bypassPullRequestAllowances` | object | no | Users/teams/apps that can bypass PRs |
| `requiredApprovingReviewCount` | number | no | Required approvals |
| `restrictions` | object | no | Push restrictions |
| `requiredConversationResolution` | boolean | no | Require conversation resolution |
| `homepage` | string | no | Repository homepage URL |

| Output | Type | Description |
|--------|------|-------------|
| `remoteUrl` | string | Clone URL (HTTPS) |
| `repoContentsUrl` | string | URL to repo contents (used by `catalog:register`) |
| `commitHash` | string | Initial commit SHA |

---

### publish:github:pull-request

Opens a pull request against an existing GitHub repo.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repoUrl` | string | yes | Repository URL |
| `title` | string | yes | PR title |
| `branchName` | string | yes | Branch name for the PR |
| `description` | string | no | PR body |
| `targetBranchName` | string | no | Base branch (default: repo default) |
| `sourcePath` | string | no | Subdirectory of workspace to include |
| `token` | string | no | GitHub token override |
| `draft` | boolean | no | Create as draft PR |
| `gitAuthorName` | string | no | Commit author name |
| `gitAuthorEmail` | string | no | Commit author email |
| `gitCommitMessage` | string | no | Commit message |
| `reviewers` | string[] | no | GitHub usernames to request review |
| `teamReviewers` | string[] | no | Team slugs to request review |
| `labels` | string[] | no | Labels to add |

| Output | Type | Description |
|--------|------|-------------|
| `remoteUrl` | string | PR HTML URL |
| `pullRequestNumber` | number | PR number |

---

### publish:gitlab

Creates a GitLab project and pushes workspace contents.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repoUrl` | string | yes | Format: `gitlab.com?repo=name&owner=group` |
| `description` | string | no | Project description |
| `defaultBranch` | string | no | Default branch (default: `main`) |
| `repoVisibility` | string | no | `public`, `private`, or `internal` |
| `token` | string | no | GitLab token override |
| `topics` | string[] | no | Project topics |
| `settings` | object | no | GitLab project settings |

| Output | Type | Description |
|--------|------|-------------|
| `remoteUrl` | string | Clone URL |
| `repoContentsUrl` | string | URL to repo contents |
| `projectId` | number | GitLab project ID |

---

### publish:gitlab:merge-request

Opens a merge request against an existing GitLab project.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repoUrl` | string | yes | Project URL |
| `title` | string | yes | MR title |
| `branchName` | string | yes | Source branch |
| `description` | string | no | MR body |
| `targetBranchName` | string | no | Target branch |
| `token` | string | no | GitLab token override |
| `removeSourceBranch` | boolean | no | Delete branch after merge |
| `assignee` | string | no | Assignee username |

| Output | Type | Description |
|--------|------|-------------|
| `mergeRequestUrl` | string | MR URL |
| `projectId` | number | Project ID |
| `mergeRequestId` | number | MR IID |

---

### catalog:register

Registers an entity in the Backstage catalog.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repoContentsUrl` | string | yes* | URL to repo root (from publish output) |
| `catalogInfoPath` | string | no | Path to catalog-info.yaml (default: `/catalog-info.yaml`) |
| `catalogInfoUrl` | string | yes* | Direct URL to catalog-info.yaml (alternative to `repoContentsUrl`) |
| `optional` | boolean | no | If true, don't fail if registration fails |

*One of `repoContentsUrl` or `catalogInfoUrl` is required.

| Output | Type | Description |
|--------|------|-------------|
| `entityRef` | string | Registered entity reference |
| `catalogInfoUrl` | string | URL to the catalog-info.yaml |

---

### catalog:write

Writes a `catalog-info.yaml` into the workspace.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `entity` | object | yes | Entity descriptor object |
| `filePath` | string | no | Path to write (default: `catalog-info.yaml`) |

**Outputs**: none

---

### catalog:fetch

Fetches entity data from the catalog.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `entityRef` | string | yes | Entity reference (e.g., `component:default/my-service`) |
| `optional` | boolean | no | Don't fail if entity not found |

| Output | Type | Description |
|--------|------|-------------|
| `entity` | object | Full entity descriptor |

---

### fs:delete

Deletes files from the workspace.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `files` | string[] | yes | Paths relative to workspace root |

**Outputs**: none

---

### fs:rename

Renames files in the workspace.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `files` | array | yes | `[{ from: "old.txt", to: "new.txt" }]` |

**Outputs**: none

---

### fs:append

Appends content to a file.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | string | yes | Path relative to workspace root |
| `content` | string | yes | Content to append |

**Outputs**: none

---

### debug:log

Logs a message (visible in task logs).

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | no | Message to log |
| `listWorkspace` | boolean | no | List workspace contents in log |

**Outputs**: none

---

### debug:wait

Pauses execution.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `seconds` | number | no | Seconds to wait (default: 5) |

**Outputs**: none

---

### github:actions:dispatch

Triggers a GitHub Actions workflow dispatch event.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repoUrl` | string | yes | Repository URL |
| `workflowId` | string | yes | Workflow filename or ID |
| `branchOrTagName` | string | yes | Ref to run against |
| `workflowInputs` | object | no | Inputs for the workflow |
| `token` | string | no | GitHub token override |

**Outputs**: none

---

## Nunjucks Filters

### Backstage-specific filters

| Filter | Description | Example |
|--------|-------------|---------|
| `parseEntityRef` | Parse `kind:namespace/name` into object | `${{ parameters.owner \| parseEntityRef \| pick('name') }}` |
| `parseRepoUrl` | Parse repo URL into `{ host, owner, repo }` | `${{ parameters.repoUrl \| parseRepoUrl \| pick('repo') }}` |
| `pick(key)` | Extract a property from an object | Chained after `parseEntityRef` or `parseRepoUrl` |
| `projectSlug` | Extract `owner/repo` from repo URL | `${{ parameters.repoUrl \| projectSlug }}` |

### Standard Nunjucks filters

| Filter | Example |
|--------|---------|
| `lower` | `${{ parameters.name \| lower }}` |
| `upper` | `${{ parameters.name \| upper }}` |
| `capitalize` | `${{ parameters.name \| capitalize }}` |
| `replace(old, new)` | `${{ parameters.name \| replace(" ", "-") }}` |
| `trim` | `${{ parameters.name \| trim }}` |
| `truncate(n)` | `${{ parameters.desc \| truncate(100) }}` |
| `dump` | `${{ parameters.config \| dump }}` (JSON-encode) |
| `join(sep)` | `${{ parameters.tags \| join(", ") }}` |
| `first` | `${{ parameters.items \| first }}` |
| `last` | `${{ parameters.items \| last }}` |
| `length` | `${{ parameters.items \| length }}` |
| `default(val)` | `${{ parameters.port \| default(8080) }}` |

## UI Field Extensions — Full Reference

### RepoUrlPicker

```yaml
repoUrl:
  type: string
  ui:field: RepoUrlPicker
  ui:options:
    allowedHosts:
      - github.com
      - gitlab.com
    allowedOwners:
      - my-org
    allowedOrganizations:
      - my-org
    allowedRepos:
      - specific-repo     # lock to specific repo (rare)
    requestUserCredentials:
      secretsKey: USER_OAUTH_TOKEN
      additionalScopes:
        github: ['workflow']
```

### OwnerPicker

```yaml
owner:
  type: string
  ui:field: OwnerPicker
  ui:options:
    catalogFilter:
      kind: [Group, User]
    defaultNamespace: default
```

### EntityPicker

```yaml
component:
  type: string
  ui:field: EntityPicker
  ui:options:
    catalogFilter:
      - kind: Component
        spec.type: service
      - kind: Component
        spec.type: website
    defaultKind: Component
    defaultNamespace: default
```

### OwnedEntityPicker

```yaml
myComponent:
  type: string
  ui:field: OwnedEntityPicker
  ui:options:
    allowedKinds: [Component]
```

### MyGroupsPicker

```yaml
team:
  type: string
  ui:field: MyGroupsPicker
```

### MultihostEntityPicker (RHDH)

```yaml
entity:
  type: string
  ui:field: MultihostEntityPicker
  ui:options:
    hosts:
      - host: backstage.example.com
        label: Production
      - host: backstage-staging.example.com
        label: Staging
```
