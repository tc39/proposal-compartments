# Compartments

Evaluators for modules.

**Stage**: 1

**Champions**:

* Bradley Farias, GoDaddy
* Mark S. Miller, Agoric
* Caridy Patiño, Salesforce
* Jean-Francois Paradis, Salesforce
* Patrick Soquet, Moddable
* Kris Kowal, Agoric

Agoric's [SES shim][ses-shim] and Moddable's [XS][xs-compartments] are actively
vetting this proposal as a shim and native implementation respectively (2022).
Most activity toward advancing this proposal occurs on those projects.

## Synopsis

Provide a mechanism for evaluating modules from ECMAScript module source code
and virtualizing module loader host behaviors.

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

Tools that will benefit from the ability to have multiple module graphs
in a single realm include:

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
* emulators ([JSDom][jsdom]) in which the emulated artifact may need a separate
  module memo from the surrounding realm.
* sub-realm sandboxes ([SES][ses] and [LavaMoat][lava-moat]) that virtualize
  evaluating guest modules and limit access to globals and built-in modules.
  This proposal prepares for the SES proposal to introduce `lockdown`, which
  isolates all evaluators, including `eval`, `Function`, and this `Compartment`
  module evaluator. That proposal will introduce the concern of per-compartment
  globals and hardened shared intrinsics.

Defining a module loader in the language also improves the language's ability
to evolve.
For example, a module loader interface that accounts for linking third-party
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

### Sketch

This is a rough sketch of potential interfaces.

