
# Compartment

## Synopsis

Compartments provide a high-level API for orchestrating modules and evaluators,
such that programs and modules can be collectively isolated to a particular
global scope.
We expect that compartments will be used to isolate Node.js packages and import
map scopes.

Agoric's [SES shim][ses-shim], Sujitech's [ahead-of-time-SES][AOT-SES] and
Moddable's [XS][xs-compartments] are actively vetting this proposal
as a shim and native implementation respectively (2022).

This proposal is a [SES Proposal][ses-proposal] milestone.

## Motivation

Many ECMAScript module behaviors are defined by the host.
The language needs a mechanism to allow programs running on one host to fully
emulate or virtualize the module loader behaviors of another host.

Module loader behaviors defined by hosts include:

* resolving a module import specifier to a full specifier,
* locating the source for a full module specifier,
* canonicalizing module specifiers that refer to the same module instance,
* populating the `import.meta` object.

For example, on the web we can expect a URL to be a suitable full module
specifier, and for every module specifier to correspond to a URL.
We can expect the canonicalized module specifier to be reflected as
`import.meta.url`.
In Node.js, we can also expect import specifiers that are not fully qualified
URLs to receive special treatment.
However, in Moddable's XS engine, we can expect a module specifier
to resemble a UNIX file system path and not have a corresponding URL.

We can also expect to have only one module instance per canonical module
specifier in a given Realm, and for `import(specifier)` to be idempotent
for the lifetime of a Realm.
Tools that require separate module memos are therefore compelled to create
realms either using Node.js's [VM context][vm-context] or `<iframes>` and
[content security policies][csp] rather than a lighter-weight mechanism,
and consequently suffer identity discontinuties between instances from
different realms.

* sub-realm sandboxes ([SES][ses] and [LavaMoat][lava-moat]) that virtualize
  evaluating guest modules and limit access to globals and built-in modules.
  This proposal prepares for the SES proposal to introduce `lockdown`, which
  isolates all evaluators, including `eval`, `Function`, and this `Compartment`
  module evaluator. That proposal will introduce the concern of per-compartment
  globals and hardened shared intrinsics.

Defining a module loader in the language also improves the language's ability
to evolve.
For example, a module loader interface that accounts for linking "virtual"
modules that are not JavaScript facilitates easier experimentation with linkage
against languages like WASM.
For another, a module loader interface allows for user space experimentation
with the notion of [import maps][import-map].

Defining a module loader in the language also provides valuable insight to the
design of every language feature that touches upon modules, and every new
module system feature adds uncertainty to the eventual inclusion of a module
loader to the language.

One such insight is that module blocks will benefit from the notion of a module
descriptor as defined by this proposal. Module blocks roughly correspond to
compiled sources and are consequently not coupled to a particular host environment.
A module descriptor is necessary to carry properties of a module not captured
in the source, like the module referrer specifier and how to populate `import.meta`.

Additionally, having a module loader interface is a prerequisite for shimming built-in
modules.

## Interfaces

