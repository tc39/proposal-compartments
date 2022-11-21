# First-class Module and ModuleSource

[Specification Changes][0-spec]

## Synopsis

Provide first-class `Module` and `ModuleSource` constructors and extend dynamic
import to operate on `Module` instances.

A `ModuleSource` represents the result of compiling EcmaScript module source
text.
A `Module` instance represents the lifecycle of a EcmaScript module and allows
virtual import behavior.
Multiple `Module` instances can share a common `ModuleSource`.

## Interfaces

### ModuleSource

```ts
interface ModuleSource {
  constructor(source: string);
}
```

Semantics: A `ModuleSource` instance gives the holder no powers.
It represents a compiled EcmaScript module and does not capture any information
beyond what can be inferred from a module's source text.
Import specifiers in a module's source text cannot be interpreted without
further information.

Note 1: `ModuleSource` does for modules what `eval` already does for scripts.
We expect content security policies to treat module sources similarly.
A `ModuleSource` instance constructed from text would not have an associated
origin.
A `ModuleSource` instance can be constructed from vetted text ([W3C Trusted
Types][trusted-types]) and host-defined import hooks may reveal module sources
that were vetted behind the scenes.

Note 2: Multiple `Module` instances can be constructed from a single `ModuleSource`,
producing one exports namespaces for each imported `Module` instance.

Note 3: The internal record of a `ModuleSource` instance is immutable and
serializable.
This data can be shared without cost between realms of an agent or even agents
of an agent cluster.

### Module instances

```ts
type ImportSpecifier = string;

type ImportHook = (this: ModuleHandler, specifier: ImportSpecifier) =>
  Promise<Module>;

type ImportMeta = {
  __proto__: null,
  // ...
  [name: string | symbol | number]: unknown,
};

type ImportMetaHook = (this: ModuleHandler, importMeta: ImportMeta) => any;

type ModuleHandler = {
  importHook?: ImportHook,
  importMetaHook?: ImportMetaHook,
  // ...
  [name: string | symbol | number]: unknown,
};

interface Module {
  constructor(
    source: ModuleSource,
    handler: ModuleHandler,
  );

  readonly source?: ModuleSource,
}
```

Semantics: A `Module` instance has an internal ***Module Record***.
Importing the module will consistently produce the same ***Module Namespace
Exotic Object***.

The module has a lifecycle and fresh instances have not been linked,
initialized, or executed.

Invoking dynamic import on a `Module` instance attempts to advance it and its
transitive dependencies to their end state.
Consistent with dynamic import for a stringly-named module,
dynamic import on a `Module` instance produces a promise for the corresponding
***Module Namespace Exotic Object***

Dynamic import induces calls to `importHook` for each unsatisfied dependency of
each module instance in separate events, before any dependency advances to the
link phase of its lifecycle.

Dynamic import within the evaluation of a `Module` also invokes the
`importHook`.

`Module` instances memoize the result of their `importHook` keyed on the given
Import Specifier.

`Module` constructors, like `Function` constructors, are bound to a realm
and evaluate modules in their particular realm.

## Examples

### Import Kicker

Any dynamic import function is suitable for initializing, linking, and
evaluating a module instance.
This necessarily implies advancing all of its transitive dependencies to their
terminal state or any one into a failed state.

```js
const source = new ModuleSource(``);
const instance = new Module(source);
const namespace = await import(instance);
```

### Module Idempotency

Since the Module has a bound module namespace exotic object, importing the same
instance must yield the same result:

```js
const source = new ModuleSource(``);
const instance = new Module(source);
const namespace1 = await import(instance);
const namespace2 = await import(instance);
namespace1 === namespace2; // true
```

### Reusing ModuleSource

Module sources are backed by a shared immutable module source record
that can be instantiated multiple times, even locally.
Multiple `Module` instances can share a module source and produce
separate module namespaces.

```js
const source = new ModuleSource(``);
const instance1 = new Module(source);
const instance2 = new Module(source);
instance1 === instance2; // false
const namespace1 = await import(instance1);
const namespace2 = await import(instance2);
namespace1 === namespace2; // false
```

### Intersection Semantics with Module Blocks

Proposal: https://github.com/tc39/proposal-js-module-blocks

In relation to module blocks, we can extend the proposal to accommodate both,
the concept of a module block instance and module block source:

```js
const instance = module {};
instance instanceof Module;
instance.source instanceof ModuleSource;
const namespace = await import(instance);
```

