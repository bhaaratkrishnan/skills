---
name: react-email-package2
description: >
  Scaffold a self-contained React Email package in a monorepo by converting the create-email
  starter into a buildable, importable workspace package. Use when creating a new email or
  transactional package, adding a React Email template, or wiring an email package into an app.
---

# React Email package

Turn the `create-email` starter into a **self-contained** workspace package: it owns its build config, its `exports` seam, and its dependencies, so a consumer adds one workspace dependency and imports it like any npm package. Works in any monorepo and with any package manager.

## Step 0 — Detect the package manager and layout

Read the workspace root for a lockfile: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` or `npm-shrinkwrap.json` → npm, `bun.lock`/`bun.lockb` → bun. If none is found, or several appear, **ask the user** which manager the repo uses.

Then find where packages live: check the workspace globs (`pnpm-workspace.yaml` `packages`, `package.json` `workspaces`, or an Nx/Turbo layout). Pick the directory that already holds the repo's packages (commonly `packages/`) and note the existing naming pattern for the new package (commonly `@<scope>/<name>`).

Completion: manager chosen and its three commands (scaffold / install / add-workspace-dep) fixed from the reference table; target path and package name decided.

## Step 1 — Scaffold the starter

Pass the target directory so the generator runs non-interactively (it prompts for a name otherwise; it does not install):

```
<pm create command> email@latest <path-to-package-dir>     # e.g. packages/email
```

Completion: `emails/`, `package.json`, and `tsconfig.json` exist under the package dir.

## Step 2 — Convert the starter into a package

Edit the generated `package.json` to this shape (keep the dependency versions create-email wrote — don't invent new ones):

```json
{
  "name": "@<scope>/<name>",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./dist/index.mjs",
    "./render": "./dist/render.mjs",
    "./package.json": "./package.json"
  },
  "scripts": {
    "build": "tsdown",
    "dev": "email dev",
    "export": "email export",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "react-email": "<keep>",
    "react-dom": "<keep>",
    "react": "<keep>"
  },
  "devDependencies": {
    "@react-email/ui": "<keep>",
    "@types/react": "<keep>",
    "@types/react-dom": "<keep>",
    "tsdown": "<keep repo tooling version, or latest>",
    "typescript": "<keep repo tooling version, or latest>"
  }
}
```

Leave the generated `tsconfig.json` as-is — it already sets `jsx: react-jsx` and `moduleResolution: bundler`, which is what the `.tsx` templates need.

Add `tsdown.config.ts`:

```ts
import { defineConfig } from "tsdown";

export default defineConfig({
  dts: { tsgo: true },
  exports: true,
  entry: ["emails/index.ts", "emails/render.ts"],
  outDir: "dist",
  format: "esm",
  clean: true,
  deps: {
    neverBundle: ["react", "react-dom", "react-email"],
  },
});
```

Add the render helper `emails/render.ts` (copy verbatim — it is generic):

```ts
import { render } from "react-email";
import React from "react";

export async function renderTemplate<P>(
  Component: React.ComponentType<P>,
  props: P,
): Promise<string> {
  return render(
    React.createElement(
      Component as React.ComponentType<Record<string, unknown>>,
      props as Record<string, unknown>,
    ),
  );
}
```

Add a barrel `emails/index.ts` and re-export every template's value and prop type from it. Delete the starter's demo templates (`vercel-invite-user.tsx`, `stripe-welcome.tsx`, `plaid-verify-identity.tsx`, `notion-magic-link.tsx`) and its `emails/static/` images.

Completion: `package.json` has `type: module`, an `exports` map exposing `.` and `./render`, and a `tsdown` build script; `tsdown.config.ts`, `emails/render.ts`, and `emails/index.ts` exist; demo files gone.

## Step 3 — Register and install

Confirm the package dir falls under the workspace's package glob (add a glob or an explicit entry if it doesn't). Then install from the workspace root with the detected manager.

Completion: install succeeds and the package is linked into the workspace (it appears in the manager's workspace listing / `node_modules`).

## Step 4 — Add a template

Each template is `emails/<template>.tsx`. Reproduce this shape:

```tsx
import { Body, Html, Text } from "react-email";

export interface <Name>Props {
  foo: string;
}

const <Name> = ({ foo }: <Name>Props) => (
  <Html>
    <Body>
      <Text>{foo}</Text>
    </Body>
  </Html>
);

<Name>.PreviewProps = { foo: "example" };

export default <Name>;
```

Re-export it from `emails/index.ts`:

```ts
export { default as <Name>, type <Name>Props } from "./<template>";
```

Completion: the component default-exports, its props are a named exported interface, `.PreviewProps` is set, and `emails/index.ts` re-exports both the value and the type.

## Step 5 — Build, then consume

```
<pm filter> <your package> build        # must run before any consumer typechecks
<pm filter> <your package> typecheck
```

In the consuming app, add the workspace dependency, then import from both subpaths:

```ts
import { <Name> } from "@<scope>/<name>";
import { renderTemplate } from "@<scope>/<name>/render";
```

Completion: `dist/index.mjs`, `dist/render.mjs`, and the matching `.d.mts` exist; the package typechecks; the consumer typechecks with both imports resolving.

## Reference

**Package-manager commands.**

| Manager | Scaffold | Install (root) | Add workspace dep to an app |
|---|---|---|---|
| pnpm | `pnpm create email@latest <dir>` | `pnpm install` | `pnpm --filter <app> add @<scope>/<name>@workspace:*` |
| yarn | `yarn create email <dir>` | `yarn install` | `yarn workspace <app> add @<scope>/<name>@workspace:*` |
| npm | `npm create email@latest <dir>` | `npm install` | set `"@<scope>/<name>": "*"` in the app, then `npm install` |
| bun | `bun create email <dir>` | `bun install` | `bun add @<scope>/<name>@workspace:* --filter <app>` |

yarn classic and npm do not understand the `workspace:` protocol — use `*` (npm) or a `file:`/`link:` range there.

**Why it stays self-contained.** `react`, `react-dom`, and `react-email` sit in the package's own `dependencies` and are kept out of the bundle with `neverBundle`, so the emitted `dist/*.mjs` `import`s them and Node resolves them from the package's own `node_modules` — a package always resolves its declared dependencies. The consumer therefore needs to add nothing but the workspace dependency. (For one-shot email rendering, duplicate React is harmless, so regular deps are simpler than `peerDependencies`.)

**Gotchas.**
- A task orchestrator (Turbo/Nx/Lerna) may need a build dependency edge so the package builds before a consumer typechecks it; on a clean tree a bare consumer typecheck fails with "cannot find module @<scope>/<name>". Build the package first.
- `email dev` (the preview server) needs the `email` binary, which ships with the `react-email` dependency.
- `exports: true` in tsdown keeps the `exports` map in sync with the entries; the map must always expose `.` and `./render` for the two import paths above.
- Anything listed in `neverBundle` that is *not* in `dependencies` becomes the consumer's problem to supply.