```ts
// A ModuleDescriptor captures a module source and per-compartment metadata.
type ModuleDescriptor =

  // Describes a module by referring to a *reusable* record of the compiled
  // source.
  // The module source captures a static analysis
  // of the `import` and `export` bindings and whether the source ever utters
  // the keywords `import` or `import.meta`.
  // The compartment will then asynchronously _load_ the shallow dependencies
  // of the module and memoize the promise for the
  // result of loading the module and its transitive dependencies.
  // If the compartment _imports_ the module, it will generate and memoize
  // a module instance and execute the module.
  // To execute the module, the compartment will construct an `import.meta` object.
  // If the source utters `import.meta`,
  // * The compartment will construct an `importMeta` object with a null prototype.
  // * If the module descriptor has an `importMeta` property, the compartment
  //   will copy the own properties of the descriptor's `importMeta` over
  //   the compartment's `importMeta' using `[[Set]]`.
  // The compartment will then begin initializing the module.
  // The compartment memoizes a promise for the module exports namespace
  // that will be fulfilled at a time already defined in 262 for dynamic import.
  | {
    record: ModuleSource,

    // Full specifier, which may differ from the `fullSpecifier` used
    // as the corresponding key of the `modules` Compartment constructor option,
    // or the `fullSpecifier` argument of the `loadHook` that returned
    // the module descriptor.
    // If present, this will be used as the referrerSpecifier for this record's
    // importSpecifiers instead.
    specifier?: string,

    // Properties to copy to the `import.meta` of the resulting module instance,
    // if the source mentions `import.meta`.
    importMeta?: Object,
  }

  // Describes a module by a *reusable* virtual module source.
  // When the compartment _loads_ the module, it will use the bindings array
  // of the virtual module source to discover shallow dependencies
  // in lieu of compiling the source, and otherwise behaves identically
  // to { source } descriptors.
  // When the compartment _imports_ the module, it will construct a
  // [[ModuleEnvironmentRecord]] and [[ModuleExportsNamespace]] based
  // entirely on the bindings array of the virtual module source.
  // If the virtual module source has a true `needsImport`, the
  // compartment will construct an `import` that resolves
  // the given import specifier relative to the module instance's full specifier
  // and returns a promise for the module exports namespace of the
  // imported module.
  // If the virtual module source has a true `needsImportMeta`
  // property, the compartment will construct an `importMeta` by the
  // same process as any { source } module.
  // `importMeta` will otherwise be `undefined`.
  // The compartment will then execute the module by calling the
  // `execute` function of the virtual module source,
  // giving it the module environment record and an options bag containing
  // either `import` or `importMeta` if needed.
  // The compartment memoizes a promise for the module exports namespace
  // that is fulfilled as already specified in 262 for dynamic import,
  // where the behavior if the execute function is async is analogous to
  // top-level-await for a module compiled from source.
  | {
    source: VirtualModuleSource,

    // See above.
    specifier?: string,

    // See above.
    importMeta?: Object,
  }

  // Use a module source from the compartment in which this compartment
  // was constructed.
  // If the compartment _loads_ the corresponding module, the
  // module source will be synchronously available only if it is already
  // in the parent compartment's synchronous memo.
  // Otherwise, loading induces the parent compartment to load the module,
  // but does not induce the parent compartment to load the module's transitive
  // dependencies.
  // This compartment then resolves the shallow dependencies according
  // to its own resolveHook and loads the consequent transitive dependencies.
  // If the compartment _imports_ the module, the behavior is equivalent
  // to loading for the retrieved module source or virtual
  // module source.
  | {
    source: string,

    // Properties to copy to the `import.meta` of the resulting module instance.
    importMeta?: Object,
  }

  // FOR SHARED MODULE INSTANCES:

  // To create an alias to an existing module instance given the full specifier
  // of the module in a different compartment:
  | {
    namespace: string,

    // The compartment from which to draw the module instance,
    // by default, this compartment.
    compartment: Compartment,
  }

  // To create an alias to an existing module instance given a module exports
  // namespace object that can pass a brand check:
  | {
    namespace: ModuleExportsNamespace,
  }

  // If the given namespace does not pass a ModuleExportsNamespace brandcheck,
  // the compartment will not support live bindings to the properties of that
  // object, but will instead produce an emulation of a module exports namespace,
  // using a frozen snapshot of the own properties of the given object.
  // The module exports namespace for this module does not reflect any future
  // changes to the shape of the given object.
  | {
    // Any object that does not pass a module exports namespace brand check:
    namespace: ^ModuleExportsNamespace,
  };

