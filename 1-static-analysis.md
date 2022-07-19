# Surface Module Source Static Analysis

## Synopsis

Extend instances of `ModuleSource` such that they reflect certain results of
static analysis, like their `import` and `export` bindings, such that tools can
inspect module graphs.

## Dependencies

This proposal depends on [Module and ModuleSource][0] from the [Compartments
proposal](README.md) to introduce `ModuleSource`.

## Design

Extend [ModuleSource][0], such that instances have the following properties:

- `bindings`, an `Array` of `Binding`s.
- `needsImportMeta`, a `boolean` indicating that the module contains
  `import.meta` syntax.

Where a `Binding` is an ordinary `Object` with one of the valid binding shapes
for each name or wildcard (`*`) bound by `import` or `export` in the text
of the module, in their order of appearance.

- `{ import: string, from: string }`

  For example, `import { a } from 'a.js'` would produce
  `{ import: 'a', from: 'a.js' }`.

  For example, `import { a, b } from 'ab.js'` would produce:
  `{ import: 'a', from: 'ab.js' }` and
  `{ import: 'b', from: 'ab.js' }`.

- `{ import: string, as: string, from: string }`

  For example, `import { a as x } from 'a.js'` would produce
  `{ import: 'a', as: 'x', from: 'a.js' }`.

- `{ export: string }`

  For example, `export { x }` would produce
  `{ export: 'x' }`.

- `{ export: string, from: string }`

  For example, `export { x } from 'x.js'` would produce
  `{ export: 'x', from: 'x.js' }`.

- `{ export: string, as: string, from: string }`

  For example, `export { x as a } from 'x.js'` would produce
  `{ export: 'x', as: 'a', from: 'x.js' }`.

- `{ importAllFrom: string, as:  string }`

  For example, `import * as x from 'x.js'` would produce
  `{ importAllFrom: 'x.js', as: 'x' }`.

- `{ exportAllFrom: string }`

  For example, `export * from 'x.js'` would produce
  `{ exportAllFrom: 'x.js' }`.

- `{ exportAllFrom: string, as: string }`

  For example, `export * as x from 'x.js'` would produce
  `{ exportAllFrom: 'x.js', as: 'x' }`.

When using dynamic import to instantiate a `Module`, the JavaScript host will
continue to depend on the [[Module Source]] internal slot and the bindings
slots of the underlying Module Source Record for its own analysis, such that
user code cannot be confused by modifications to a `ModuleSource` instance that
might share the underlying immutable Module Source Record which in turn may be
safely shared among agents in an agent cluster.

## Motivation

A mechanism to statically analyze the shallow dependencies of a JavaScript
module will allow tools to create a module graph from module texts without
executing them, and without a heavy dependency on a full JavaScript parser.
This is the first step in many JavaScript module system tools including
build systems, bundlers, import map generators, and hot module replacement
systems, test dependency watchers.

The weight and performance of a JavaScript meta-parser (about 1MB) often
precludes production use-cases that make direct use of JavaScript module
source.
Surfacing this feature at the language level will likely allow production
systems to operate directly on JavaScript sources instead of generated
artifacts.
This would make production systems more closely resemble systems tested during
development, and make debugging production systems map more closely to
development analogues.

* bundlers ([Browserify][browserify], [WebPack][webpack], [Parcel][parcel],
  &c), virtualize loading but not evaluation of module graphs and emulate other
  host environments, like a Node.js program emulating a web browser.
* import mappers ([import-map][import-map]) like bundlers need to be able to
  collect transitive dependencies according to ECMAScript language and specific
  host behaviors.
  A ECMAScript native module loader interface would expedite evolution of import map
  runtimes in JavaScript.
* hot module replacement (HMR) systems (WebPack, SnowPack, &c), which need the
  ability to instantiate new module graphs when dependencies change and the
  ability to bequeath subgraphs to new graphs.
  * Node.js [defers][node-hmr] to ECMAScript to provide a module loader
    interface to aid HMR.
* persistent testing apparatuses ([Jest][jest]), because a persistent service
  reinstantiates whole module graphs to reconstruct tests and test subjects.
  * Jest currently resorts to exploiting Node.js's [vm][vm-context] module to
    instantiate separate realms and attempts ([and
    fails][jest-ses-interaction]) to provide the illusion of a single realm by
    patching client realms with some of the intrinsics of the host realm.

## Examples

### Analyzing a module graph

The following code produces a module graph from modules plainly published on
the web using URLs as import specifiers and memo keys.
No modules are executed.

