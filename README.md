# Module global

Stage: 1

Champions:

- Zbyszek Tenerowicz (ZTZ), Consensys, @naugtur
- Kris Kowal (KKL), Agoric, @kriskowal
- Richard Gibson (RGN), Agoric, @gibson042
- Mark S. Miller (MM), Agoric, @erights

## Problem statement 

A way to evaluate a module and its dependencies in the context of a new global scope within the same Realm


Sever the tie to the `GlobalEnvironmentRecord` in `ModuleEnvironmentRecord` instances and replace with `ScopeCeiling` whenever provided.



> This proposal picks up from the previous proposal for
> [Evaluators](https://github.com/tc39/proposal-compartments/blob/7e60fdbce66ef2d97370007afeb807192c653333/3-evaluator.md)
> from the [HardenedJS](https://hardenedjs.org) [`Compartment` proposal][proposal-compartments] and depends upon [proposal-import-hook][],
> [proposal-esm-phase-imports][], and [proposal-source-phase-imports][].

## Motivation

### Domain Specific Languages

Tools like Mocha, Jest, and Jasmine install the verbs and nouns of their
domain-specific-language in global scope.

Isolating these changes currently requires creation of a new realm,
and creating new realms comes with the hazard of identity discontinuity.
For example, `array instanceof Array` is not as reliable as `Array.isArray`,
and the hazard is not limited to intrinsics that have anticipated this
problem with work-arounds like `Array.isArray` or thenable `Promise` adoption.

Some of these tools work around this problem by using the platforms existing
facility for creating a new global context, albeit an iframe or the Node.js `vm`
module.
Then, they are obliged to graft the intrinsics of one realm over the other,
which leaks for the cases of syntactically undeniable Realm-specific intrinsics
like the `AsyncFunction` constructor and prototype, and requires the
implementer to be vigilant to the extent that they graft every intrinsic from
one realm to another.
We have found such arrangements to be fragile and leaky. Also costly in memory efficiency and developer time.

This proposal provides an alternate solution: evaluate modules or scripts in a
separate global scope with shared intrinsics.

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

### Isolating/encapsulating unreliable code

The way modern software is composed has already undermined the validity of the assumption that every author participating has their intentions well aligned for the benefit of the software working correctly. A whole new level of unreliability is now added with the popularity of _coding agents_ and _vibe coding_ where creating syntactically valid but effectively unpredictable JavaScript and integrating it into existing software to check whether it seems to implement the desired functionality is becoming a popular way of building software.

It is not an entirely new concern, as test runners have been concerned with isolating test cases to avoid them relying on global side-effects of other test cases. It is now a concern for a much wider audience with more at stake.

With `Global` constructor comes the ability to isolate fragments of the application in a way that unreliable code cannot rely on shared global state without the maintainer of the software knowing about it.
AI generated sources from independently working agents can come with colliding names for global variables to use and may need separate global scopes to collaborate or coexist. Similarly a misguided attempt at an inline polyfill by an AI or a package author could be prevented by freezing the parts of the new global in which the unreliable code subsequently runs.
Using a new global instead of a new Realm avoids the issues like identity discontinuity impeding the composition of software where function calls need to happen across the isolated and non-isolated code.


### Incremental or in-context execution

There are tools that currently use much more complex and costly mechanisms (similar to the ones described in [Domain Specific Languages](#domain-specific-languages) among other) to provide the ability to execute fragments of JavaScript code in a very specific context of the tool.

That includes REPLs, inline code execution results in editors (eg. [Quokka.js](https://quokkajs.com/)) and various use cases of IDEs in the browser.

Maintaining the global state between executions of user-provided code snippets would benefit from the ability to control scope 

## Proposal

Allow evaluating a module without access to global context by severing the tie to the `GlobalEnvironmentRecord` in `ModuleEnvironmentRecord` instances 
by setting `[[OuterEnv]]` to a different record than `module.[[Realm]].[[GlobalEnv]]`, replacing it with user-defined emulation of a global we will refer to as **Scope Ceiling**

### Scope Ceiling

[16.2.1.7.3.1 InitializeEnvironment ( )](https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#sec-source-text-module-record-initialize-environment)

```scheme
3. Let realm be module.[[Realm]].
4. Assert: realm is not undefined.
+ if module.[[ScopeCeiling]] is not EMPTY, then
+     Let outer be NewObjectEnvironment(module.[[ScopeCeiling]], false, null)
+     Let env be NewModuleEnvironment(outer)
+   Else
      Let env be NewModuleEnvironment(realm.[[GlobalEnv]]).
5. Set module.[[Environment]] to env.
```
> note:
> - `module` is Source Text Module Record
> - NewObjectEnvironment could be called earlier
> - `[[ScopeCeiling]]` will likely be accessed indirectly


The association between `ModuleRecord` and `ScopeCeiling` ultimately needs to be introduced by an intermediary - potentially the same intermediary that introduces a Module Map or an `importHook`. (e.g. Compartment) 

> We initially considered associating `ScopeCeiling` with `ModuleSource` in its constructor. It doesn't compose well with the esm-phase-imports proposal anymore.

---

#### Execution Context interactions


> Note: `module.[[Environment]]` becomes `LexicalEnvironment` of the `moduleContext`

```scheme
11. Set the Realm of moduleContext to module.[[Realm]].
12. Set the ScriptOrModule of moduleContext to module.
13. Set the VariableEnvironment of moduleContext to module.[[Environment]].
14. Set the LexicalEnvironment of moduleContext to module.[[Environment]].
15. Set the PrivateEnvironment of moduleContext to null.
16. Set module.[[Context]] to moduleContext.
17. Push moduleContext onto the execution context stack; 
  moduleContext is now the running execution context.
```


1. The `x` IdentifierReference is evaluated. Per [§13.1.3](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-identifiers-runtime-semantics-evaluation) (Runtime Semantics: Evaluation of IdentifierReference), calls [`ResolveBinding("x")`](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-resolvebinding).

2. [`ResolveBinding`](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-resolvebinding) reads `env` from the **running execution context's `LexicalEnvironment`**. The running execution context is `module.[[Context]]`, and its `LexicalEnvironment` is the module's `ModuleEnvironmentRecord`. Since module code is always strict, `strict` = **true**. Calls `GetIdentifierReference(moduleEnv, "x", true)`.


3. [`GetIdentifierReference`](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-getidentifierreference) calls **`moduleEnv.HasBinding("x")`** on the `ModuleEnvironmentRecord`. The module environment holds only the module's own top-level `var`/`let`/`const`/`class` declarations and imported bindings. `x` is none of these, so `HasBinding` returns **false**.

4. `GetIdentifierReference` reads `outer` = **`moduleEnv.[[OuterEnv]]`** reaching `ScopeCeiling`

5. `ScopeCeiling.[[OuterEnv]]` is null, so it cannot progress to Realm global for lookup


The change would result in an option to run a module in a context that does not have the means to reach globals lexically. 
The *undeniables* would come from the shared Realm, convenience of accessing intrinsics lexically via their global name
would depend on the provider of `ScopeCeiling` adding them.

---

## Intersection Semantics

Depends on an umbrella proposal to have an entrypoint to defining a `ScopeCeiling`, so likely will be folded into a Compartment proposal or its subset.


[proposal-source-phase-imports]: https://github.com/tc39/proposal-source-phase-imports
[proposal-esm-phase-imports]: https://github.com/tc39/proposal-esm-phase-imports
[proposal-compartments]: https://github.com/tc39/proposal-compartments
[proposal-import-hook]: https://github.com/endojs/proposal-import-hook