type CompartmentConstructorOptions = {
  // Globals to copy onto this compartment's unique globalThis.
  globals: Object,

  // The compartment uses the resolveHook to synchronously elevate
  // an import specifier (as it appears in the source of a ModuleSource
  // or bindings array of a VirtualModuleSource), to
  // the corresponding full specifier, given the full specifier of the
  // referring (containing) module.
  // The full specifier is the memo key for the module.
  resolveHook: (importSpecifier: string, referrerSpecifier: string) => string,

  // The compartment's load and import methods may need to load or execute
  // additional modules.
  // If the compartment does not have a module on hand,
  // it will first consult the `modules` object for a descriptor of the needed module.
  modules?: Record<string, ModuleDescriptor>,

  // If loading or importing a module misses the compartment's memo and the
  // `modules` table, the compartment calls the asynchronous `loadHook`.
  // Note: This name differs from the implementation of SES shim and a
  // prior revision of this proposal, where it is currently called `importHook`.
  loadHook?: (fullSpecifier: string) => Promise<ModuleDescriptor?>
};

interface Compartment {
  // Note: This single-argument form differs from earlier proposal versions,
  // implementations of SES shim, and Moddable's XS, which accept three arguments,
  // including a final options bag.
  constructor(options?: CompartmentConstructorOptions): Compartment;

  // Accessor for this compartment's globals.
  // The globalThis object initially has only "shared intrinsics",
  // followed by compartment-specific "eval", "Function", "Module", and
  // "Compartment", followed by any properties transferred from the "globals"
  // constructor option with the semantics of Object.assign.
  globalThis: Object,

  // Evaluates a program program using this compartment's associated
  // evaluators.
  // A subsequent proposal might add options to either the compartment
  // constructor or the evaluate method to persist the global contour
  // between calls to evaluate, making compartments a suitable
  // tool for a REPL, such that a const or let binding from one evaluation
  // persists to the next.
  // Subsequent proposals might afford other modes, like sloppy globals mode.
  // TODO is sloppyGlobalsMode only sensible in the context of the shim?
  evaluate(source: string): any;

  // load causes a compartment to load module descriptors for the
  // transitive dependencies of a specified module into its
  // memo, but does not execute any modules.
  // The load function is useful for tools like bundlers and importmap
  // generators that load a module graph in an emulated host environment
  // but cannot and should not emulate evaluation.
  async load(fullSpecifier: string): Promise<void>

  // import induces the asynchronous load phase and then executes
  // the given module and any of its transitive dependencies that
  // have not already begun initialization.
  async import(fullSpecifier: string): Promise<ModuleExportsNamespace>;

  // TC53: some embedded systems hosts exclude promises and so require these
  // synchronous variants of import and load.
  // On hosts where these functions cannot succeed synchronously,
  // they must throw an error without the side effect of initializing any
  // additional modules.
  // For modules that have a top-level await, these would return
  // after the module first awaits but not after any subsequent promise queue
  // job.

  // If a host supports both importNow and import, they must share
  // a memo of full specifiers to promises for the module exports namespace.
  // importNow and loadNow followed by import or load must not accidentally
  // reexecutes the underlying module or produce different namespaces.

  importNow(fullSpecifier: string): ModuleExportsNamespace;