```js
const graph = new Map();

const load = async url => {
  if (graph.has(url)) {
    return;
  }
  const response = await fetch(url);

  // Account for redirects.
  if (response.url !== url) {
    graph.set(url, new Set([response.url]));
    return load(response.url);
  }

  const edges = new Set();
  graph.set(url, edges);

  const text = await response.text();
  const source = new ModuleSource(text);

  const dependencies = [];
  for (const binding of source.bindings) {
    const from = binding.from ?? binding.importAllFrom ?? binding.exportAllFrom;
    if (from) {
      const importUrl = new URL(binding.from, url).href;
      edges.add(importUrl);
      dependencies.push(load(importUrl));
    }
  }
  await Promise.all(dependencies);
};

await load('https://example.com/example.js');
```

### Hot module replacement

Hot module replacement allows a developer to automatically reload a module when
any of its transitive dependencies change, invaliding any intermediate modules,
and allowing for graceful hand-off of module scoped stage when necessary.

<details>
  <summary>Hot module replacement sketch</summary>

  This sketch outlines how one can use `Module` and `ModuleSource` to construct
  a watcher graph that reuses these objects between reloads when possible.
  The sketch assumes the existence of a fictitious `watch` interface that is a
  parody of `fetch`, except producing a promise `changed` that will settle when
  the response is no longer valid.

  ```js
  const getImports = source => source.bindings.map(binding =>
    binding.from ??
    binding.importAllFrom ??
    binding.exportAllFrom
  ).filter(Boolean);

  const sources = new Map();
  const modules = new Map();
  const watchers = new Map();
  const states = new Map();
  const getStates = new Map();

  const invalidateModule = url => {
    const watcher = watchers.get(url);
    if (watcher) {
      watcher();
      watchers.delete(url);
    }
    modules.delete(url);
    for (const importSpecifier of getImports(source)) {
      const url = new URL(importSpecifier, url).href;
      invalidateModule(url);
    }

    // Hand-off state in preparation for an upgrade.
    const getState = getStates.get(url);
    if (getState) {
      states.set(url, getState());
      getStates.delete(url);
    }
  };

  const invalidateSource = url => {
    invalidateModule(url);
    sources.delete(url);
  };

  const importHook = async (importSpecifier, importerMeta) => {
    const url = new URL(importSpecifier, importerMeta.url).href;
    let module = modules.get(url);
    if (!module) {
      let source = sources.get(url);
      if (!source) {
        const response = await watch(url);
        response.changed.then(() => invalidateSource(url));
        const text = await response.text();
        source = new ModuleSource(text);
        sources.set(url, source);
      }
      const registerGetState = getState => {
        getStates.set(url, getState);
      };
      const state = stages.get(url);
      const importMeta = { url, state, registerGetState };
      module = new Module(source, { importHook, importMeta });
      modules.set(url, module);
    }
    return module;
  }

  const watchModule = async (url, { signal }) => {
    while (!signal.aborted) {
      const { promise, resolve } = Promise.defer();
      watchers.set(url, resolve);
      await importHook(url, import.meta);
      await promise;
      // Blink once to debounce coincident changes.
      await Promise.delay(100);
    }
  };

  const entrypoint = 'https://example.com/example.js';
  await watchModule(entrypoint);
  ```

  This assumes a protocol for state hand-off:

  ```js
  let state = import.meta.state;
  import.meta.registerGetState(() => state);
  ```

</details>

## Design Questions

Do we also need to reflect `isAsync`?
This appears to depend on whether implementations need to know
whether execution will be asynchronous before actually beginning to execute.
XS appears to have managed to implement virtual module sources without
an explicit indicator.
ECMA-262 currently has [[isAsync]] on ***Cyclic Module Record***,
which would suggest that, if engines can be implemented without knowing
a source will be asynchronous, the specification will need to be refactored to
reflect that.

## Design Rationales

The property `needsImportMeta` allows virtual import hooks to omit properties
from the `importMeta` of any `Module` instance derived from the source,
having proof that the module will never access `import.meta`.
Concretely, `import.meta.resolve` would be a closure over the module's referrer
in hosts that provide it.
In module graphs with thousands of module instances that largely do not use
this property, avoiding the allocation of per-module closures can allow a
significant reduction in memory pressure.

A similar optimization might be possible for `import`.
With the design as written, `needsImport` would only be `false` for modules
that make no use of static `import` or `export` `from` clauses and also never
use the syntactic form for dynamic `import`.
Since virtual module graphs can share relatively few `importHook` instances,
the potential savings would be negligible, so we've omitted this flag.

[0]: ./0-module-and-module-source.md
[browserify]: https://browserify.org/
[import-map]: https://github.com/WICG/import-maps
[jest-ses-interaction]: https://github.com/facebook/jest/issues/11952
[jest]: https://jestjs.io/
[node-hmr]: https://github.com/nodejs/node/issues/40594
[parcel]: https://parceljs.org/
[vm-context]: https://nodejs.org/api/vm.html#vm_vm_createcontext_contextobject_options
[webpack]: https://webpack.js.org/
