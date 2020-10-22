# Compartments

Compartmentalization of host behavior hooks for JS.

**Stage**: 1

**Champions**:

* Bradley Farias, GoDaddy
* Mark S. Miller, Agoric
* Caridy Patiño, Salesforce
* Jean-Francois Paradis, Salesforce
* Patrick Soquet, Moddable
* Kris Kowal, Agoric

## Synopsis

Provide a mechanism to generate ECMAScript code that provides compartmentalized
host behavior from other ECMAScript code.

This is extracting some desired behavior from the existing [SES Proposal][ses]
for generalized use.

## Motivation

ECMAScript has many environments in which it runs, and many behaviors are host
driven. For a variety of use cases, the ability to alter the standard host
behavior is desirable.

Some of these environments are build time instead of run time, such as
bundling tool chains that override how things like import behavior. These are
explicitly not creating an isolation of the object address space for JS and
share the global with things using them.

Various efforts like [JSDom][jsdom], [SES][ses], and even testing frameworks
seek to override host behaviors within Realms. In TC39 we sometimes refer to
this as the ability to virtualize behavior.

Currently there is no standard way of virtualizing all behavior, nor of
compartmentalizing behaviors to specific source text. By introducing a means to
virtualize behaviors for source texts a variety of workflows are able to be
created without relying on host specific APIs. Existing method often replace
host driven APIs such as `Date` in various ways such as detached
`<iframes>`, using [CSP][csp], [Node.js's `vm.createContext`][vm-context],
XS's [Compartments implementation][xs-compartments], etc.

Historically, the SES proposal sought to achieve this behavior through isolation
as a security boundary, but the utility of changing these behaviors is outside
of purely security concerns. A large number of potential use cases lie purely in
the ability to virtualize some host behavior.

Currently, things like changing the effective time zone, locale, or limiting the
ability to `eval` code withing a source text is not possible in JS.TC39 in the
past has seen desires to override specific host behaviors such as [Zones][zones]
in specific situations. This proposal seeks to provide a way to coordinate host
behavior of such APIs in a generalized manner.

It seeks to do so in the following solution space:

* without requiring separating the address space
* without requiring an asynchronous messaging system
* keeping shared intrinsics of source texts
* does not allow escaping existing host security mechanisms

It leaves concerns about those to other proposals such as [Realms][realms] or
even host APIs.

### Rationale

There are several ways in which these behaviors could be overriden.

* They could only be allowed to be changed in a manner that is only virtualized
  to a newly allocated global scope.
* They could be done via API virtualization at the global scope level.
* They could be changed on a source text level to allow shared references
  without identity discontinuity.

The status quo is API virtualization at the global scope level, but not all
behavior is virtualizable doing so; notably, direct eval, `import.meta`, and
`import`.

There are ways to virtualize most other APIS

### Sketch

This is a rough sketch of potential APIs.

```ts
// Shared space by Realm
type FullSpecifier = string;
type ModuleNamespace = object;
// Used to create a Reusable Instance factory for Module Records
// exotic
interface StaticModuleRecord {
  // intend to add reflection of import/export bindings

  // needs to allow duplicates
  staticImports(): {specifier: string, exportNames}[];
}
interface SourceTextStaticModuleRecord extends StaticModuleRecord {
  // no coerce
  constructor(source: string);
}
type CompartmentConstructorOptions = {
  // JF has a better way for:
  //   randomHook(): number; // Use for Math.random()
  //   nowHook(): number; // Use for both Date.now() and new Date(),
  // Supplied during Module instance creation instead of option
  //   hasSourceTextAvailableHook(scriptOrModule): bool; // Used for censorship
  resolveHook(name: string, referrer: FullSpecifier): FullSpecifier

  // timing
  // importHook delegates to importNowHook FIRST,
  // they share a memo cache
  // order to ensures importNow never allows async value of import
  // to be accessed prior to any attached promise
  importHook(fullSpec: FullSpecifier): Promise<StaticModuleRecord>;
  importNowHook(fullSpec: FullSpecifier): StaticModuleRecord?;

  // copy own props after return
  importMetaHook(fullSpec: FullSpecifier, moduleRecord: SourceTextStaticModuleRecord): object

  // e.g.: 'fr-FR' - Affects appropriate ECMA-402 APIs within Compartment
  localeHook(): string;
  // This is important to be able to override for deterministic testing and such
  localTZAHook(): string;

  // determines if the fn is acting as an "eval" function
  isDirectEvalHook(evalFunctionRef: any): boolean;

  // prep for trusted types non-string
  canCompileHook(source: any, {
    evaluator: functionRef, // can be a value from isDirectEvalHook
    isDirect?: boolean
  }): boolean; // need to allow mimicing CSP including nonces
};
// Exposed on global object
// new Constructor per Compartment
//
// CreateRealm needs to be refactored to take params
//  - intrinsics: an intrinsics record from
//                6.1.7.4 Well-Known Intrinsic Objects
interface Compartment {
  constructor(
    // extra bindings added to the global
    endowments?: {
      [globalPropertyName: string]: any
    },
    // need to figure out module attributes as it progresses
    // maps child specifier to parent specifier
    moduleMap?: {[key: FullSpecifier]: FullSpecifier | ModuleNamespace},
    // including hooks like isDirectEvalHook
    options?: CompartmentConstructorOptions
  ): Compartment // an exotic compartment object

  // access this compartment's global object, getter
  globalThis: object;

  // do an eval in this compartment
  // default is strict indirect eval
  evaluate(
    // trusted types prep means use of `any`
    src: any,
    // FUTURE:
    //   for other eval goals like Module, need to discuss import()/eval() to
    //   get other Goals vs an option
    // options?: object
  ): any;

  // Return signature differs to allow avoiding then() exports
  // Used to ensure ability to be compatible with static import
  async import(specifier: string): Promise<{namespace: ModuleNamespace}>;
  // Desired by TC53
  importNow(specifier: string): ModuleNamespace;
  // Necessary to thread a module exports namespace from this compartment into
  // the `moduleMap` Compartment constructor argument, without importing (and
  // consequently executing) the module.
  module(specifier: string): ModuleNamespace;
}
```

## Design rationales

### Boxed module namespace returned by compartment import

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

In this proposal, the Compartment.import function differs from the
behavior of dynamic import by returning the namespace in a box.

```js
compartment.import('./thenable.js').then(({namespace: x}) => {
  // x will be a module namespace object with a then function.
})
```

[csp]: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
[jsdom]: https://www.npmjs.com/package/jsdom
[realms]: https://github.com/tc39/proposal-realms
[ses]: https://github.com/tc39/proposal-ses
[vm-context]: https://nodejs.org/api/vm.html#vm_vm_createcontext_contextobject_options
[xs-compartments]: https://blog.moddable.com/blog/secureprivate/
[zones]: https://github.com/domenic/zones/tree/eb65c6d43b452a877c24561cd64c6901e790ecf0
