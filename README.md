# Compartments

**Stage**: 1

**Champions**:

* Mark S. Miller, Agoric
* Caridy Patiño, Salesforce
* Patrick Soquet, Moddable
* Kris Kowal, Agoric
* Jack Works, Sujitech
* Guy Bedford, OpenJS Foundation

**Emeritus**:

* Bradley Farias, GoDaddy
* Jean-Francois Paradis, Salesforce

# Synopsis

Compartments are a mechanism for isolating and providing limited power to
programs within a shared realm.
Each compartment shares the intrinsics of a realm, but a different set of
evaluators (`eval`, `Function`, and a new evaluator, `Module`) and a global
object.
Having a separate global object allows each compartment to be granted access to
only those powerful objects it needs, its own isolated evaluators, powerless
constructors, and shared prototypes.

The Compartments proposal was approved for Stage 1 (exploration of a problem)
with the charter, "to compartmentalize host behaviors".
The problem we set out to solve was excess authority flowing from global scope
and host behaviors into third-party dependencies and plugins in large
applications.
Through exploring this problem, we discovered that the bulk of the solution, by
weight, was virtualizing the EcmaScript module loader.
Provided an EcmaScript module loader, we could then build a solution for
isolating code for both scripts and modules.

Over the course of two years, we refined the Compartment class to account for
the need to make and import bundles, emulate various host module specifier
namespaces, link modules between multiple compartments, and support
non-EcmaScript module languages.

We then began working with champions of module blocks, module fragments,
deferred import, and import reflection to ensure these proposals were coherent.
From these discussions, we discovered a set of lower-level interfaces from
which compartments could be constructed in user code that were more coherent
with these other proposals.

With that, the Compartments proposal consists of five layers:

- [Module and ModuleSource][0]: Provide first-class `Module` and
  `ModuleSource` constructors and extend dynamic import to operate on `Module`
  instances. ([Specification Changes][0-spec])

- [Surface Module Source Static Analysis][1]: Extend instances of
  `ModuleSource` such that they reflect certain results of static analysis,
  like their `import` and `export` bindings, such that tools can inspect module
  graphs.

- [Virtual Module Sources][2]: Extend the `Module` constructor such that it
  accepts virtual module sources: objects that implement a protocol that is
  sufficient for virtualizing the evaluation of modules in languages not
  anticipated by ECMA-262 or host implementations.

- [Evaluators][3]: Provide an `Evaluators` constructor that produces
  a new `eval` function, `Function` constructor, and `Module` constructor
  such that execution contexts generated from these evaluators refer
  back to this set of evaluators, with a given global object and virtualized
  host behavior for dynamic import in script contexts.

- [Compartment][4]: Compartments are a high-level mechanism for isolating
  and providing limited power to programs within a shared realm.
  Compartments can be implemented in user code using `Evaluators`, `Module`,
  and `ModuleSource`.

[Motivating use cases for module proposal features](./GRAPH.md) illustrates
the motivation for each feature from the layers of this Compartments proposal
and related module proposals.

[0]: ./0-module-and-module-source.md
[0-spec]: https://tc39.es/proposal-compartments/0-module-and-module-source.html
[1]: ./1-static-analysis.md
[2]: ./2-virtual-module-source.md
[3]: ./3-evaluator.md
[4]: ./4-compartment.md