  loadNow(fullSpecifier: string): void;
}
```

## User Code

Compartments can be implemented in user-code.
This is a partial and untested sketch.

<details>
  <summary>Compartment in user code</summary>

  ```js
  class Compartment {
    #modules = new Map();
    #descriptors = new Map();
    #referrers = new Map();
    #globalThis = Object.create(null);

    constructor({ resolveHook, loadHook, globals }) {
      this.#resolveHook = resolveHook;
      this.#loadHook = loadHook;
      this.#importHook = async (importSpecifier, importMeta) => {
        const referrerSpecifier = this.#referrers.get(importMeta);
        const fullSpecifier = this.#resolveHook(
          importSpecifier,
          referrerSpecifier
        );
        return this.#load(fullSpecifier);
      };
      this.#evaluators = new Evaluators({
        globalThis: this.#globalThis,
        importHook: this.#importHook
      });
      // Copy eval, Function, and Module into the associated globalThis.
      Object.assign(this.#globalThis, this.#evaluators);
      Object.assign(this.#globalThis, globals);
    }

    get globalThis() {
      return this.#globalThis;
    }

    evaluate(script) {
      return this.#evaluators.eval(script);
    }

    async #descriptor(specifier) {
      let eventualDescriptor = this.#descriptors.get(specifier);
      if (!eventualDescriptocr) {
        eventualDescriptor = this.#loadHook(specifier);
        this.#descriptors.set(specifier, eventualDescriptor);
      }
      return eventualDescriptor;
    }

    async load(specifier) {
      let eventualModule = this.#modules.get(specifier);
      if (!eventualModule) {
        const descriptor = await this.#descriptor(specifier);
        eventualModule = await this.#load(descriptor);
        this.#modules.set(specifier, eventualModule);
      }
      return eventualModule;
    }

    async #load(descriptor, specifier) {
      if ("source" in descriptor) {
        const importMeta = Object.create(null);
        Object.assign(importMeta, descriptor.importMeta);
        this.#referrers.set(importMeta, specifier);
        return new this.#evaluators.Module(
          descriptor.source,
          {
            importHook: this.#importHook,
            importMeta,
          }
        );
      } else if ("specifier" in descriptor) {
        const compartment = descriptor.compartment || this;
        return compartment.load(descriptor);
      } else if ("module" in descriptor) {
        return descriptor.module;
      } else if ("namespace" in descriptor) {
        // Contingent on a Module.get utility that obtains a `Module`
        // reference for a brand-checked namespace.
        const module = Module.get(descriptor.namespace);
        if (module !== undefined) {
          return module;
        } else
          // Contingent on virtual module source protocol amendment.
          return this.#moduleForObject(descriptor.namespace);
        }
      } else {
        throw new Error("Weird descriptor");
      }
    }

    async import(specifier) {
      return import(await this.#load(specifier));
    }

    // Contingent on virtual module source protocol amendment.
    #moduleForObject(exports) {
      return new this.#evaluators.Module({
        bindings: Reflect.keys(exports).map(name => ({ exports: name })),
        execute(imports) {
          Object.assign(imports, exports);
        },
      });
    }
  }
  ```
</details>

Since compartments can be implemented in terms of the lower-numbered layers of
this proposal, it may no longer be necessary for the language to provide a the
Compartment constructor.
However, one of the primary motivations is being able to use compartments to
evolve import maps in user code.
For a user-code implementation of an import-map runtime to be viable, it must
be sufficiently terse to be inlined in a script block.
The above code demonstrates how much such a runtime would likely weigh.

## Motivating Examples and Design Rationales

### Multiple-instantiation

This example illustrates the use of a new compartment to support multiple
instantiation of modules, reusing the host's compartment and module source
memos as a cache.
This example creates five instances of the example module and its transitive
dependencies.

```js
for (let i = 0; i < 5; i += 1) {
  new Compartment().import('https://example.com/example.js');
}
```

Assuming that the language separately adopted hypothetical `import static`
syntax to defer execution but load a module and its transitive dependencies, a
bundler would be able to observe the need to capture the example module and its
transitive dependencies, such that the *only* instances are in guest compartments.

```js
import static 'https://example.com/example.js';
for (let i = 0; i < 5; i += 1) {
  new Compartment().import('https://example.com/example.js');
}
```

### Virtualized web compartment

This example illustrates a very reductive emulation of a web-based compartment.
The module specifier domain is strictly URLs and import specifiers are resolved
relative to the referrer module specifier using URL resolution.
The compartment populates `import.meta.url` with the response URL.

The compartment also ensures that the import specifiers of whatever module is
loaded get resolved relative to the physical location of the resource.
If the response URL shows that the fetch followed redirects, the `loadHook`
returns a reference (`{instance: response.url}`) to the actual module instead
of returning a record.
The compartment then follows-up with a request for the redirected location.

```js
const compartment = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    return new URL(importSpecifier, referrerSpecifier).href;
  },
  async loadHook(url) {
    const response = await fetch(url, { redirect: 'manual' });
    if (response.url !== url) {
      return { instance: response.url };
    }
    const source = await response.text();
    return {
      source: new ModuleSource(source),
      importMeta: { url: response.url },
    };
  },
});
await compartment.import('https://example.com/example.js');
```

By returning an alias module descriptor, the compartment can ensure
that requests for both the request URL and the response URL refer
to the canonicalized module.

---

For the same intended effect but a single fetch, we might
alternately use a `specifier` property of record module descriptors.
In the following example, both the request URL and the response URL
would realize cache keys in the compartment.

```js
const compartment = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    return new URL(importSpecifier, referrerSpecifier).href;
  },
  async loadHook(url) {
    const response = await fetch(url);
    const source = await response.text();
    return {
      source: new ModuleSource(source),
      specifier: response.url,
      importMeta: { url: response.url },
    };
  },
});
await compartment.import('https://example.com/example.js');
```

So, we guide the host to instead return the new cache key so the compartment
can memoize the `loadHook`, sending a single request for the canonicalized
module specifier.

A design tension with the `specifier` property is that it invites a race
between two concurrent `loadHook` calls for different specifiers that
converge on the same response specifier.
Implementations must take care to ensure that the module record memo and the
module record *promise* memo point to the same record for a given ultimate full
specifier.

### Virtualized Node.js compartment

This example illustrates a very reductive emulation of a Node.js compartment.
As in a web based loader, the domain of full module specifiers is fully
qualified URLs.

However, Node.js import specifiers are rarely fully qualified URL's.
Instead, we distinguish *relative module specifiers* (those starting with `.`
or `..` path components) from *absolute module specifiers* (all others).
In turn, absolute module specifiers can refer either to Node.js built-in
modules or links to modules in other *packages*.

> For the purpose of this proposal, we beg a distinction between the
> Node.js-specific terms *relative* and *absolute*, and the
> Compartment-specific terms *full module specifier* (a suitable memo key in a
> compartment) and *import module specifier* (a specifier as might appear in a
> static import or export statement or be passed to dynamic `import`, which
> must be promoted to a full specifier in the context of the full specifier
> of the referrer module (the *referrer specifier*).

The `resolveHook` is synchronous, and as such is not in a position
to search the file system for a package in an ancestor `node_modules` directory
that matches the prefix of the absolute import specifier, and failing that,
to indicate a built-in Node.js module.
So, instead, the `resolveHook` produces a URL based on the referrer specifier
and carries the import specifier in the fragment.

For the purposes of this example, we do not consider the case that an import
specifier may be a fully qualified URL.

```js
import url from 'node:url';
import fs from 'node:fs/promises';