### Intersection Semantics with deferred execution

The possibility to load the source, and create the instance with the default
`importHook` and the `import.meta` of the importer, that can be imported at any
given time, is sufficient:

```js
// static
import module example from 'example.js';
// or dynamic
const example = await import.module('example.js');

example instanceof Module;
example.source instanceof ModuleSource;
const namespace = await import(example);
```

If the goal is to also control the `importHook` and the `importMeta` of the
importer, then the program can use the `source` property of the unexecuted
module and construct a new `Module`.

```ts
import module example from 'example.js';
module.source instanceof ModuleSource;
const instance = new Module(module.source, { importHook });
const namespace = await import(instance);
```

In these cases, the `ModuleSource` may benefit from the origin
being captured in host data, such that it can be evaluated
under a no-unsafe-eval Content-Security-Policy.

### Virtual host import behavior

The `Module` constructor takes an optional second argument to override the
host-defined behavior for importing its shallow dependencies.
The second argument is a handler object.
The `Module` constructor eagerly captures the relevant methods
like `importHook` and `importMetaHook`, which are all optional.

```js
class ModuleHandler {
  importHook(specifier) {
    // Defer to the host:
    return import.module(specifier);

    // Or, make a separate instance of the same:
    const dependency = await import.module(specifier);
    return new Module(dependency.source);

    // Or, use a block:
    return module {};

    // Or, make a new Module from whole cloth:
    const response = await fetch(specifier);
    const text = await response.text();
    const source = new ModuleSource(text);
    return new Module(source);
  }
}

const source = new ModuleSource(`import foo from 'foo'`);
const module = new Module(source, new ModuleHandler());
await import(module);
```

In this example, the `importHook` will be called once with `"foo"` and must
return a `Module` instance by any of the available means.

For an `importHook` to receive `this` requires `importHook` to be defined as a
`class` method, a `function` declaration, or a concise method and that, unlike
a `Proxy` and perhaps against the grain of expectations, changing the
`importHook` property does not alter the behavior of any `Module` instance that
has already captured the `importHook` property of the module handler when it
was constructed.

### Virtual Host relative import behavior

The above example does not account for relative import specifiers.
For example, to emulate a Node.js module, an absolute specifier refers
either to a nearby package dependency or a built-in module, in order of
precedence, but a relative specifier (starting with `"./"` or `"../"`)
gets resolved using path math relative to the physical location of the
containing module, the `referrer`.

The `Module` instance will call its hooks as methods of the module handler
object, so module handlers are free to carry additional metadata.
This variation on a `ModuleHandler` carries an additional `url` property
for import resolution and assumes an appropriate `resolve` function.

```js
class ModuleHandler {
  async importHook(specifier) {
    const url = resolve(specifier, this.url);
    // Use the host's loader.
    const module = import.module(url);
    // But create separate instances, transitively.
    return new Module(module.source, new ModuleHandler(url));
  }
}

const { source } = await import.module(entryUrl);
const module = new Module(source, new ModuleHandler(entryUrl));
```

In this example, `import.module` uses the host's import behavior, actual or
virtual.
If the host is a virtual host, that involves the surrounding module's `Module`
instance's `importHook`.

### Virtual host `import.meta` behavior

A `ModuleHandler` may implement an `importMetaHook` that receives
an empty object with a null prototype that it may embellish
the first time (if at all) a module evaluates an `import.meta`
expression.
This example extends the prior handler implementation such
that it reveals the module's own URL.

```js
class MetaModuleHandler extends ModuleHandler {
  importMetaHook(importMeta) {
    importMeta.url = this.url;
  }
}
```

### Virtual host import assertion behavior

Open issue: https://github.com/tc39/proposal-compartments/issues/37

### Intersection Semantics with import.meta.resolve()

Proposal: https://github.com/whatwg/html/pull/5572

```ts

class ModuleHandler {
  constructor(url) {
    this.url = url;
  }
  async importHook(specifier) {
    const url = import.meta.resolve(specifier, this.url);
    const response = await fetch(url);
    const sourceText = await response.text();
    const source = new ModuleSource(sourceText);
    return new Module(source, new ModuleHandler(url));
  },
  importMetaHook(importMeta) {
    importMeta.url = this.url;
    importMeta.resolve = (specifier, referrer = this.url) =>
      import.meta.resolve(specifier, referrer);
  },
}

const source = new ModuleSource(`export foo from './foo.js'`);
const instance = new Module(source, new ModuleHandler(import.meta.url));
const namespace = await import(instance);
```

