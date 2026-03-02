# Editor Workspaces — focus by area

Use the workspace files and IDE setup described here to keep your editor and AI context focused on the area you're working in, without splitting into multiple repos.

---

## VS Code workspaces (API and Web)

VS Code `.code-workspace` files are **VS Code-specific** — Xcode cannot open them.

- **com-lib-api.code-workspace**
  - Folders: `api/`, `schemas/openapi/`, `docs/`
  - Excludes: `web/`, `ios/`, `infra/`, `tools/`, build outputs (`bin/`, `obj/`, `node_modules/`, `dist/`)
  - Recommended extensions: C#, Podman, YAML, GitHub Actions, markdownlint
- **com-lib-web.code-workspace**
  - Folders: `web/`, `schemas/openapi/`, `docs/`
  - Excludes: `api/`, `ios/`, `infra/`, `tools/`, build outputs
  - Recommended extensions: Angular Language Service, Prettier, ESLint

### Why this helps
- Smaller, more relevant file graph in the editor speeds up search and indexing.
- **GitHub Copilot** and code navigation prioritize open files and workspace folders, reducing distraction.
- **Claude Code** uses the workspace root and open folders to scope its context window — opening `com-lib-api.code-workspace` focuses Claude on `api/`, `schemas/openapi/`, and `docs/` rather than the entire monorepo.
- Keeps monorepo benefits (atomic PRs, shared contracts) while limiting day-to-day noise.

### How to open
- VS Code: **File → Open Workspace from File…** and select the desired `.code-workspace` file.
- Command line: `code com-lib-api.code-workspace`

### Tips
- Keep `schemas/openapi/` in both API and Web workspaces to avoid DTO drift.
- When doing cross-cutting work, open two VS Code windows side-by-side (API + Web).
- If a folder grows noisy, add patterns to `files.exclude`, `search.exclude`, and `files.watcherExclude` inside the workspace JSON.

---

## Xcode (iOS)

**Xcode is the primary IDE for iOS development.** SwiftUI previews, the iOS Simulator, Instruments, and the Swift debugger all require Xcode. The VS Code `com-lib-ios.code-workspace` exists only for developers who prefer VS Code for lightweight Swift editing.

### Opening the project
Open the Xcode project directly from the `ios/` folder:

```
ios/
└── CommunityLibrary.xcodeproj   (or .xcworkspace if CocoaPods is added later)
```

Xcode only shows the contents of the `ios/` directory. You never see `api/`, `web/`, or `infra/` — scoping is automatic.

### Xcode workspaces vs VS Code workspaces
These are entirely different things:

| | VS Code `.code-workspace` | Xcode `.xcworkspace` |
|---|---|---|
| Purpose | Group folders, configure editor settings | Group multiple Xcode projects (e.g. app + CocoaPods Pods project) |
| Created by | You (committed to the repo) | Xcode / CocoaPods (auto-generated) |
| Used by | VS Code | Xcode |

This project uses **Swift Package Manager (SPM)**, so no `.xcworkspace` is needed initially. If CocoaPods is ever introduced, Xcode will generate one automatically — open that instead of the `.xcodeproj`.

### Claude Code context when working in Xcode
Since Claude Code runs in the terminal (not inside Xcode), launch it from the `ios/` subdirectory to keep its context scoped to iOS:

```bash
cd ios
claude
```

Or open a Claude Code session with `ios/` as the root directory in VS Code using `com-lib-ios.code-workspace` if you want both editors side-by-side.

---

## com-lib-ios.code-workspace (VS Code only)

This workspace exists for developers who prefer VS Code with the Swift extension for editing iOS code. It is **not required** for Xcode users.

- Folders: `ios/`, `docs/`
- Excludes: `api/`, `web/`, `infra/`, `tools/`, `DerivedData/`, `Pods/`, `build/`
- Recommended extensions: Swift, markdownlint

---

## Related docs
- Repository organization: `docs/repo-organization.md`
- Roadmap and stories: `docs/roadmap.md`
- API contracts: `schemas/openapi/` (to be generated), `docs/api-endpoints.md`