```ts
interface ModuleExportsNamespace {}

// Module instances have a module environment record, an exotic object
// that reflects every name in the lexical scope of the module.
// The environment record does not contain a property for any names that are
// imported and reexported without a lexical binding.
// An `import name as alias` binding will have a property with the lexically bound alias. 
// An `export name as alias` binding will have a property with the lexically bound name,
// whereas the module exports namespace will have a property with the alias.
interface ModuleEnvironmentRecord {}

// Bindings reflect the `import` and `export` statements of a module.
type Binding =
  { import: '*' | string, as?: string, from: string } |
  { export: '*' | string, as?: string, from?: string };

// Compartments support ECMAScript modules and linkage to other kinds of modules,
// notably allowing for JSON or WASM.
// These must provide an initializer function and may have bindings.
// The bindings correspond to the equivalent `import` and `export` declarations
// of an ECMAScript module.
type ThirdPartyStaticModuleRecord = {
  bindings?: Array<Binding>,
  // Initializes the module if it is imported.
  // Initialize may return a promise, indicating that the module uses
  // the equivalent of top-level-await.
  initialize(environment: ModuleEnvironmentRecord, meta: Object),
  // TODO decide upon a positive or negative form of `usesMeta` or `noMeta`.
  // Avoid calling `importMetaHook` for this module since it won't use the
  // `meta` property. 
  noMeta?: boolean,
  // TODO consider whether to reflect top-level-await statically.
};

// Static module records are an opaque token representing the compilation
// of a module that can be reused across multiple compartments.
interface StaticModuleRecord {
  // Static module records can be constructed from source.
  // XS allows third-party module records and source descriptors to
  // be precompiled as well.
  constructor(source: string | { source: string } | ThirdPartyStaticModuleRecord);

  // Static module records reflect their bindings for information only.
  // Compartments use internal slots for the compiled code and bindings.
  bindings: Array<Binding>;
}

// A ModuleDescriptor captures a static module record and per-compartment metadata.
type ModuleDescriptor = 
  // Using a string for a module descriptor indicates a static module record
  // should be inherited from the parent compartment, but not the parent
  // compartment's instance.
  string
  // A static module record descriptor produces a unique instance
  // of a module in a compartment.
  | {
    record: StaticModuleRecord | ThirdPartyStaticModuleRecord,

    // An optional alias.
    // For example, in a Node.js package compartment, '.' may imply a memo entry
    // for './index.js'.
    // TODO this design does not yet account for the possibility of multiple
    // aliases, which have not been necessary for a faithful imitation of Node.js.
    // That might be better accounted for with {compartment?, specifier}
    // module descriptors, below.
    specifier?: string,

    // Properties to copy to the `import.meta` of the resulting module instance.
    meta?: Object,
  }
  // As a shorthand for constructing a StaticModuleRecord,
  // a module descriptor can provide source.
  | {
    source: string,
    specifier?: string,
    meta?: Object,
  }
  // To refer to a module instance in another compartment, we need either to
  // have a module descriptor that can refer to it, or have a `module` method and
  // use module exports namespaces as opaque tokens to refer to modules in foreign
  // compartments.
  | {
    // A compartment instance.
    // We do not preclude the possibility that compartment is the same compartment,
    // as that is possible to achieve in moduleMapHook and loadHook, albeit not
    // in moduleMap.
    // TODO Idea: it may be sufficient to make this compartment instance optional,
    // defaulting to the current instance, in order to implement arbitrary
    // numbers of aliases to a single module instance, from one or many other
    // compartments, which would obviate the need for `Compartmnt.prototype.module`.
    compartment: Compartment,
    // The full specifier of the corresponding module instance in the other compartment.
    specifier: string,
  }
  | {
    namespace: ModuleExportsNamespace,
  };

type CompartmentConstructorOptions = {
  // inherit, when true, indicates that this compartment shares the globals, global
  // contour, and evaluators of the surrounding compartment.
  // This would interact with the SES `lockdown` function: after lockdown,
  // a true would be incoherent and throw an exception.
  // When false, the compartment's globalThis would be populated only
  // with a subset of the language's intrinsics and the compartment's
  // specialized evaluators.
  // For this proposal to be fully specified, we would need to realize
  // the concept of shared intrinsics in anticipation of Lockdown.
  // In order for the module-loader Compartment to be useful for a
  // Lockdown-Compartment shim, we must disallow vendors from including
  // any shared intrinsics beyond those specified by the module loader proposal.
  inherit: boolean,

  // Globals to copy onto this compartment's unique globalThis.
  // Constructor options with globals and inherit: true would be incoherent and
  // effect an exception.
  globals: Object,

  // The compartment uses the resolveHook to synchronously elevate
  // an import specifier (as it appears in the source of a StaticModuleRecord
  // or bindings array of a third-party StaticModuleRecord), to
  // the corresponding full specifier, given the full specifier of the
  // referring (containing) module.
  // The full specifier is the memo key for the module.
  // TODO This proposal does not yet account for import assertions,
  // which evidence shows are actually import type specifiers as they were
  // originally designed and may need to also participate in the memo key.
  resolveHook: (importSpecifier: string, referrerSpecifier: string) => string,

  // The compartment's load and import methods may require to load or initialize
  // additional modules.
  // If the compartment does not have the corresponding module on hand,
  // it will first consult the moduleMap for a descriptor of the needed module.
  moduleMap?: Record<string, ModuleDescriptor>,

  // If the moduleMap does not contain a descriptor for a necessary module,
  // it will then consult the moduleMapHook.
  // The moduleMapHook is necessary for matching computed specifiers,
  // like specifiers that match a prefix and have a corresponding
  // module descriptor.
  // The moduleMap and moduleMapHook are both synchronous.
  moduleMapHook?: (fullSpecifier: string) => ModuleDescriptor?,

  // If loading or importing a module misses the compartment's memo, the
  // moduleMap, and the moduleMapHook returns undefined, the compartment
  // calls the asynchronous loadHook.
  // Note: This name differs from the implementation of SES shim and a 
  // prior revision of this proposal, where it is currently called importHook.
  loadHook?: (fullSpecifier: string) => Promise<ModuleDescriptor?>

  // TC53: Moddable implements a loadNowHook, which was necessary
  // for builds that omit promise machinery,
  // TODO but may no longer be necessary
  // given the existence of a synchronous `moduleMapHook` that
  // can return the full breadth of possible module descriptora.
  loadNowHook?: (fullSpec: FullSpecifier) => ModuleDescriptor?;

  // A ModuleDescriptor can have a `meta` property.
  // The compartment assigns these properties over the true `import.meta`
  // object for the module instance.
  // That object in turn has a null prototype.
  // However some properties of `meta` are too expensive for a host
  // to instantiate for every module, like Node.js's `import.meta.resolve`,
  // which can be avoided for modules that never utter `import.meta`
  // in their sources.
  // This hook gets called for any module that utters `import.meta`,
  // so some work can be deferred until just before the compartment
  // initializes the module.
  importMetaHook?: (meta: Object) => void,
};

interface Compartment {
  // Note: This differs from the implementations of SES shim and Moddable's XS,
  // which accept two arguments before the options bag.
  // TODO Settle #5 before commiting to a single-options bag constructor.
  // We can effect a migration by initially requiring a symbol on the first
  // argument options bag, like `[Compartment.options]: true`, in order
  // afford new code that is forward compatible while still supporting
  // legacy argument patterns.
  constructor(options?: CompartmentConstructorOptions): Compartment;

  // Accessor for this compartment's globals.
  // If inherit is true, globalThis is object identical to the incubating
  // compartment's globalThis.
  // If inherit is false, globalThis is a unique, ordinary object
  // intrinsic to this compartment.
  // The globalThis object initially has only "shared intrinsics",
  // followed by compartment-specific "eval", "Function", and "Compartment",
  // followed by any properties transferred from the "globals"
  // constructor option with the semantics of Object.assign.
  globalThis: Object,

  // Evaluates a program program in this compartment.
  // If the inherit compartment constructor option is false (or absent),
  // this function is equivalent to indirect eval in the incubating
  // compartment.
  // If the inherit compartment constructor option is true, (or absent after
  // lockdown!), this is equivalent to indirect eval with this compartment's
  // own evaluator, globals, and a temporary global contour.
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
  // memo, but does not initialize any modules.
  // The load function is useful for tools like bundlers and importmap
  // generators that load a module graph in an emulated host environment
  // but cannot and should not emulate evaluation.
  async load(fullSpecifier: string): Promise<void>

  // import induces the asynchronous load phase and then initializes
  // the given module and any of its transitive dependencies that
  // have not already begun initialization.
  async import(fullSpecifier: string): Promise<ModuleExportsNamespace>;

  // Necessary to thread a module exports namespace from this compartment into
  // the `moduleMap` Compartment constructor argument, without importing (and
  // consequently executing) the module.
  // TODO it may be possible to remove this method if module descriptors
  // with the {specifier, compartment?} shape solve the same problem better.
  module(specifier: string): ModuleNamespace;

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
  // reinitialize the underlying module or produce different namespaces.

  importNow(fullSpecifier: string): ModuleExportsNamespace;

  loadNow(fullSpecifier: string): void;
}
```

