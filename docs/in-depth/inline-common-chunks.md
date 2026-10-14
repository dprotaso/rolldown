# Inline Common Chunks

`experimentalInlineCommonChunks` copies the code of small common chunks into the chunks that import them, then removes the separate common chunk files. The copies use a shared runtime, so each module still executes only once.

## Why small common chunks are created

Rolldown's [automatic code splitting](./automatic-code-splitting.md) follows two rules inherited from Rollup's output model:

1. A module's code appears in only one output chunk.
2. An entry only loads application modules that are reachable from that entry.

A module shared by several entries therefore goes into a common chunk. Modules can share that chunk only if the same set of entries needs them. With three entries, there can already be four such sets: `AB`, `AC`, `BC` and `ABC`. Combining two sets would make at least one entry load a module it does not use. Dynamic import targets count as entries too, so these combinations also arise between routes in a single-page application.

The size of the modules does not change these rules. For example, suppose A, B and C all use a large library, but only A and B use a small helper. The library can go into a common chunk shared by all three entries. The helper needs a separate common chunk, however small it is.

We could use [manual code splitting](./manual-code-splitting.md) to put the helper in the library chunk. That removes a file by giving up the second rule: C now downloads the helper even though it does not use it.

Inlining gives up the first rule instead. Put a copy of the helper in A and another in B, and neither entry needs a separate file for it. C still loads only the library.

```dot
digraph {
    bgcolor="transparent";
    rankdir=TB;
    node [shape=box, style="filled,rounded", fontname="Arial", fontsize=12, margin="0.2,0.1", color="${#3c3c43|#dfdfd6}", fontcolor="${#3c3c43|#dfdfd6}"];
    edge [color="${#3c3c43|#dfdfd6}"];

    subgraph cluster_before {
        label="Separate common chunks";
        fontname="Arial";
        fontcolor="${#3c3c43|#dfdfd6}";
        color="${#3c3c43|#dfdfd6}";
        a [label="A", fillcolor="${#fff0e0|#4a2a0a}"];
        b [label="B", fillcolor="${#fff0e0|#4a2a0a}"];
        c [label="C", fillcolor="${#fff0e0|#4a2a0a}"];
        helper [label="helper", fillcolor="${#dcfce7|#14532d}"];
        library [label="library", fillcolor="${#dbeafe|#1e3a5f}"];
        a -> helper;
        b -> helper;
        { a b c } -> library;
    }

    subgraph cluster_after {
        label="Helper inlined";
        fontname="Arial";
        fontcolor="${#3c3c43|#dfdfd6}";
        color="${#3c3c43|#dfdfd6}";
        a_inline [label="A + helper", fillcolor="${#fff0e0|#4a2a0a}"];
        b_inline [label="B + helper", fillcolor="${#fff0e0|#4a2a0a}"];
        c_inline [label="C", fillcolor="${#fff0e0|#4a2a0a}"];
        library_inline [label="library", fillcolor="${#dbeafe|#1e3a5f}"];
        { a_inline b_inline c_inline } -> library_inline;
    }
}
```

The diagram omits the shared runtime chunk, which is needed to keep the copies behaving as one module.

## Sharing state between copies

Copying the code alone would change its behavior. Consider a helper with a counter:

```js
// helper.js
console.log('helper initialized');
let count = 0;
export function inc() {
  return ++count;
}

// a.js and b.js
import { inc } from './helper.js';
console.log(inc());
```

If A loads before B in the same JavaScript environment, the helper logs `helper initialized` once, A logs `1`, and B logs `2`. Both calls to `inc` change the same `count`.

If we put the helper's body directly in both files, each file gets its own `count`. Both calls log `1`, and `helper initialized` is logged twice. Exported objects and functions would also be created twice, so consumers would no longer receive the same values.

The code needs to be defined in more than one place, but executed only once. Rolldown does this by wrapping the common chunk's code in a function, called a _factory_, and registering that function with a shared runtime. Registration keeps the first factory for a given identifier and ignores later copies.

Here is the output in both A and B, simplified and with the library import omitted:

```js
import { __share, __share_require, __share_export, __esmMin } from './rolldown-runtime.js';

__share('helper', (exports) => {
  let count;
  function inc() {
    return ++count;
  }
  const init_helper = __esmMin(() => {
    console.log('helper initialized');
    count = 0;
  });
  __share_export(exports, {
    inc: () => inc,
    init_helper: () => init_helper,
  });
});

const helper = __share_require('helper');
helper.init_helper();
console.log((0, helper.inc)());
```

`__share` registers the factory without calling it. The first `__share_require('helper')` runs the factory and saves its exports object. Every later call returns that same object.

When A loads, it registers the factory, gets the shared exports, and calls `init_helper`. That initializes `count` to zero and logs `helper initialized`. When B loads, its factory is ignored. It receives the same `inc` and `init_helper` functions that A received. `__esmMin` makes the initializer run only once, so B's call to `init_helper` does nothing. Its call to `inc` increments the shared count to `2`.