const compartment = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    if (importSpecifier.startsWith('./') || importSpecifier.startsWith('../')) {
      return new URL(importSpecifier, referrerSpecifier).href;
    } else if (importSpecifier.startsWith('node:')) {
      return importSpecifier;
    } else {
      return new URL(`#${importSpecifier}`, referrerSpecifier);
    }
  },
  async loadHook(fullSpecifier) {
    const { protocol, pathname, hash } = new URL(fullSpecifier);
    if (protocol === 'node:') {
      return { namespace: await import(fullSpecifier) };
    }
    if (hash !== undefined) {
      const packageDescriptor = findDescriptor(
      
    }
    const path = url.fileURLToPath(fullSpecifier);
    const link = await fs.readlink(path).catch(error => {
      if (error.code === 'EINVAL') {
        return undefined;
      } else {
        throw error;
      }
    });
    if (link !== undefined) {
      return { 
    }
    const source = await fs.readFile(path);
    return {
      source: new ModuleSource(source),
      importMeta: { url: response.url },
    };
  },
});
await compartment.import('file://usr/share/node_modules/example/example.js');
```

### Bundling or archiving

Compartments can be employed to virtualize a foreign environment and generate
bundles, archives, or other stored forms of a program for transmission and
deferred execution.

This first snippet is a minimal bundler.
It differs from the first minimal example only in that it generates a `sources`
Map and calls `load` instead of `import`, ensuring we do not attempt to run any
of the loaded code locally.

```js
const sources = new Map();
const compartment = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    return new URL(importSpecifier, referrerSpecifier).href;
  },
  async loadHook(fullSpecifier) {
    const response = await fetch(fullSpecifier);
    const source = await response.text();
    sources.set(fullSpecifier, source);
    return {
      source: new ModuleSource(source),
      importMeta: { url: response.url },
    };
  },
});
await compartment.load('https://example.com/example.js');
```

Then, we presumably serialize the `sources` map and recreate it in another
environment.
This next figure uses the `sources` to reconstruct the original compartment
compartment module graph and execute it.

```js
const evaluator = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    return new URL(importSpecifier, referrerSpecifier).href;
  },
  loadHook(fullSpecifier) {
    const source = sources.get(fullSpecifier);
    if (source === undefined) {
      throw new Error('Assertion failed: incomplete sources');
    }
    return {
      source: new ModuleSource(source),
      importMeta: { url: response.url },
    };
  },
});
await evaluator.import('https://example.com/example.js');
```

### Inter-compartment linkage

One motivating use of compartments is to isolate Node.js-style packages and
limit their access to powerful modules and globals, to mitigate software supply
chain attacks.
With such an application, we would construct a special compartment for each package
and allow compartments to link modules across compartment boundaries.

In this trivial example, we construct a pair of compartments, `even` and `odd`,
which in turn contain mutually dependent `even` and `odd` modules that
participate in a dependency cycle.
For simplicity, the domain of module specifiers is exactly the names `even` and
`odd`, and these compartments do not support resolution.

These compartments use `{ instance, compartment }` module descriptors to indicate
linkage across compartment boundaries.

```js
const even = new Compartment({
  resolveHook: specifier => specifier,
  loadHook: async specifier => {
    if (specifier === 'even') {
      return { source: new ModuleSource(`
        import isOdd from 'odd';
        export default n => n === 0 || isOdd(n - 1);
      `) };
    } else if (specifier === 'odd') {
      return { instance: specifier, compartment: odd };
    } else {
      throw new Error(`No such module ${specifier}`);
    }
  },
});

