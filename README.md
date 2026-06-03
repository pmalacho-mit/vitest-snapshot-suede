# Vitest Snapshot Suede

This repo is a [suede dependency](https://github.com/pmalacho-mit/suede). 

To see the installable source code, please checkout the [release branch](https://github.com/pmalacho-mit/vitest-snapshot-suede/tree/release).

## Installation

```bash
bash <(curl https://suede.sh/install/release) --repo pmalacho-mit/vitest-snapshot-suede
```

<details>
<summary>
See alternative to using <a href="https://github.com/pmalacho-mit/suede#suedesh">suede.sh</a> script proxy
</summary>

```bash
bash <(curl https://raw.githubusercontent.com/pmalacho-mit/suede/refs/heads/main/scripts/install/release.sh) --repo pmalacho-mit/vitest-snapshot-suede
```

</details>

Just learned about [snapshot testing](https://vitest.dev/guide/snapshot) from [this video](https://www.youtube.com/watch?v=rUYP4C29yCw) (presenter calls them "expect tests").

I'm not satisfied with that API, as I'd like for the files written by expect tests to be able to be encoded more _richly_ (e.g. snapshot is written out as markdown). 

I'm imagining an API like:

```ts
// some.test.ts
import { test } from 'vitest';
import { snapshot } from "./vitest-snapshot-suede";

test('toUpperCase', snapshot("md", ({ header, paragraph }) => {
  const result = toUpperCase('foobar');
  header(1)`Result`;
  paragraph`${result}`;
}))
```

And the first time that runs, there's no snapshot, but it will write out a snapshot to: `some.test.snapshots`.

So we have:

- some.test.ts
- some.test.snapshots/
  - toUpperCase.md

And the test file will get updated to:

```ts
// some.test.ts
import { test } from 'vitest';
import { snapshot } from "./vitest-snapshot-suede";
import toUpperCase from "./some.test.snapshots/toUpperCase.md?raw";

test('toUpperCase', snapshot("md", ({ ... }) => {
  ... 
}, toUpperCase))
```

So I need to look into how vitest snapshot inline updates the code in the file at a specific location of a test.

I then also want it to be possible to use "./some.test.snapshots/toUpperCase.md?raw" to navigate directly to that file, claude says its possible like:

The solution is a tiny TypeScript language-service plugin that detects a definition request on an import string ending in a Vite query, strips the query, and returns the real file:

```ts
// tools/vite-raw-goto/index.ts  (compile to .js, or just write it as .js)
import path from "node:path";
import type * as ts from "typescript/lib/tsserverlibrary";

const VITE_QUERY = /\?(raw|url|inline|worker|sharedworker)$/;

function init({ typescript: ts }: { typescript: typeof import("typescript/lib/tsserverlibrary") }) {
  function create(info: ts.server.PluginCreateInfo): ts.LanguageService {
    const ls = info.languageService;
    const proxy = Object.create(null) as ts.LanguageService;
    for (const k of Object.keys(ls) as Array<keyof ts.LanguageService>) {
      const fn = ls[k] as any;
      proxy[k] = (...args: any[]) => fn.apply(ls, args);
    }

    function findString(node: ts.Node, sf: ts.SourceFile, pos: number): ts.StringLiteralLike | undefined {
      if (pos < node.getFullStart() || pos > node.getEnd()) return undefined;
      if (ts.isStringLiteralLike(node) && pos >= node.getStart(sf)) return node;
      let found: ts.StringLiteralLike | undefined;
      node.forEachChild((c) => (found ??= findString(c, sf, pos)));
      return found;
    }

    proxy.getDefinitionAndBoundSpan = (fileName, position) => {
      const original = ls.getDefinitionAndBoundSpan(fileName, position);
      const sf = ls.getProgram()?.getSourceFile(fileName);
      if (!sf) return original;

      const lit = findString(sf, sf, position);
      if (!lit || !VITE_QUERY.test(lit.text) || !lit.text.startsWith(".")) return original;

      const bare = lit.text.replace(VITE_QUERY, "");
      const target = path.resolve(path.dirname(fileName), bare);
      if (!ts.sys.fileExists(target)) return original;

      return {
        textSpan: { start: lit.getStart(sf) + 1, length: lit.getWidth(sf) - 2 },
        definitions: [{
          fileName: target,
          textSpan: { start: 0, length: 0 },
          kind: ts.ScriptElementKind.moduleElement,
          name: bare,
          containerName: "",
          containerKind: ts.ScriptElementKind.unknown,
        }],
      };
    };
    return proxy;
  }
  return { create };
}
export = init;
```

Wiring it up:

```jsonc
// tsconfig.json
{ "compilerOptions": { "plugins": [{ "name": "vite-raw-goto" }] } }
```

```jsonc
// .vscode/settings.json — so VS Code finds the local plugin
{ "typescript.tsserver.pluginPaths": ["./tools"] }
```

The plugin needs to resolve as a module named `vite-raw-goto` under `./tools` (give it a `package.json` with a `main`), and the `index.ts` must be compiled to JS since tsserver loads JS. One gotcha: if it doesn't activate, switch VS Code to the workspace TypeScript version (Command Palette → "TypeScript: Select TypeScript Version" → Use Workspace Version). It only affects the editor, which is exactly what you want.

Lighter-weight options, with their limits: `typescript-cleanup-defs` can suppress the `client.d.ts` result so you stop landing there, but since there's no real TS target it leaves go-to-definition as a no-op rather than opening the file. And antfu's `vscode-goto-alias` does redirect-on-hit-of-generated-`.d.ts`, but it's built for the `typeof import()` auto-import pattern, so it won't cover `?raw` out of the box. The plugin above is the only approach I know of that actually drops you on the source file.
