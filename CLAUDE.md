# CLAUDE.md

## Project Structure

Excalidraw is a **monorepo** with a clear separation between the core library and the application:

- **`packages/excalidraw/`** - Main React component library published to npm as `@excalidraw/excalidraw`
- **`excalidraw-app/`** - Full-featured web application (excalidraw.com) that uses the library
- **`packages/`** - Core packages: `@excalidraw/common`, `@excalidraw/element`, `@excalidraw/math`, `@excalidraw/utils`
- **`examples/`** - Integration examples (NextJS, browser script)

## Development Workflow

1. **Package Development**: Editor features belong in `packages/*`, but **ShenGo customizations must be captured as `patches/*.patch`**, not committed as source edits (see below)
2. **App Development**: Work in `excalidraw-app/` for app-specific features (same rule: fork-only behavior goes in patches)
3. **Testing**: Always run `yarn test:update` before committing
4. **Type Safety**: Use `yarn test:typecheck` to verify TypeScript
5. **Patches**: `yarn apply:patches` (also runs first in `yarn build:packages`)

## Development Commands

```bash
yarn test:typecheck  # TypeScript type checking
yarn test:update     # Run all tests (with snapshot updates)
yarn fix             # Auto-fix formatting and linting issues
```

## Architecture Notes

### Package System

- Uses Yarn workspaces for monorepo management
- Internal packages use path aliases (see `vitest.config.mts`)
- Build system uses esbuild for packages, Vite for the app
- TypeScript throughout with strict configuration

## Fork customizations: patches only

This tree tracks upstream Excalidraw. **ShenGo-specific changes must live in `patches/*.patch`**, not as committed edits in `packages/` or `excalidraw-app/`.

The goal: when we pull upstream, merge conflicts should only involve the patch files (and only if a patch no longer applies). Upstream source stays mergeable.

### How patches are applied

- Directory: `patches/` (sorted alphabetically by filename).
- Script: `scripts/applyPatch.js` (`yarn apply:patches`).
- `yarn build:packages` runs `apply:patches` first.
- Apply is idempotent: already-applied patches are skipped (`git apply --reverse --check`).
- Unstaged non-patch working-tree changes block apply unless every patch is already applied.

Do **not** commit the result of `git apply`. Applied hunks are local/build-time only. The committed source of truth is the `.patch` file.

### Making or changing a customization

1. Start from a clean tree (or with patches already applied).
2. Edit the relevant files in `packages/` (or wherever the patch targets).
3. Capture **only** that feature as a unified diff:
   ```bash
   git diff -- packages/path/to/changed/files > patches/<name>.patch
   ```
   Or, if updating an existing patch, regenerate the whole file from the full set of files that patch owns.
4. Revert the working-tree edits in those files (`git checkout -- <files>` or `git restore`).
5. Confirm `yarn apply:patches` applies cleanly from a clean tree.
6. Commit the `.patch` file (and `scripts/applyPatch.js` if you changed the applicator). Do not commit the applied source.

One feature per patch. Do not fold unrelated ShenGo behavior into an existing patch.

If upstream moved code, **update the patch**, do not “just edit the package so it works.” After an upstream pull:

```bash
yarn apply:patches
```

If a patch fails `--check`, refresh that patch against the new sources and leave other patches untouched.

### Current patches

Applied in filename order:

| Patch | Purpose |
| --- | --- |
| `arrow-text-reposition.patch` | Drag arrow labels along the path; `labelPosition` on arrows; bound-text hit testing with threshold. |
| `bound-text-stroke-color.patch` | Bound text does not inherit a transparent container stroke; stroke-color actions treat bound text separately. |
| `expose-rotate.patch` | Imperative `rotateSelectedElements`, hide default rotate handle, per-element selection color. |
| `fonts-local-assets.patch` | Load fonts from same-origin `/fonts` instead of the esm.sh CDN fallback when possible. |
| `frame-auto-resize.patch` | Frame auto-resize, in-frame title insets, descendant collection, related UI/export/restore. |
| `preserve-arrow-bindings.patch` | Do not unbind arrow endpoints when a drag finds no hit (keeps existing bindings while editing). |
| `shape-shadow.patch` | Optional `shadow` on elements; render/restore/API support. |
| `ui-shape-footer.patch` | Shape chrome/footer slots, reserved footer height, related App/index/types/CSS. |
| `visibility-hidden.patch` | `isElementHidden` / hidden elements skip hit-testing, search, render, and export as needed. |
| `z-min-element-size.patch` | Optional `minWidth` / `minHeight` enforced during interactive resize. |

Patches that touch the same files (e.g. `types.ts`, `newElement.ts`, `App.tsx`, `restore.ts`) are ordered so later patches apply on top of earlier ones. If you add a patch that overlaps, pick a filename that sorts into the right sequence, or adjust existing patches so they still apply in alphabetical order.

### What not to do

- Do not land ShenGo behavior as a normal commit in upstream files.
- Do not “fix” a failed patch by committing applied source.
- Do not hand-edit a `.patch` unless you also re-apply it on a clean tree.
- Do not add new npm/git patch systems (`patch-package`, etc.) — use `patches/` + `applyPatch.js`.