const odd = new Comaprtment({
  resolveHook: specifier => specifier,
  loadHook: async specifier => {
    if (specifier === 'odd') {
      return { source: new ModuleSource(`
        import isEven from 'even';
        export default n => n !== 0 && isEven(n - 1);
      `) };
    } else if (specifier === 'even') {
      return { instance: specifier, compartment: even };
    } else {
      throw new Error(`No such module ${specifier}`);
    }
  },
});
```

An alternative design that Agoric's SES shim and XS's native Compartment
explored used module exports namespace objects as handles that could be passed
between compatment hooks.
However, to support the bundler use case, it became necessary to add a method
that could get a module exports namespace object for a module that had not yet
been loaded (`compartment.module(specifier)`, much less instantiated, nor
executed.
The invention of a module descriptor allowed us to remove this complication,
among others: the `moduleMapHook` became superfluous since both the `modules`
constructor option and the `loadHook` could use module descriptors instead.

### Linking with a virtual module source (JSON example)

To support non-JavaScript languages, a compartment provides a `loadHook` that
returns virtual module source implementations.
This example virtual module source declares its bindings (equivalent to
`export default` in this case) and provides an executor.
The executor receives a module environment record according to the shape
declared in its bindings.
This compartment makes the simplifying assumption that all modules are JSON.
A more elaborate version of this example will switch on the response MIME type
and account for import assertions.

```js
const compartment = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    return new URL(importSpecifier, referrerSpecifier).href;
  },
  async loadHook(fullSpecifier) {
    const response = await fetch(fullSpecifier);
    const source = await response.text();
    const source = {
      bindings: [
        { export: 'default' },
      ],
      execute(env) {
        env.default = JSON.parse(source);
      }
    };
    return { source };
  },
});
await compartment.import('https://example.com/example.json');
```

### Export Aliases and Module Imports Namespace

This example contrasts the properties of module imports namespace and a module
exports namespace when bindings contain aliases.

```js
const compartment = new Compartment({
  resolveHook(importSpecifier, referrerSpecifier) {
    return new URL(importSpecifier, referrerSpecifier).href;
  },
  loadHook(fullSpecifier) {
    const record = {
      bindings: [
        {export: 'internal', as: 'external'},
      ],
      execute(env) {
        env.internal = JSON.parse(source); // <----
      }
    };
    return { record };
  },
});
const fullSpecifier = 'https://example.com/example.js'
await compartment.load();
const { external } = compartment.importNow(fullSpecifier);
//      ^-----
```

### Virtual module source reexports

This example illustrates how a virtual module source can simply
reexport another module with no special logic in an executor.
This example makes the simplifying assumption that the compartment
does not support relative module specifiers and that module specifiers
are arbitrary names.

```js
const compartment = new Compartment({
  resolveHook(specifier) {
    return specifier;
  },
  loadHook(specifier) {
    switch (specifier) {
    case 'alex':
      return { namespace: { a: 10, b: 20, c: 30 } },
    case 'blake':
      return { source: { bindings: { exportAllFrom: 'alex' } } };
    }
  },
});