In this example, we implement a module handler class that resolves import
specifiers from a given URL, using the host-provided resolver,
`import.meta.resolve`.
This resolver, in its two-argument form, allows us to resolve specifiers at
arbitrary locations.

The first time a guest module uses its own `import.meta`, the virtualized
`importMetaHook` will create a closure for its own emulation of
`import.meta.resolve`, supporting both one- and two-argument forms.
The `importMetaHook` allows us to avoid this expensive per-module allocation
except in modules that absolutely need it.


## Design

A ***Module Source Record*** is an abstract class for immutable representations
of the dependencies, bindings, initialization, and execution behavior of a
module.

Host-defined import-hooks may specialize module source records with annotations
such that the host can enforce content-security-policies.

A ***EcmaScript Module Source Record*** is a concrete ***Module Source
Record*** for EcmaScript modules.

`ModuleSource` is a constructor that accepts EcmaScript module source text and
produces an object with a [[ModuleSource]] slot referring to an ***EcmaScript
Module Source Record***.

`ModuleSource` instances are handles on the result of compiling a EcmaScript
module's source text.
A module source has a [[ModuleSource]] internal slot that refers to a
***Module Source Record***.
Multiple `Module` instances can share a common module source.

Module source records only capture information that can be inferred from static
analysis of the module's source text.

Multiple `ModuleSource` instances can share a common ***Module Source Record***
since these are immutable and so hosts have the option of sharing them between
realms of an agent and even agents of an agent cluster.

The `Module` constructor accepts a source and  
A `Module` has a 1-1-1-1 relationship with a ***Module Environment Record***,
a ***Module Source Record***, and a ***Module Exports Namespace Exotic Object***.

## Design Rationales

### Should `importHook` be synchronous or asynchronous?

When a source module imports from a module specifier, you might not have the
source at hand to create the corresponding `Module` to be returned. If
`importHook` is synchronous, then you must have the source ready when the
`importHook` is invoked for each dependency.

Since the `importHook` is only triggered via the kicker (`import(instance)`),
going async there has no implications whatsoever.
In prior iterations of this, the user was responsible for loop thru the
dependencies, and prepare the instance before kicking the next phase, that's
not longer the case here, where the level of control on the different phases is
limited to the invocation of the `importHook`.

### Can cycles be represented?

Yes, `importHook` can return a `Module` that was either `import()` already or
was returned by an `importHook` already.

### Idempotency of dynamic imports in ModuleSource

Any `import()` statement inside a module source will result of a possible
`importHook` invocation on the `Module`, and the decision on whether or not to
call the `importHook` depends on whether or not the `Module` has already
invoked it for the `specifier` in question. So, a `Module`
most keep a map for every `specifier` and its corresponding `Module` to
guarantee the idempotency of those static and dynamic import statements.

User-defined and host-defined import-hooks will likely enforce stronger
consistency between import behavior across module instances, but module
instances enforce local consistency and some consistency in aggregate by
induction of other modules.

### toString

