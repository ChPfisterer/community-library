# VS Code Workspaces — focus by area

Use the provided workspace files to keep your editor and AI context focused on the area you’re working in, without splitting into multiple repos.

## Available workspaces
- com-lib-api.code-workspace
  - Folders: api/, schemas/openapi/, docs/
  - Excludes web/, ios/, infra/, tools/, and common build outputs (bin/obj, node_modules, dist)
  - Recommended extensions: C#, Podman, YAML, GitHub Actions, markdownlint
- com-lib-web.code-workspace
  - Folders: web/, schemas/openapi/, docs/
  - Excludes api/, ios/, infra/, tools/, build outputs
  - Recommended extensions: Angular Language Service, Prettier, ESLint
- com-lib-ios.code-workspace
  - Folders: ios/, docs/
  - Excludes api/, web/, infra/, tools/, DerivedData/Pods/build
  - Recommended extensions: Swift, markdownlint

## Why this helps
- Smaller, more relevant file graph in the editor speeds up search and indexing.
- Copilot and code navigation prioritize open files and current workspace folders, reducing distraction.
- Keeps monorepo benefits (atomic PRs, shared contracts) while limiting day-to-day noise.

## How to open
- In VS Code: File → Open Workspace from File… and select the desired `.code-workspace` file.
- Or from the command line (optional):
  - `code com-lib-api.code-workspace`

## Tips
- Keep `schemas/openapi/` in both API and Web workspaces to avoid DTO drift.
- When doing end-to-end work, open two windows with different workspaces (API and Web).
- If a folder grows noisy, add patterns to `files.exclude`, `search.exclude`, and `files.watcherExclude` within the workspace JSON.

## Related docs
- Repository organization: `docs/repo-organization.md`
- Roadmap and stories: `docs/roadmap.md`
- API contracts: `schemas/openapi/` (to be generated), `docs/api-endpoints.md`