const { a, b, c } = await compartment.import('blake');
```

### Thenable Module Hazard

An exported value named `then` can be statically imported, but dynamic import
confuses the module namespace for a thenable object.
The resolution of the promise returned by dynamic import, in this case, is the
eventual resolution of the thenable module.
And the eventual resolution is unlikely to be an intended effect.

Consider `thenable.js`:

```js
export function then(resolve) {
  resolve(42);
}
```

A neighboring module might dynamically import this.

```js
import('./thenable.js').then((x) => {
  // x will be 42 in this case, not a module namespace object with a then
  // function.
})
```

This is the behavior of a dynamic import today, despite it being surprising.

We have chosen to embrace this hazard since it would be worse to have
dynamic import and compartment import behave differently.

However, with `compartment.importNow`, a program can mitigate this hazard.
With `importNow`, the following program will not invoke the `then`
function exported by `'./thenable.js'`.

```js
await compartment.load('./thenable.js');
const thenableNamespace = compartment.importNow('./thenable.js');
```

## Design Questions

### User code or native code

There are some reasons to make native Compartments that are not fully addressed
by the lower-level primtiives out of which they can be implemented in user
code.

1. A native implementation may be able to avoid reifying some intermediate
   objects, which may be important for embedded systems.
2. A higher-level API will be more approachable to a more casual user.
3. The runtime for a bundler in a web page might be considerably lighter in
   terms of Compartment than it might be in terms of the consitituent objects.
   Bundler runtimes need to be as small as possible to meet the needs of
   webpage delivery performance.

[browserify]: https://browserify.org/
[import-map]: https://github.com/WICG/import-maps
[lava-moat]: https://github.com/LavaMoat/LavaMoat
[node-hmr]: https://github.com/nodejs/node/issues/40594
[ses-proposal]: https://github.com/tc39/proposal-ses
[ses-shim]: https://github.com/endojs/endo/tree/master/packages/ses
[AOT-SES]: https://github.com/DimensionDev/aot-secure-ecmascript
[webpack]: https://webpack.js.org/
[xs-compartments]: https://blog.moddable.com/blog/secureprivate/
[vm-context]: https://nodejs.org/api/vm.html#vm_vm_createcontext_contextobject_options
[redirect-manual]: https://fetch.spec.whatwg.org/#concept-filtered-response-opaque-redirect