Whether `ModduleSource` instances retain the original source may vary by host
and modules should reuse
[HostHasSourceTextAvailable](https://tc39.es/ecma262/#sec-hosthassourcetextavailable)
in deciding whether to reveal their source, as they might with a `toString`
method.
The [module block][module-blocks] proposal may necessitate the retention of
text on certain hosts so that hosts can transmit use sources in their serial
representation of a module, as could be an extension to `structuredClone`.

### Factoring ECMA-262

This proposal decouples a new ***Module Source Record*** and ***EcmaScript
Module Source Record*** from the existing ***Module Record*** class hierarchy
and introduces a concrete ***Virtual Module Record***.
The hope if not expectation is that this refactoring makes evident that
***Virtual Module Record***, ***Cyclic Module Record***, and the abstract
base class ***Module Record*** could be refactored into a single concrete
record (***Module Record***) since all meaningful variation can be expressed
with implementations of the abstract ***Module Source Record***.
But, this proposal does not go so far as to make that refactoring normative.

This proposal does not impose on host-defined import behavior.

### Referrer Specifier

The host-defined or virtual-host-defined import hook for a module is
responsible for taking "import specifiers", the string expressed in both static
and dynamic import statements, and resolving them relative to the logical or
physical specifier for the module.
Although it is possible to construct a host or virtual-host that only supports
absolute or fully-qualified specifiers, nearly every module will have a
referrer of one kind or another.

A module will typically have three distinct but often coincident data:

- This `referrer` concept, the basis for resolving import specifiers.
- The `import.meta.url`, the basis for locating resources adjacent to modules
  hosted on the web using URL math.
  The `import.meta` object should not have a `url` property unless it
  is useful for that purpose, and as such should not exist for modules
  located in bundles or archives, and should not be used in modules that
  will be bundled or archived, but will often be identical to the `referrer`
  when it is present.
- The origin of the source, the basis for a host deciding whether to allow the
  module to be evaluated when a Content-Security-Policy is in place.
  The origin will often be identical to the origin of `import.meta.url`.
  The origin will exist in host data of the module source and must
  not be virtualizable, lest a virtual host be able to escape a same origin
  policy.

So, although these values are often related, they must be independent features.
Virtual host implementations cannot necessarily rely on the `import.meta.url`
to exist and virtual host implementations must be able to virtualize the
referrer.

The authors of this proposal vacilated between versions that made the referrer
explicit or implicit.
All of the authors shared a goal to make the `Module` and `ModuleSource`
features incrementally explainable.
Since it is possible to construct a trivial module system without the concept
of a referrer, a graduated tutorial introducing this feature (like this
explainer!) should be able to defer or formally ignore referral until it
becomes necessary to explain relative module specifier resolution.

To that end, we agreed that the referrer should be optional and appear later
in function signatures than other more essential parameters, if at all.
We relegated the `referrer` to at least be a property of the `Module`
constructor's options bag.
We then realized that the options bag was more analogous to the `Proxy` handler
object, such that the virtual host could choose to extend the handler with any
property of any type to carry the referrer, such that an import hook might
resolve a relative specifier using `this.referrer` (a `string` with the
module's own specifier), `this.slot` (a `number` representing the index into an
array of pre-computed resolutions for a bundle), or whatever other protocol the
virtual host chose to implement.

This was only possible because host-defined resolution behavior and
virtual-host-defined resolution behavior do not need to communicate.
Host-defined modules resolve import specifiers based on information in the
referring module record's host data internal slot.
At no point does a host-defined module need to consult the referrer of a
virtual-host-defined module.

We additionally elected not to repeat what we believe to be a mistake in the
design of `Proxy`.
The `Module` constructor eagerly captures all the optional methods of the
module handler rather than accessing them afresh for each invocation.
So, we sacrifice some dynamism but still thread the handler as the target
object for all method invocations.

A consequence of this design is that any `Module` instance will retain a
`handler`, where an options bag would be immediately released to the GC
nursery.
And, if the handler carries a `referrer`, every `Module` instance needs a
unique handler instance, even though many module instances can share
the same hooks.

## Design Variables 

### Relationship to Content-Security-Policy

On hosts that implement a Content-Security-Policy, the [[Origin]] is
host-specific [[HostData]] of a module source.
A module source constructed from a trusted types object could also
inherit an origin.
All of this can occur outside the purview of ECMA-262 and
is orthogonal to the `Module` referrer, which can be different than the origin.

### The name of module instances

`Module` instance and `ModuleInstance` instance are both contenders
for the name of module instances.
We are tentatively entertaining `Module` because it feels likely that we arrive
in a world where `(module {}) instanceof Module`.
The naming calculus would shift significantly if module blocks are reified as
module source instead of module instance.

We will no doubt arrive in a state of confusion with anything named "module"
without qualification.
For example, `WebAssembly.Module` more closely resembles a module source than a
module instance.
That would suggest module source should also be `Module`, and consequently
`ModuleInstance`.

### Form of the kicker method

A functionally equivalent proposal would add an `import` method to the `Module`
prototype to get a promise for the module's exports namespace instead of
overloading dynamic `import`.
Using dynamic import is consistent with an interpretation of the module blocks
proposal where module blocks evaluate to `Module` instances.

### Options bag or flat arguments

The options bag described here for the `Module` constructor leaves some
room open for accounting for import assertions.
It also raises questions about the default import hook and import meta,
which are answerable but have not yet been fully discussed.

[module-blocks]: https://github.com/tc39/proposal-js-module-blocks
[trusted-types]: https://w3c.github.io/webappsec-trusted-types/dist/spec/
[0-spec]: https://tc39.es/proposal-compartments/0-module-and-module-source.html