Registration, factory execution and module initialization are separate steps. This lets a chunk register shared code before another chunk needs it, while `strictExecutionOrder` controls when the module's top-level code runs. `__share_export` exposes exported variables through getters to preserve live bindings.

The factory wraps a whole common chunk, which may contain several modules. Those modules still share a scope and refer to each other directly inside it. Only references crossing into the inlined chunk go through its exports object. The large library in our example remains an ordinary ESM import.

## Avoiding unnecessary copies

A and B each need a factory if either page can load first. A route loaded only through A may not need one: A has already registered the factory by the time it starts the dynamic import. The route can get the shared exports from the runtime.

Rolldown checks every known path from the configured and plugin-emitted entries before leaving out a copy. If the same route can also be loaded through another entry that does not supply the factory, Rolldown has to keep a copy available on that path too. Static dependencies can supply factories in the same way, provided they register them before the consumer runs.

This is also why loading an arbitrary output chunk by its URL can fail. A non-entry chunk may depend on registrations from its parents. Imports hidden in generated strings or made by external modules are not part of the graph Rolldown checks. Modules that need to load independently should be entries; a plugin can emit one with `preserveSignature: 'strict'` when its exports must be kept. Another option is to exclude its shared dependencies, including transitive ones, from inlining.

The registry belongs to the shared runtime chunk. Copies share state when they import that same runtime, so loading files from different builds through the same runtime URL is unsupported.

## Enabling inlining

```js [rolldown.config.mjs]
import { defineConfig } from 'rolldown';

export default defineConfig({
  input: 'src/main.js',
  preserveEntrySignatures: false,
  output: {
    dir: 'dist',
    format: 'es',
    codeSplitting: {
      experimentalInlineCommonChunks: {
        maxSize: 10 * 1024,
      },
    },
  },
});
```

This selects eligible common chunks smaller than 10 KiB. The size is the sum of their modules' transformed source sizes, before rendering, minification and compression. `maxSize` defaults to `0`, which disables inlining; `Infinity` removes the size check. The [option reference](/reference/OutputOptions.codeSplitting#experimentalinlinecommonchunks) describes the configuration requirements.

Inlining applies to automatic common chunks. Entries, dynamic import targets and chunks created by manual groups remain files. You can use manual groups to keep a library together and let inlining handle small common chunks among the remaining modules. Matching any module with [`exclude`](/reference/TypeAlias.ExperimentalInlineCommonChunksOptions#exclude) keeps its entire common chunk as a file.

The 10 KiB value is an example threshold. Loading A alone transfers one copy of the helper; loading both A and B can transfer two. The shared runtime prevents duplicate execution, but the browser still downloads each file's bytes. A change to the helper also changes each file containing it.

The runtime adds a shared file of its own, plus registration code, initialization wrappers and export lookups. Those lookups also limit what a minifier can do across the factory boundary. Inlining can reduce requests at the cost of more transferred code and runtime work; `maxSize` does not tell you whether that tradeoff improves loading performance for your application.

::: details Current restrictions and plugin compatibility

The feature is experimental and currently requires ESM output and an explicit `preserveEntrySignatures: false`. That setting allows unused entry exports to be removed, which matters for frameworks that consume them. Inlining enables `strictExecutionOrder` when it is omitted; explicitly setting it to `false` is an error. `preserveModules`, `experimental.devMode` and `experimental.onDemandWrapping` must be off. Test with `strictExecutionOrder: true` first, particularly if the application has [top-level await in static import cycles](https://github.com/rolldown/rolldown/issues/9548).

Some common chunks stay as files even below the threshold. These include chunks containing top-level await or a static dependency on it, retained `import.meta`, unresolved dynamic imports, or imports of external modules. A chunk also stays a file if another output chunk re-exports one of its bindings. Direct `eval` and static cycles involving a chunk that stays a file can prevent inlining too. Declaration modules and the builtin `asset` and `copy` modules are excluded. Resolved internal dynamic imports can be copied; their targets still need files to load.

Static cycles through external modules back into generated output are unsupported, including cycles that return to an entry. Bundle the modules forming such cycles together or exclude their shared dependencies from inlining.

Plugins see an inlined module in the `moduleIds` and `modules` of every output chunk containing it. CSS extraction and SSR manifest plugins that assume a module belongs to one chunk need to account for this. For Vite builds, add this matcher to `experimentalInlineCommonChunks` to keep style modules from being copied into multiple stylesheets:

```js
exclude: /\.(css|scss|sass|less|styl|stylus|pcss|postcss|sss)(\?|$)/,
```

A `renderChunk` hook that embeds different values in different copies makes the first registered copy determine the value for all consumers. Exclude affected modules, as well as modules that rely on separate file-local addon bindings or their initialization order. If code relies on function or class `.name`, use [`output.keepNames`](/reference/OutputOptions.keepNames), since each copy is named alongside the code in its containing file.

:::

For more background, see the [design RFC](https://github.com/rolldown/rolldown/discussions/10693). Application results and bug reports are welcome in the [feedback issue](https://github.com/rolldown/rolldown/issues/11040).