### Design Rationales

An exported value named `then` can be statically imported, but dynamic import
confuses the module namespace for a thenable object.
The resolution of the promise returned by dynamic import, in this case, is the
eventual resolution of the thenable module.
This is unlikely to be an intended effect.

Consider `thenable.js`:

```js
export function then(resolve) {
  resolve(42);
}
```

This might be dynamically imported by a neighboring module.

```js
import('./thenable.js').then((x) => {
  // x will be 42 in this case, not a module namespace object with a then
  // function.
})
```

This is the behavior of a dynamic import today, despite it being surprising.

We have chosen to embrace this hazard since it would be worse to have
dynamic import and compartment import behave differently.

[browserify]: https://browserify.org/
[import-map]: https://github.com/WICG/import-maps
[jest-ses-interaction]: https://github.com/facebook/jest/issues/11952
[jsdom]: https://www.npmjs.com/package/jsdom
[lava-moat]: https://github.com/LavaMoat/LavaMoat
[node-hmr]: https://github.com/nodejs/node/issues/40594
[parcel]: https://parceljs.org/
[ses-proposal]: https://github.com/tc39/proposal-ses
[ses-shim]: https://github.com/endojs/endo/tree/master/packages/ses
[webpack]: https://webpack.js.org/
[xs-compartments]: https://blog.moddable.com/blog/secureprivate/
[vm-context]: https://nodejs.org/api/vm.html#vm_vm_createcontext_contextobject_options
