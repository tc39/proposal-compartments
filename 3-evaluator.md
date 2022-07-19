
# Evaluators

## Synopsis

Provide an `Evaluators` constructor that produces a new `eval` function,
`Function` constructor, and `Module` constructor such that execution contexts
generated from these evaluators refer back to this set of evaluators, with a
given global object and virtualized host behavior for dynamic import in script
contexts.

## Interfaces

```ts
interface Evaluators {
  constructor({
    globalThis?: Object,
    importHook?: ImportHook,
    importMeta?: Object,
  });

  eval: typeof eval,
  Function: typeof Function,
  Module: typeof Module,
};
```

## Motivation

### Domain Specific Languages

Tools like Mocha, Jest, and Jasmine install the verbs and nouns of their
domain-specific-language in global scope.

Isolating these changes currently requires creation of a new realm,
and creating new realms comes with the hazard of identity discontinuity.
For example, `array instanceof Array` is not as reliable as `Array.isArray`,
and the hazard is not limited to intrinsics that have anticipated this
problem with work-arounds like `Array.isArray` or thenable `Promise` adoption.

Evaluators provide an alternate solution: evaluate modules or scripts in a
separate global scope with shared intrinsics.

```js
const dsl = const new Evaluator({
  globalThis: {
    __proto__: globalThis,
    describe,
    before,
    after,
  }
});

const source = await import(entrypoint, { reflect: 'module-source' });
const module = new dsl.Module(source);
await import(module);
```

In this example, only the entrypoint module for the DSL sees additional
globals.
The `Module` constructor adopts the host's import hook.
Notably, in this model, each entrypoint module could be granted separate
closures for the DSL.
Current DSLs cannot execute concurrently or depend on dynamic scope to track
the entrypoint that called each DSL verb.

### Enforcing the principle of least authority

On the web, the same origin policy has become sufficiently effective at
preventing cross-site scripting attacks that attackers have been forced to
attack from within the same origin.
Conveniently for attackers, the richness of the JavaScript library ecosystem
has produced ample vectors to enter the same origin.
The vast bulk of a modern web application is its supply chain, including code
that will be eventually incorporated into the scripts that will run in the same
origin, but also the tools that generate those scripts, and the tools that
prepare the developer environment.

The same-origin-policy protects the rapidly deteriorating fiction that
web browsers mediate an interaction between just two parties: the service and
the user.
For modern applications, particularly platforms that mediate interactions among
many parties or simply have a deep supply chain, web application developers
need a mechanism to isolate third-party dependencies and minimize their access
to powerful objects like high resolution timers or network, compute, or storage
capability bearing interfaces.

Some hosts, including a community of embedded systems represented at [ECMA
TC53][tc53], do not have an origin on which to build a same-origin-policy, and
have elected to build their security model on isolated evaluators, through the
high-level Compartment interface.

## Design

Where ***Execution Contexts*** and instances of `eval`, `Function`, and
`Module` were previously bound to a realm, they become bound to their
evaluators instead, which in turn is bound to the realm.

All references to %eval% must now refer to the
[[Realm]].[[Evaluators]].[[Eval]].
All other references to [[Context]].[[Realm]] must be replaced with
[[Context]].[[Evaluators]].[[Realm]], particularly to address the
intrinsics of the realm.

The rules for direct eval do not change as a consequence.
The name `eval` must be bound to the [[Context]].[[Evalutors]].[[Eval]]
for the `eval` special form to be interpreted as direct eval.
The creator of new evaluators must arrange for `evaluators.eval`
to be threaded into lexical scope for this to continue working.
For example:

```js
const localThis = { __proto__: globalThis };
const evaluators = new Evaluators({ globalThis: localThis });
localThis.eval = evaluators.eval;
evaluators.eval(`
  eval('var local = 42');
  typeof local === 'number'; // true
`);
```

The new global `Evaluators` constructor accepts a `globalThis`.
***Global Environment Record*** in ***Execution Contexts*** will
reach out to this object for properties in global scope.
If absent, the new evaluators will receive the `Evaluator` constructors
own [[Context]].[[Evaluators]].

The `Evaluators` constructor accepts an `importHook` having the same
signature as that specified in [Module and ModuleSource][0] of the
[Compartments proposal](README.md).
The `importHook` serves multiple roles.

The new `Module` constructor will adopt its evaluator's [[ImportHook]] and
[[ImportMeta]] if called without the corresponding `importHook` or `importMeta`
options.  Module execution contexts will in turn use the [[ImportHook]] and
[[ImportMeta]] from their `Module` constructor.

Dynamic import in script execution contexts will use their
[[Context]].[[Evaluators]].[[ImportHook]].
The import hook will receive the given specifier and the
[[Context]].[[Evaluators]].[[ImportMeta]].

## Design Questions

### Threading globals

Host implementors may not be able to accommodate an arbitrary value for
`globalThis`.
The proposal as written asks for the best user experience, but may need to
adjust if host implementations cannot support an arbitrary object, or if
limitations on the given object are not sufficient.

Evaluators will be useful in two different modes:

- Sharing a `globalThis`
- Having a separate `globalThis`

For separate globals, it would be okay for the `Evaluator` constructor to
receive a bag of properties to copy onto a global object constructed by the
host on their behalf.

For shared globals, copying properties isn't useful, so the argument
pattern would have to be different.

[0]: ./0-module-and-module-source.md
[tc53]: https://www.ecma-international.org/technical-committees/tc53/
